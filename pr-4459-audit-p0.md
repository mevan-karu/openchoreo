<!--
PR title must follow Conventional Commits format: type(scope): subject
Scope is optional and subject must start with a lowercase letter.

Suggested title:
  feat(api): build a policy-driven audit logging framework for REST and MCP

See: docs/contributors/github_workflow.md#pr-title-convention
-->

## Purpose
> Briefly describe the problem or need driving this PR and how it resolves the issue. Include links to related issues if applicable.

The audit logging framework merged in PR #1465 was silently unwired by #2588 during an API-layer refactor and had been dead code ever since — `GetActionDefinitions()` had zero callers, the middleware was registered on no route, and nothing consumed `audit.SetResource`/`audit.AddMetadata`. This PR restores emission for a verified slice of routes, makes the wiring structurally impossible to silently drop again, and builds the restoration out into a full framework: a surface-neutral core shared by REST and MCP, an ordered policy engine that controls what gets published, and matching adapters for both surfaces — so the same policy applies identically no matter which one produced the event.

Closes #4459.

## Approach
> Summarize the solution and implementation details.

### Restoration (wiring + identity + config toggle)
- **Single chain constructor.** `APIMiddlewares`/`APIMiddlewareOptions` (`internal/openchoreo-api/api/handlers/handler.go`) is the one place the OpenAPI middleware chain is assembled. Production and the test helper (`newTestHTTPHandlerWithLogger`) both build through it, so they can't drift the way #2588 did.
- **Actor identity fixed.** `extractActor`/`ExtractActor` no longer fabricates an identity: it uses the token's `sub` claim when present, otherwise records `"unknown"` — never the literal string `"<nil>"` that `fmt.Sprintf("%v", nil)` used to produce.
- **Resource population.** `audit.SetResource` is called after the error check on every mutating Project/Environment/DataPlane/Secret handler, including `Resource.ID` (the K8s UID) wherever the handler has access to the persisted object — names are mutable and don't uniquely identify a resource across renames.
- **Config toggle.** `AuditConfig.Enabled`, wired through `config.yaml` and Helm (`values.yaml`/`values.schema.json`/`configmap.yaml`). `enabled` is threaded as an explicit parameter into both constructors rather than derived from the `Emitter` — it disables all audit-related work on that surface, not just publishing.

### Core framework (`internal/server/middleware/audit`)
Split into surface-neutral files with no HTTP or MCP SDK imports, so REST and MCP build the same `Envelope` shape from the same types:
- `operation.go` / `operation_def.go` — `Operation` is the surface-neutral shape; `OperationDef` is a service's own source-of-truth table (ID, Action, ResourceType, Category, plus optional MCP tool binding fields). `Operations()`/`MCPBindings()` derive each surface's view from one table, so a service never maintains two definitions of the same operation.
- `envelope.go` / `emitter.go` — `Envelope` is everything an adapter must supply; `Emitter.Emit` resolves publish/suppress via the `PolicySet` and fans out to every configured `Sink`, stamping `EventID`/`Timestamp`/`Service` once centrally so sinks can't disagree on identity.
- `adapter.go` — `ExtractActor`, `RequestIDFromHeader`, `SourceIPFromHeader`, `EmitFromContext`: the shared logic both REST's `Middleware` and MCP's audit middleware call to turn their own request shape into an `Envelope`.
- `mcp_binding.go` — `MCPBinding` carries the fully resolved `Operation` (not just an ID), so a binding can't drift apart from its operation.
- `pattern_map.go` — `BuildPatternMap` cross-references each `Operation.ID` (OpenAPI operationId) against the live spec to build an exact `"METHOD path"` → `Operation` map, matching what `r.Pattern` reports once routed. Replaces the old regex-based resolver (`resolver.go`, deleted), which had two live failure modes: a bad capture-group name silently skipped, and pattern shadowing on first match. `BuildPatternMap` errors loudly instead — unmatched operationId or pattern collision both fail construction.

### Policy engine (`policy.go`)
`PolicySet` is an ordered, first-match-wins rule list, immutable after construction (required since concurrent MCP tool calls resolve settings from the same set in parallel goroutines). Within one rule, populated selector fields are AND'd; values within one field are OR'd. `NewPolicySet` is the single admissibility gate — both `AuditConfig.Validate` and the real build path call it, so there's no second, independently-maintained validation path. It rejects:
- an actor/entitlement selector used to *suppress* publishing (escalation-only — a wrong match should log more, never less);
- a rule that overrides nothing (almost certainly a copy-paste mistake that would silently swallow later rules for anything it matches);
- an empty-match rule with `publish: false` (would silence every event — the exact failure class this framework exists to prevent).

### REST and MCP adapters
- **REST** (`middleware.go`): `NewMiddleware` builds the pattern map from `BuildPatternMap` and constructs the `Middleware` in one call. `Handler` wraps the emit in `defer`+`recover`+re-`panic`, so a panicking handler still produces a failure record before the panic propagates.
- **MCP** (new package `pkg/mcp/mcpaudit`): mirrors the REST middleware for `tools/call`. Before calling the handler, it seeds a placeholder `Resource{Type, Name}` from the bound operation — necessary because a PDP-denied call never reaches the handler that would otherwise set it, so this is the only source of resource info for a denied MCP call. `NewMiddleware(MiddlewareOptions{Emitter, Bindings, Enabled})` only builds the middleware; `pkg/mcp`'s `NewHTTPServer` owns composing it outermost in the receiving chain. This ordering matters: go-sdk's `AddReceivingMiddleware` wraps middleware in the *opposite* order from `APIMiddlewares`' last-is-outermost convention, so audit is listed first specifically so it observes the authz filter's denials as well as tool outcomes.
- `pkg/mcp/tools/filter.go` now exports three sentinel errors (`ErrNoSubject`, `ErrForbidden`, `ErrPDPFailure`) so a PDP outage is recorded as a failure, never misclassified as a policy denial.

### Category taxonomy
`Category` is two values: `management` (create/update/delete on Project/DataPlane/Environment/Secret today) and `authorization` (reserved for the AuthzRole/AuthzRoleBinding operations landing later). Earlier draft categories (`auth`, `observability`) were dropped — neither mapped to a real operation, and `auth` in particular has no populated path today: OpenChoreo validates externally-issued JWTs rather than issuing its own sessions, and on both REST and MCP the JWT middleware runs *before* the audit middleware in the chain, so an authentication failure never reaches it. Wiring that up is left as explicit future work rather than carrying a placeholder category for it now.

### Verification
`TestAuditMiddlewareWired`/`TestAuditMiddlewareWired_ProjectCRUD` (REST) and the new MCP wiring tests in `pkg/mcp/server_audit_test.go`/`pkg/mcp/mcpaudit` drive real requests through the production constructors and assert a complete `AUDIT-LOG` record is emitted. Both wiring guards were verified real by temporarily removing the audit entry from `APIMiddlewares` and from the MCP receiving-middleware chain respectively — each corresponding test fails, and passes again once restored. `go build`, `go vet`, `make lint`, and the full audit/config/mcp test suite are green.

## Related Issues
> Include any related issues that are resolved by this PR.

Closes #4459.

## Checklist
- [x] Tests added or updated (unit, integration, etc.)
- [ ] Samples updated (if applicable)
- [ ] Added `backport/<release-branch>` label if this should be backported (e.g., `backport/release-v1.0`)
<!-- Tick the box below if AI generated code or content in this PR. Simple autocomplete or asking AI to explain code doesn't count. See docs/contributors/AI-POLICY.md -->
- [x] This PR includes AI-generated code or content

## Remarks
> Add any additional context, known issues, or TODOs related to this PR.

- Coverage is deliberately partial: 12 of 114 state-modifying routes are audited, on both REST and MCP. A full rewrite against all mutating routes, plus a CI gate that fails the build when a new mutating route ships without a matching audit definition, is tracked as follow-up work.
- Audit sits inside auth on both surfaces (REST and MCP), so a rejected/unauthenticated request (a 401, or an MCP call with no valid session) still produces no event. Auditing auth failures needs a second, outer middleware instance on each surface and is out of scope here.
- `/exec/` and wirelogs remain unaudited — they're on separate mux/middleware chains this PR doesn't touch.
- `golangci-lint`/`make lint` is fully clean across the repo as of this PR (previously noted pre-existing staticcheck findings elsewhere in the repo have since been resolved on `main`).
