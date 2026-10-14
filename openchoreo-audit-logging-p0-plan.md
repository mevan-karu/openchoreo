# Audit Logs Module — P0 Implementation Plan

**Companion to:** [`openchoreo-audit-logging-scope.md`](./openchoreo-audit-logging-scope.md) — the requirements register · [`openchoreo-audit-logging-phased-plan.md`](./openchoreo-audit-logging-phased-plan.md) — the P0–P10 sequencing. Read that document's §2 and its P0 section first.

**Epic:** [#3343](https://github.com/openchoreo/openchoreo/issues/3343) · **Phase:** P0 of P0–P10 · **Prior implementation:** [PR #1465](https://github.com/openchoreo/openchoreo/pull/1465) *(merged, then unwired)*

**Requirements delivered:** EM-7, EM-4b · PR-1 *(partially — the framework emits on both surfaces; full coverage is P1)* · PR-4, PR-6 · OP-1, OP-3

**Status:** ✅ **Shipped** · **Date:** 2026-08-10 · **Revised:** 2026-08-13 (scope extended — see §5b), 2026-08-24 (reconciled against the shipped code — see §0), 2026-08-30 (re-validated against `92a7e1ba`; two amendments — see §0's rewrite note and 0.6's ordering addendum) · **Verified against:** PR [#4462](https://github.com/openchoreo/openchoreo/pull/4462) and its successors [#4460](https://github.com/openchoreo/openchoreo/pull/4460) and [#4571](https://github.com/openchoreo/openchoreo/pull/4571) on `feat/audit-logging-p1`

> **Re-validation, 2026-08-30.** Every P0 deliverable (0.1–0.13, 0.10a–e) was re-checked against
> the working tree and still holds; build, vet, the full audit test suite and `make lint` are
> clean. **Two things changed under P0's feet**, both from PR #4571 and both recorded in place:
> the commit table in §0 rotted for the third time, and deliverable 0.6's review rule gained a
> second clause. Nothing in P0's *design* was contradicted.

> ### §0 — Commit-citation policy (2026-08-24)
>
> **The SHAs this document cites no longer exist on the branch.** It was rebased or squashed,
> and `6bd147a1` (restoration), `96fe08ca` (framework) and `8c8b33b7` (resource.type +
> level-gating) are all unreachable:
>
> ```sh
> $ git merge-base --is-ancestor 6bd147a1 HEAD   # → false, for all three
> ```
>
> They survive as loose objects in some working copies, so `git cat-file -t` still answers
> `commit` — but they would not resolve in a fresh clone and are gc-able. The branch is
> `feat/audit-logging-p1`, not `feat/audit_logging`.
>
> **Policy: cite PRs, not SHAs.** This branch has already been rewritten once. Where a SHA
> appears below, read it as "the change described", and take the durable citation from
> PR #4462 and its successors. For reference, the current six commits are:
>
> | Commit | Old citation | Content |
> |---|---|---|
> | `431092a7` | `6bd147a1` + `96fe08ca` | The policy-driven framework, with a hand-written `definitions.go`. The restoration commit has no separate successor — it was folded in here |
> | `268cbf8d` | — | Codecov test-coverage follow-up |
> | `f676c146` | `8c8b33b7` | `resource.type` consistency and audit level-gating |
> | `0ad4c9bd` | — | `MCPBindingKey{ToolName, Scope}` (0.10a–e) and `X-Request-ID` hardening |
> | `1e463391` | — | PDP-error sanitisation, `resource.namespace`, explicit-logger requirement |
> | `8d257ca9` | — | P1: the generator, the coverage gate, and much more — see the P1 plan's §0 |
>
> Note also that **`main` is behind this branch's base** (`git merge-base main HEAD` =
> `61e15120`), so `git diff main...HEAD` includes ~12 unrelated upstream commits. Scope any
> audit diff as `431092a7^..8d257ca9`.
>
> > **Rewritten again — the table above is now stale too (2026-08-30).** Every SHA in it fails
> > `git merge-base --is-ancestor <sha> HEAD`. That is the **third** rewrite this branch has
> > taken, and the second time a "here are the current SHAs" table has rotted, so no replacement
> > table is offered. The branch today carries three audit commits:
> >
> > | Commit | PR | Content |
> > |---|---|---|
> > | `a7f1d9c1` | [#4462](https://github.com/openchoreo/openchoreo/pull/4462) | **P0** — this document: framework, core, both adapters |
> > | `6ee4f787` | [#4460](https://github.com/openchoreo/openchoreo/pull/4460) | **P1** — generator, coverage gate, full binding table, hierarchy |
> > | `92a7e1ba` | [#4571](https://github.com/openchoreo/openchoreo/pull/4571) | Auth rejections on exec/wirelogs/MCP — see the P1 plan's §0.8 |
> >
> > **Use the PR numbers.** The SHAs are recorded to scope a diff today and will rot the same way.

> P0 began as a **restoration** phase: the audit framework was merged by PR #1465, unwired by `a64ea04d` (#2588), and left as unreachable dead code. Deliverables 0.1–0.6 restored it and made a second silent removal impossible.
>
> **P0 now also carries the core framework** — the surface-agnostic emitter, the configuration model, and both the REST and MCP adapters (§5b). The reasoning is that everything after P0 should be an *improvement on a settled foundation* rather than a reshaping of it, and all the architectural risk lives in the core.
>
> Everything in §2 was verified with a reproducing command alongside each claim. **Four statements in the phased plan's P0 section are corrected here** — see Appendix A. The most important is that task 0.4's regression guard, as originally specified, would not have guarded anything.

> **Post-implementation corrections (external review of PR #4462, 2026-08-19).** The review found three defects and confirmed one plan/implementation mismatch as a documentation gap, not a code gap. **Items 1–3 are fixed on `feat/audit_logging`; item 5 is open** and is the one remaining code change in P0:
>
> 1. **Exit criterion 5 (§8) was not actually met.** `Logger.LogEvent` emitted at `slog.LevelInfo` through the *application* logger, so `logging.level: warn` silently dropped every audit event even with `audit.enabled: true` — a second, undocumented kill switch. Fixed: `NewLogger` now wraps the given handler's `Enabled` to always return true, so the app's log level can no longer gate audit output. `audit.enabled` is now genuinely the only kill switch, closing the gap this note used to flag.
> 2. **Exit criterion 12 was not met on REST** (§8) and **deliverable 0.13's REST half was unimplemented** (below): a PDP denial inside a handler never reached the `SetResource` call that would set `resource.type`/`resource.name`, so a 403 shipped with `resource: null`. Separately, `resource.type` diverged across all four success/denial × REST/MCP combinations because REST's `operationDefs.ResourceType` used the plural form (`"projects"`) while every handler's `SetResource` call used the singular (`"project"`) — so a policy `match: {resources: [projects]}` never matched a real event. Fixed: `Operation.ResourceType` is singular everywhere; `Resource` (the type `SetResource` takes) carries no `Type` field at all — `Event.ResourceType` is stamped from `Operation.ResourceType` at emit time (`buildEvent`), so a handler-supplied value can no longer diverge from it, and REST now seeds a resource (path-derived name, via a new `Operation.RESTResourceParam`) before calling the handler, mirroring what the MCP adapter already did.
> 3. **Deliverable 0.3's defensive `"<nil>"` guard in `ExtractActor` was unimplemented.** Only the resolver-side fix (reading `sub` explicitly rather than through `fmt.Sprintf`) landed; the belt-and-braces check in `ExtractActor` itself did not. Fixed: `ExtractActor` now also rejects a literal `"<nil>"` `ID`, falling back to `"unknown"`.
> 4. **The category taxonomy decision is final, not pending.** The shipped taxonomy (`management`/`authorization`) matches neither this plan's P0 set (`resource`/`auth`/`observability`) nor P1's target (`resource`/`permission`/`access`) — see §3 and Appendix B. Decided 2026-08-19: **`management`/`authorization` ships as the permanent taxonomy.** P1's planned rename (Appendix B, and phased-plan.md's P1 section) is dropped — there is nothing left to rename to.
>
>    **Confirmed unqualified, 2026-08-19, while planning P1** — `management`/`authorization` is the complete and permanent `Category` set. An intermediate revision of this note proposed adding a third value, `access`, for requests rejected before any operation resolves (P1's PR-7 work). **That was reversed the same day.** `Category` groups *operation types*; a rejected request is not a kind of operation but an *outcome*, so the distinction belongs on `Result`. The decisive evidence was surface parity: on MCP, `tools.ErrNoSubject` fires on a **bound** tool, so `op != nil` and the event legitimately carries `category: management` — the same logical event would have had a real category on MCP and `access` on REST, breaking the surface-parity rule the P1 plan sets for its own selectors.
>
>    **Instead, `Result` gains `unauthenticated`** — and this fixes a **live** conflation, not just a P1 blocker: `determineResult` maps `401 || 403 → ResultDenied` (`middleware.go:157-166`) and `classifyResult` maps `ErrNoSubject, ErrForbidden → ResultDenied`, so on all 12 already-audited operations "unauthenticated" and "authorization refused" are recorded identically today. MCP's sentinels already name the two cases exactly, so the distinction is being discarded at the audit boundary rather than being absent. **The fix is P1 task 1.0b**, not P0 — P0 is closed, and PR-7 coverage is the change's main consumer. Full reasoning in `openchoreo-audit-logging-p1-plan.md` §4.3.1 and its Appendix B item 7; §6's closing note records why it is not filed here.
>
> 5. **The MCP binding key cannot express a scope-collapsed tool, and a collision is silent — found 2026-08-19 while planning P1, and pulled into P0 as `0.10a`.** REST has two operations per scope (`CreateComponentType` / `CreateClusterComponentType`); MCP collapses them into one tool routing on a `scope` argument, and 21 tools do this. `OperationDef` carries one `MCPToolName`, `MCPBindings` writes `bindings[d.MCPToolName]` with no collision check, and the adapter looks up on tool name alone — so both scopes resolve to whichever def came last, mis-attributing a namespaced create as a platform-wide cluster one. `definitions_test.go` asserts unique operation *IDs*, which differ, so it passes. **Inert today** (none of P0's 6 bindings collide) and springs the moment P1 adds the fan-outs. It sits in P0 because it is a core-shape defect, and P0's premise is that later phases improve a settled core rather than reshape it. **Fixed 2026-08-19** — see 0.10a–0.10e in §6 for the shipped change and 0.12 for the resolution-ordering constraint it satisfies.
>
> `§4`'s `AddReceivingMiddleware` evidence table and 0.12's description of MCP wiring are also corrected in place below — the shipped code uses one call listing audit first (`server.AddReceivingMiddleware(auditMw, filterMw)`), not a second, separately-registered call as originally specified. This was not a defect; the plan's description of the mechanism was wrong, the outcome (audit observes the filter's denials) was right.

---

## 1. Goal and shape of the phase

Three outcomes. The second is why the phase originally existed; the third is the 2026-08-13 extension.

1. **Audit events emit again** from `openchoreo-api`, for a small set of routes that provably match reality, with a complete event body.
2. **The wiring cannot be silently removed again.** The regression happened because `definitions.go` has no compile-time relationship to any route and the chain was assembled inline in `main()`. A refactor rewrote the API layer out from under the audit hook and nothing failed. P0 closes that.
3. **The core framework is settled, on both surfaces.** One surface-agnostic emitter, one configuration model, and two thin adapters — REST and MCP — so that P1 onward adds coverage and fields rather than reshaping the foundation.

P0 stays deliberately small on **coverage** — 6 routes of 114, and MCP bindings only for those — while spending its effort on the structural guarantees. The rewrite against all 114 state-modifying routes, the 69-entry MCP binding table, the generator, the CI coverage gate, the category taxonomy rename and auth-failure coverage are all P1.

---

## 2. Verified current state

Assessed against `main` at `f9f0a89c`. Each claim carries its reproducing command.

### 2.1 The framework is unreachable

```sh
$ grep -rln "server/middleware/audit" --include="*.go" .
internal/openchoreo-api/audit/definitions.go        # the only importer

$ grep -rn "audit\.SetResource\|audit\.AddMetadata\|audit\.NewMiddleware\|audit\.NewLogger" \
    --include="*.go" .
                                                     # no results
```

`GetActionDefinitions()` has zero callers. All seven handler-side helpers in `context.go` have zero callers. The middleware is registered on no route group in any service.

The wiring is recoverable from history — `git show a64ea04d^:internal/openchoreo-api/handlers/handlers.go` shows `initAuditMiddleware()` at `:288` and `api := routes.With(jwtAuth, auditMiddleware)` at `:85`.

### 2.2 There are 25 action definitions; 6 match a live route

```sh
# definitions
$ awk '/Method:[ \t]+"/ { m=$0; sub(/.*Method:[ \t]+"/,"",m); sub(/".*/,"",m) }
       /Pattern:[ \t]+"/ { p=$0; sub(/.*Pattern:[ \t]+"/,"",p); sub(/".*/,"",p); print m" "p }' \
    internal/openchoreo-api/audit/definitions.go | sort > /tmp/defs.txt
$ wc -l < /tmp/defs.txt
25

# live state-modifying routes
$ grep -oE 'm\.HandleFunc\("(POST|PUT|PATCH|DELETE) "\+options\.BaseURL\+"[^"]*"' \
    internal/openchoreo-api/api/gen/server.gen.go \
  | sed -E 's/m\.HandleFunc\("//; s/ "\+options\.BaseURL\+"/ /; s/"$//' | sort > /tmp/live.txt
$ wc -l < /tmp/live.txt
114
$ awk '{print $1}' /tmp/live.txt | sort | uniq -c
  38 DELETE
  42 POST
  34 PUT
```

**Zero `PATCH` routes exist**, so all three `PATCH` definitions are provably dead.

Comparing the two sets with parameter names normalised gives **6 matches and 19 dead definitions**. The 6 survivors match **including their parameter names**, so no editing is required — task 0.2 is a pure deletion.

**Keep these 6, exactly as written today:**

| Method | Pattern | Action |
|---|---|---|
| POST | `/api/v1/namespaces/{namespaceName}/projects` | `create_project` |
| POST | `/api/v1/namespaces/{namespaceName}/environments` | `create_environment` |
| POST | `/api/v1/namespaces/{namespaceName}/dataplanes` | `create_dataplane` |
| POST | `/api/v1alpha1/namespaces/{namespaceName}/secrets` | `create_secret` |
| PUT | `/api/v1alpha1/namespaces/{namespaceName}/secrets/{secretName}` | `update_secret` |
| DELETE | `/api/v1alpha1/namespaces/{namespaceName}/secrets/{secretName}` | `delete_secret` |

**Delete the other 19**, in four groups:

| Group | Why dead |
|---|---|
| 6 × `authz/roles` and `authz/role-mappings` | Routes are now `namespaces/{ns}/authzroles`, `/api/v1/clusterauthzroles`, and `*rolebindings` |
| `POST /api/v1/apply`, `DELETE /api/v1/delete` | Do not exist |
| `POST /api/v1/namespaces/{ns}/git-secrets` | Now `POST /api/v1alpha1/namespaces/{ns}/gitsecrets` |
| 10 × `.../projects/{p}/components/...` | Components are flat under namespace; `deploy`, `promote`, `traits`, `workloads`, `workflow-runs`, `component-releases`, `bindings`, `workflow-parameters` and both component CRUD entries all miss |

This also means **PR-2 is ✅ on paper and ❌ in fact** — authorization-change auditing is declared, and all six of its patterns are wrong.

### 2.3 `extractActor` discards the subject ID — confirmed

`internal/server/middleware/audit/middleware.go:126-130`:

```go
// Extract actor ID from entitlement values (first value is typically the user ID/email)
actorID := "unknown"
if len(subjectCtx.EntitlementValues) > 0 {
    actorID = subjectCtx.ID
}
```

`subjectCtx.ID` — the `sub` claim — is gated on an unrelated field, and the comment describes code that no longer exists. **Reachable in practice:** `detectUserTypeFromClaims` (`internal/server/middleware/auth/jwt/resolver.go`) returns `(true, []string{})` for a present-but-empty entitlement claim, so a successfully authenticated subject lands here and is recorded as `actor.id = "unknown"`. Defeats EM-7.

### 2.3a The subject ID claim is not configurable — and audit is its first consumer

The actor identity has exactly one construction site, and the claim name is hardcoded:

```go
// internal/server/middleware/auth/jwt/resolver.go:90-95
return &auth.SubjectContext{
    ID:                fmt.Sprintf("%v", claims["sub"]),
    Type:              userTypeConfig.Type,
    EntitlementClaim:  jwtMechanism.Entitlement.Claim,
    EntitlementValues: entitlements,
}, nil
```

What the subject model **does** expose (`security.subjects.<type>`, `internal/openchoreo-api/config/security.go:177-185`) is `display_name`, `priority` and `mechanisms.<type>.entitlement.claim`. So the **entitlement** claim is configurable per subject type per mechanism; the **identity** claim is not.

**This is correct and should stay that way.** EM-7 already fixes the choice — *"Actor identity is taken from the validated token's `sub` claim and does not depend on any other claim being present."* `sub` is the only claim OIDC guarantees present and stable; allowing an operator to point identity at `email` or `preferred_username` would silently rewrite audit history when a user's email changes, and break the IdP correlation EM-8 and DO-5 depend on. A human-readable label is a *display* concern — an additional field in P2/P3, never an override of the identity field.

**Audit is the first and only consumer of `sub`.** `authz.GetAuthzSubjectContext` (`internal/authz/core/converter.go:16-20`) copies only `Type`, `EntitlementClaim` and `EntitlementValues` — it **drops `ID`**. Every authorization path routes through that converter (`services/authz.go:37`, `:75`; `api/handlers/authz.go:110`, `:178`; `mcphandlers/authz.go:248`; `observer/authz/helpers.go:112`; `pkg/mcp/tools/filter.go:96`, `:153`). The only reader of `SubjectContext.ID` in the repository is the audit middleware (`middleware.go:112`).

That explains why both actor defects went unnoticed — nothing consumed the field — and it means **P0 makes `sub` load-bearing for the first time**. Deliverable 0.3 is correspondingly higher-stakes than a two-line fix suggests.

### 2.3b `sub` is never validated as present, so `"<nil>"` is reachable

`validateClaims` (`internal/server/middleware/auth/jwt/middleware.go:222-269`) validates **only `iss` and `aud`**. There is no required-claims check and `sub` is not among them. A token without `sub` therefore authenticates successfully, and `fmt.Sprintf("%v", nil)` renders `ID` as the **literal four-character string `"<nil>"`**.

This interacts badly with deliverable 0.3. Today such a token yields `actor.id = "unknown"` — wrong, but honestly wrong. After the entitlement gate is removed, `"<nil>"` is non-empty and would be recorded as a legitimate actor identity: **a fabricated identity that reads as real data**. For an audit trail that is worse than the bug being fixed.

The phased plan schedules this as P2 task 2.8. **It must move into P0** — P0 is what makes it reachable, and it is three lines in code P0 is already editing.

### 2.4 Middleware ordering mechanics — confirmed, and they favour us

`gen.HandlerWithOptions` stores the slice on the wrapper (`server.gen.go:9183-9187`) and each generated route applies it:

```go
for _, middleware := range siw.HandlerMiddlewares {
    handler = middleware(handler)
}
```

So **later slice entries wrap outermost**. The current production slice is `{WebhookRawBodyMiddleware, authMiddleware, loggerMiddleware}` (`cmd/openchoreo-api/main.go:227-230`), giving `logger → auth → webhookRawBody → handler`. Inserting audit between `WebhookRawBodyMiddleware` and `authMiddleware` yields `logger → auth → audit → webhookRawBody → handler` — matching the pre-`#2588` position.

Two details worth knowing:

- `options.BaseURL` is not set, so route patterns are literal `/api/v1/...` and match the definitions directly.
- Middlewares are applied **inside** the generated wrapper, *after* path-parameter binding. A parameter that fails `BindStyledParameterWithOptions` returns 400 without reaching audit. Acceptable for P0; note it in the coverage matrix in P1.

### 2.5 Handler-side plumbing works — confirmed

Strict handlers receive the request context: `response, err := handler(r.Context(), w, r, request)` (`server.gen.go:26443`). So `audit.SetResource(ctx, …)` called from a strict handler body reaches the `AuditData` container the middleware placed in the context. No refactor needed to populate `resource`.

### 2.6 A reusable HTTP test harness already exists

`internal/openchoreo-api/api/handlers/http_test_helpers_test.go` provides `newTestHTTPHandler` (`:29`), `injectTestSubject` (`:44`) and `doRequest` (`:56`). `newTestHTTPHandler` already builds a real `gen.HandlerWithOptions` chain with a fake K8s client and an injected subject — the exact shape P0's guard test needs. There are ~15 `*_http_test.go` files using it.

### 2.7 The resolver's silent-failure trap is not currently sprung

`NewActionResolver` (`resolver.go:46-50`) **silently `continue`s** when `regexp.Compile` fails, so a path variable that is not a valid Go capture-group name disables auditing for that route with no error. Checked:

```sh
$ grep -oE '\{[^}]*\}' /tmp/live.txt | sort -u | grep -vE '^\{[A-Za-z0-9_]+\}$'
(no output — all live route params are valid capture-group names)
```

So this is a **P1 gate concern, not a P0 blocker**. P1's coverage test (task 1.6) must assert every pattern compiles, because the first route named `{role-name}` will silently lose auditing.

---

## 3. Non-goals

Each of these is deliberate, and each is recorded so a reviewer does not read an omission as an oversight.

- **Auth-failure auditing (PR-7).** P0 restores audit **inside** auth. That is the same position the phased plan's §2.4 identifies as why rejected tokens were never audited — `authMiddleware` short-circuits 401s before any inner middleware runs. It is retained because a populated `SubjectContext` (EM-7) requires it, and because **one middleware instance cannot be both inside and outside auth**. P1's two-instance design resolves it. Restoring this placement is not an endorsement of it.
- **MCP (PR-4).** Left unaudited. See §4 — this needs its own section because MCP is enabled by default.
- **`/exec/` and wirelogs.** Registered on a separate top-level mux outside the OpenAPI chain (`main.go:232-269`) precisely because response-writer wrappers break `http.Hijacker`. `component:exec` is the most sensitive action in the API; P1 (task 1.9) must either audit it by another mechanism or record the exclusion.
- **The definitions rewrite and the CI coverage gate** — P1 tasks 1.5 and 1.6.
- ~~**The category taxonomy.** `resource` / `auth` / `observability` stays as-is in P0. The rename to `resource` / `permission` / `access` (EM-4a/4b/4c) is P1 task 1.2, because it is only meaningful once `access` has a producer.~~ **Superseded 2026-08-19** — see correction 4. P0 shipped `management`/`authorization`, which is now the permanent set; there is no P1 rename, and no `access` category at all. The rejection case became `result: unauthenticated`, in P1 (task 1.0b).
- **`X-Request-ID` hardening.** The logger middleware accepts an inbound client-supplied value unvalidated, so a caller can choose the correlation ID of its own audit events. Real, but P1 task 1.10 — it matters once EM-16 promotes `request_id` to the trace correlation ID.
- **The `component_workflows.go` defect from the scope doc's §1.** The file does not exist on `main` (deleted by `c30c5a4e`, the helper by `a64ea04d`). **There is nothing to fix.** It survives only as the review rule in deliverable 0.6.
- **Everything from P2 onward** — schema versioning, `producer`, `origin`, first-class tenancy, the attribute registry, transports, durable queue, read path, UI.

---

## 4. MCP: a second middleware at the protocol layer

This section is new relative to the phased plan. It matters for two reasons: **MCP is enabled by default** (`MCPDefaults()` → `Enabled: true`, `internal/openchoreo-api/config/mcp.go`), so the unaudited surface ships on; and it corrects a premise the scope doc's EM-11 and PR-4 rest on.

**P0's wiring does not reach MCP.** `/mcp` is registered directly on `baseMux` (`main.go:217`) with its own chain — `mcpLoggerMw → mcpAuth401Mw → jwtMiddleware` (`:211-217`). The middleware slice P0 modifies feeds only `gen.HandlerWithOptions`, so MCP is untouched.

**Three independent reasons the existing middleware cannot simply be added to that chain:**

| Blocker | Evidence |
|---|---|
| **Route granularity** | Every MCP call is a single `POST /mcp`. `Resolve` matches on method + `r.URL.Path` (`resolver.go:71-87`), so all MCP traffic collapses to one action. *Which* tool ran lives in the JSON-RPC body, which the middleware never reads — and SE-1 forbids body logging. |
| **Streaming** | `pkg/mcp/server.go:63` uses `mcp.NewStreamableHTTPHandler`. The audit `responseWriter` (`middleware.go:31-50`) implements neither `http.Flusher` nor `http.Hijacker`, so wrapping it breaks streaming — the same defect that pushed `/exec/` out of the OpenAPI chain. |
| **Session lifecycle** | In a stateful session one HTTP POST can carry many JSON-RPC messages, and the session outlives the request (`withSessionQueryParams`, `pkg/mcp/server.go:82-96`). One-event-per-request does not map to one-event-per-action. |

**The premise correction.** The scope doc concludes MCP auditing is *"a matter of routing MCP-served requests through the same middleware and carrying `origin: mcp` through, not a second set of action definitions."* Verified otherwise: `mcphandlers.CreateProjectRelease` calls `h.services.ProjectReleaseService.CreateProjectRelease(ctx, namespaceName, &pr)` (`internal/openchoreo-api/mcphandlers/project_release.go:53`) — the **same service method** the OpenAPI handler calls. MCP is a **sibling entry point onto the shared service layer**, not a wrapper over the HTTP API. An MCP-driven mutation therefore emits **zero** events, and no middleware placement changes that.

**The answer is a second middleware at the MCP protocol layer, not service-layer emission.** An earlier draft of this section recommended emitting from the shared service layer and suppressing the duplicate. That is unnecessary: the MCP SDK exposes a receiving-middleware hook that sits at the MCP boundary, exactly analogous to the HTTP middleware at the HTTP boundary.

| Capability | Evidence |
|---|---|
| **Ordering within one `AddReceivingMiddleware` call, first-listed-is-outermost** | *(Corrected post-implementation — see note below.)* `server.AddReceivingMiddleware(auditMw, filterMw)` (`pkg/mcp/server.go:82`) is a single call listing audit first, which the go-sdk wraps outermost — the opposite convention from oapi-codegen's last-is-outermost `Middlewares` slice. `server_audit_test.go` confirms audit **observes the authz filter's denials** |
| Tool name and **full raw argument JSON** | `req.GetParams().(*mcp.CallToolParamsRaw)` → `.Name`, `.Arguments`. A *superset* of the tool closure's view — for the 42 scope-collapsed tools there is no typed args struct at all |
| **Per-request** HTTP headers | `req.GetExtra().Header` is the live header map of the POST carrying that call (`$SDK/mcp/streamable.go:1143-1149`); `mcpLoggerMw` is outermost in the MCP chain, so `X-Request-ID` is present |
| Result and error | `err == nil && !IsError` → success; `err == nil && IsError` → tool failure; `err != nil` → protocol error |
| Subject | `auth.GetSubjectContextFromContext(ctx)` is **ctx-based** (`auth/interface.go:43`), so it works here. `jwt.GetClaims` is request-based and does not |

**Double-counting never arises.** HTTP audits by route, MCP audits by tool, and no request traverses both — `/mcp` sits outside the slice `gen.HandlerWithOptions` applies, and even inside it `r.Pattern` would be `/mcp`, which has no operation.

**What MCP does require is a declared binding table**, because the tool↔operation mapping is not derivable: 21 tools fan out to two operations via an MCP-only `scope` argument, 10 collapse two tools onto one operation, and `create_component_release` targets `generateRelease` rather than `createComponentRelease`. P0 declares bindings only for the operations §5's definitions already cover; the full 69-entry table is P1.

**One gap P0 must close for `result: denied` to work on MCP.** `pkg/mcp/tools/filter.go` returns bare `fmt.Errorf` at its three denial sites — no sentinel, no `*jsonrpc.Error` — so an outer middleware cannot distinguish a denial from a failure except by string matching. Deliverable 0.11 adds sentinels.

---

## 5. Deliverables

### 0.1 — Extract the middleware chain, and re-wire audit into it

Add to `internal/openchoreo-api/api/handlers/handler.go`, beside `InitJWTMiddleware` (`:36`):

```go
// APIMiddlewareOptions carries the dependencies APIMiddlewares needs.
type APIMiddlewareOptions struct {
    Logger         *slog.Logger
    AuthMiddleware func(http.Handler) http.Handler // auth.OpenAPIAuth(...) in production
    Audit          config.AuditConfig
}

// APIMiddlewares returns the ordered middleware chain for the generated OpenAPI routes.
//
// oapi-codegen applies these last-to-first, so the last entry is the outermost
// wrapper. Execution order is:
//
//	logger → auth → audit → webhookRawBody → handler
//
// loggerMiddleware is outermost so it captures all responses including 401s from
// auth. audit sits inside auth so SubjectContext is populated (EM-7); auditing
// auth *failures* needs a second, outer instance and is deliberately P1.
// webhookRawBodyMiddleware stays innermost so HMAC validation sees the raw bytes.
//
// This is the single definition of the chain. cmd/openchoreo-api/main.go supplies
// dependencies and owns no ordering, and TestAuditMiddlewareWired drives exactly
// what this function returns. Rebuilding this chain anywhere else — including in a
// test helper — silently un-guards audit, which is how #2588 removed it unnoticed.
func APIMiddlewares(opts APIMiddlewareOptions) []gen.MiddlewareFunc {
    auditLogger := audit.NewLogger(opts.Logger, "openchoreo-api")
    resolver := audit.NewActionResolver(apiaudit.GetActionDefinitions())
    auditMw := audit.NewMiddleware(auditLogger, resolver, opts.Audit.Enabled)

    loggerMw := apilogger.LoggerMiddleware(opts.Logger.With("component", "openapi"))

    return []gen.MiddlewareFunc{
        WebhookRawBodyMiddleware,
        auditMw.Handler,
        opts.AuthMiddleware,
        loggerMw,
    }
}
```

Imports needed: `apiaudit "github.com/openchoreo/openchoreo/internal/openchoreo-api/audit"`, `"github.com/openchoreo/openchoreo/internal/server/middleware/audit"`, `apilogger "github.com/openchoreo/openchoreo/internal/server/middleware/logger"`. No import cycle — `internal/openchoreo-api/audit` imports only the shared middleware package.

`cmd/openchoreo-api/main.go:220-230` then collapses to:

```go
handler := gen.HandlerWithOptions(strictHandler, gen.StdHTTPServerOptions{
    BaseRouter: baseMux,
    Middlewares: openapihandlers.APIMiddlewares(openapihandlers.APIMiddlewareOptions{
        Logger:         logger,
        AuthMiddleware: authMiddleware,
        Audit:          cfg.Audit,
    }),
})
```

Delete the now-redundant `loggerMiddleware` local at `main.go:196`, keeping `authMiddleware` (`:197`) since it is passed in. Move the ordering comment block (`:220-226`) into the `APIMiddlewares` doc comment — the ordering rationale must live with the ordering.

### 0.2 — Delete the 19 dead action definitions

Pure deletion from `internal/openchoreo-api/audit/definitions.go`; the 6 survivors in §2.2 need no edit. Add a file-level comment so the short list is not mistaken for coverage:

```go
// The definitions below cover only the routes verified to exist on main. 108 of the
// 114 state-modifying routes in api/gen/server.gen.go are NOT yet audited — the full
// rewrite and the CI gate that keeps this honest are P1 (tasks 1.5, 1.6).
```

### 0.3 — Fix `extractActor`, and stop `"<nil>"` becoming an identity

Two changes, and the second is not optional — see §2.3a and §2.3b for why this deliverable carries more weight than its diff size suggests.

**(a) Remove the entitlement gate.** In `internal/server/middleware/audit/middleware.go:126-130`, drop the `len(subjectCtx.EntitlementValues) > 0` condition, assign from `subjectCtx.ID` whenever it is usable, fall back to `"unknown"` otherwise, and replace the misleading comment. Satisfies **EM-7**.

**(b) Treat an absent `sub` as absent, not as the string `"<nil>"`.** Because `validateClaims` never requires `sub` (§2.3b), `resolver.go:91` can produce the literal `"<nil>"`, which (a) would then record as a real actor identity. Fix at the source in `internal/server/middleware/auth/jwt/resolver.go:90-95` — read the claim explicitly rather than through `fmt.Sprintf`:

```go
// sub is not guaranteed present: validateClaims checks only iss and aud.
// Leave ID empty rather than rendering "<nil>", which downstream consumers
// (the audit middleware is the only one today) would treat as a real identity.
subject, _ := claims["sub"].(string)
```

and leave `ID` empty when the claim is missing or not a string. Then have `extractActor` fall back to `"unknown"` on an empty `ID`, so an unidentifiable subject is recorded as unidentifiable rather than as a fabricated ID.

Also add a **defensive guard in `extractActor`** rejecting `"<nil>"` even if it arrives from some other path. Belt-and-braces is justified here: a fabricated actor identity in an audit trail is a correctness failure that no downstream consumer can detect.

> **Scope note.** The phased plan assigns this to P2 task 2.8. It moves here because P0's change (a) is what makes it reachable and harmful, and because P0 is already editing this code path. Mark 2.8 as delivered by P0 when updating the phased plan.

Add `internal/server/middleware/audit/middleware_test.go` (the package has no tests today) with `TestExtractActor`:

| Input | Expected |
|---|---|
| no subject context | `type=anonymous`, `id=anonymous` |
| subject with `ID` set, `EntitlementValues` empty | `id` = the subject ID **(the regression under test)** |
| subject with empty `ID` | `id=unknown` |
| subject with `ID` = `"<nil>"` | `id=unknown` **(not `"<nil>"`)** |

And in `internal/server/middleware/auth/jwt/resolver_test.go`, a case asserting that claims without `sub` yield an empty `ID` rather than `"<nil>"`.

**Do not** make the identity claim configurable. EM-7 fixes it to `sub`, and §2.3a records why that is the right constraint rather than a missing feature.

### 0.4 — The regression guard

The phased plan specifies this as *"a Go test in `internal/openchoreo-api/audit/` that asserts the audit middleware is present in the chain."* **That would guard nothing** — that package cannot see `main.go`'s chain, and a test that builds its own chain proves the middleware works, not that production wires it. The regression that occurred would still pass. Hence:

1. **Refactor `newTestHTTPHandler`** (`http_test_helpers_test.go:29`) to build its chain via `APIMiddlewares`, passing `injectTestSubject` in the `AuthMiddleware` slot and `config.AuditConfig{Enabled: true}`. Every existing `*_http_test.go` then exercises the production ordering, which also proves audit's `responseWriter` wrapper breaks nothing across ~15 existing suites.
2. **Add `newTestHTTPHandlerWithLogger(t, services, logger)`**; have `newTestHTTPHandler` delegate to it with `slog.Default()`.
3. **Add `internal/openchoreo-api/api/handlers/audit_wiring_test.go`** with `TestAuditMiddlewareWired`:
   - build a `bytes.Buffer` logger — `slog.New(slog.NewJSONHandler(&buf, nil))`;
   - build a project bundle on that logger, reusing `newTestScheme`, `allowAllPDP` and `seedProject`;
   - `POST /api/v1/namespaces/<testNS>/projects` via `doRequest`;
   - assert the captured JSON contains one `"msg":"AUDIT-LOG"` record with `action=create_project`, `category=resource`, `result=success`, `actor.id=test-user` (nested under the `actor` group), and a populated `resource` object.

**State the invariant in the test's doc comment:** the guard is real only because production and test share one chain constructor. A future "simplification" that rebuilds the chain in the helper silently un-guards audit.

### 0.5 — Minimal `AuditConfig`

New `internal/openchoreo-api/config/audit.go`:

```go
// AuditConfig defines audit logging settings. Per-category enablement, delivery
// mode, suppression and transports are P5 (task 5.9).
type AuditConfig struct {
    // Enabled enables audit event emission.
    Enabled bool `koanf:"enabled"`
}

func AuditDefaults() AuditConfig { return AuditConfig{Enabled: true} }
```

Register in `Config` (`config.go:15-30`) as `Audit AuditConfig \`koanf:"audit"\`` and in `Defaults()` (`:33-42`). **No `Validate` method** — a bare bool has nothing to validate, and an empty one is noise. Do not add it to `Config.Validate()`.

**Read the toggle inside the middleware, not in the chain builder.** Add an `enabled bool` field to `audit.Middleware`, take it as the third parameter of `NewMiddleware`, and short-circuit as the first statement of `Handler`:

```go
if !m.enabled {
    next.ServeHTTP(w, r)
    return
}
```

The middleware therefore stays in the chain unconditionally. This matters: no configuration path can un-wire audit, and the guard test needs no special construction to be meaningful. `NewMiddleware` has no other callers, so the signature change is safe.

Operator surface:
- `cmd/openchoreo-api/config.yaml` — an `audit:` block with the key documented in the file's existing commented style.
- `install/helm/openchoreo-control-plane/values.yaml` — `audit: { enabled: true }` under `openchoreoApi.config`.
- `install/helm/openchoreo-control-plane/templates/openchoreo-api/configmap.yaml` — render it alongside the existing `mcp:` block.

**`helm-generate` is part of `make code.gen`.** Editing `values.yaml` without regenerating fails `make code.gen-check`.

### 0.6 — Populate `resource` on the six audited handlers

Without this, every event emits `resource: null` and the demo shows a half-populated event. Add `audit.SetResource(ctx, &audit.Resource{Type: …, Name: …})` on the success path of:

| Handler | File |
|---|---|
| `CreateProject` | `internal/openchoreo-api/api/handlers/projects.go:48` |
| `CreateEnvironment` | `internal/openchoreo-api/api/handlers/environments.go:49` |
| `CreateDataPlane` | `internal/openchoreo-api/api/handlers/dataplanes.go:52` |
| `CreateSecret` | `internal/openchoreo-api/api/handlers/secrets.go:64` |
| `UpdateSecret` | `internal/openchoreo-api/api/handlers/secrets.go:125` |
| `DeleteSecret` | `internal/openchoreo-api/api/handlers/secrets.go:157` |

`Type` takes the resource kind — `project`, `environment`, `dataplane`, `secret`.

> **Review rule, carried from PR #1465.** Place the call **after** the `err != nil` check, never immediately after the service call. In PR #1465, `setAuditResource(ctx, "component_workflow_run", workflowRun.Name, …)` sat before the error check and would panic on a `(nil, err)` return. `CreateProject` (`projects.go:64-78`) has exactly that shape — the service call is followed by five error branches before `created` is safe to dereference. Roughly 18 call sites shared this shape in that diff, and P1 re-creates them, so the rule matters beyond these six.
>
> **Partly machine-enforced since P1 (added 2026-08-24).** P1 grew to 78 `SetResource` call
> sites across 35 handler files, which is well past what a review rule can hold, so
> `internal/openchoreo-api/api/handlers/audit_setresource_ast_test.go` now parses the handler
> package and asserts the call is **present**:
>
> - `TestHandlers_BodyCarriedCreatesCallSetResource` — every operation with no
>   `RESTResourceParam` must have a `(*Handler)<OperationID>` method that calls
>   `audit.SetResource`. With no path parameter there is no fallback, so a missing call audits
>   an empty resource on **every success** and no runtime test would catch it.
> - `TestHandlers_UpdatesCallSetResource` — the mirror for `update_*`, guarding the UID.
>
> **Presence, not position.** The AST test cannot tell whether the call sits before or after
> the `err != nil` check, so the rule above still needs human review for ordering — but the
> more common omission is now caught mechanically. P2's `resource.uid` work and P3's attribute
> layer both inherit this gate.
>
> > **The rule gained a second half, and "presence, not position" is now the live gap
> > (2026-08-30).** PR [#4571](https://github.com/openchoreo/openchoreo/pull/4571) found that at
> > **73 of the 78** sites the call sat after the `err != nil` check *and also after* the
> > `convert[...]` step that marshals the object into its generated response type. A mutation
> > that **succeeded** and then failed to convert returned 500 with an empty `resource` — an
> > audit record saying nothing changed when something had. All 73 moved above the conversion.
> >
> > **The rule is now: after the error check, *before* the response conversion.** It binds every
> > call site added from here on, observer's included. Because the AST gate still checks only
> > presence, nothing prevents a regression — filed as **task 1.16** in the P1 plan's §0.4.

**Do not add tenancy.** First-class `org` / `project` / `component` / `environment` fields are P2 (EM-5). Introducing `AddMetadata` tenancy keys here creates work for P2 task 2.6 to undo.

---

## 5b. Deliverables — the core framework

> **Scope extension, 2026-08-13.** Deliverables 0.1–0.6 shipped as `6bd147a1`. P0 now also carries the **surface-agnostic core, the configuration model and both adapters**, so that everything after P0 is an improvement on a settled foundation rather than a reshaping of it. Coverage and enforcement — all 114 REST operations, all 69 MCP bindings, the generator, the validation gate — remain P1.
>
> The cut is **core-then-coverage**. All the architectural risk sits here: whether the two-adapter model holds, whether the MCP middleware observes denials, whether `SetResource` reaches through the MCP context, whether one config genuinely serves both surfaces. P0 answers that on the 12 operations §5 already covers, where a mistake is cheap. P1 is then bulk mechanical work behind a gate.

### 0.7 — The surface-agnostic core

`Operation`, `Envelope`, `Emitter` and `PolicySet` in `internal/server/middleware/audit/`, with no HTTP or MCP types in the core:

```go
type Operation struct {
    ID           string   // "createProject" — the OpenAPI operationId, canonical key
    Action       string   // "create_project"
    ResourceType string   // "projects" — derived, so every event carries it
    Category     Category // stamped, not operator-configurable
}

type Envelope struct {   // everything a surface adapter must supply
    Origin    Origin      // api | mcp
    Actor     Actor
    Result    Result
    Resource  *Resource
    RequestID string
    SourceIP  string
    Metadata  map[string]any
}

func (e *Emitter) Emit(ctx context.Context, op *Operation, env Envelope)
```

`Emitter` and `PolicySet` must be **immutable after construction** — concurrent `tools/call` on one MCP session run in parallel goroutines (`$SDK/mcp/server.go:1429-1431`).

### 0.8 — The configuration model

Four blocks, each keyed on its natural unit. P0 implements blocks 1–2; blocks 3–4 are present, empty and documented as owned by P3 and P5.

```yaml
audit:
  enabled: true                       # 0.5, shipped

  # 1. WHAT EXISTS — code, not operator config: definitions, MCP bindings,
  #    exemptions. `category` is stamped from resource kind and is NOT
  #    operator-configurable: it is immutable in published records and is
  #    what backends key retention on, so reclassifying splits history.

  # 2. POLICIES — ordered, first match wins, as Kubernetes resolves levels.
  defaults:
    publish:    true
    pre_action: false        # reserved, P10a
    delivery:   async        # reserved, P10b — only `async` in P0
    retries:    3            # reserved, P10b

  policies:
    - match: { categories: [permission] }
      set:   { delivery: blocking, pre_action: true }
    - match: { actions: [create_workflow_run], actor_types: [service_account] }
      set:   { publish: false }

  # 3. INFORMATION — reserved for P3
  actor_claims: []
  attributes: {}
  redact: []

  # 4. SINKS — reserved for P5
  sinks:
    log: { enabled: true }
```

**Selector admissibility.** A selector is legal only if it passes two gates. **Timing:** `publish` is decided after the handler returns and may match on outcome; `pre_action` and `delivery` are decided before the action, so anything knowable only afterwards is unavailable. **Surface parity:** one rule must behave identically on both surfaces, so a selector available on only one is excluded.

| Selector | `publish` | `pre_action` / `delivery` |
|---|---|---|
| `categories`, `resources`, `operations`, `actions`, `origins` | ✓ | ✓ |
| `actor_types` | ✓ | ✓ |
| `actors`, `entitlements` | **rejected** | ✓ |
| `results` | ✓ | **✗** — not known in time |
| arbitrary JWT claims | **✗** | **✗** — `jwt.GetClaims` is request-based, so unavailable on MCP |

**`actors` and `entitlements` are escalation-only, enforced at config validation.** SE-2 says producers cannot suppress their own events; a suppression rule keyed on entitlement means anyone obtaining that group becomes unaudited, and group membership is usually delegated. A wrong match on an *escalation* rule yields more auditing — fail-safe — so the restriction is directional. `actor_types` stays permitted for suppression, since `service_account` is not something a human subject can assume.

Follow `openchoreo-api`'s structured koanf pattern so `--dump-config` verifies the merged result. **Do not follow Observer's config pattern** — it is env-only with a hand-maintained ~40-entry mapping.

### 0.9 — REST adapter on the core

Refactor the middleware onto `Emitter`. Replace the regex resolver with an **exact-map lookup on `r.Pattern`** — verified available in the middleware position, since `ServeMux.ServeHTTP` sets `r.Pattern`, `r.pat` and `r.matches` (`$GOROOT/src/net/http/server.go:2826`) and `WithContext` shallow-copies the struct (`request.go:372-375`). This retires two failure modes outright rather than guarding them: the silent `regexp.Compile` skip (§2.7) and first-match-wins shadowing.

**Guard an empty `r.Pattern` so it fails loudly.** Silently emitting nothing is the failure class this work exists to remove.

### 0.10 — MCP bindings for the P0 operation set

The 12 operations `definitions.go` declares, bound to their MCP tools where one exists, each declaring the argument that carries the resource name. Enough to prove the mechanism without the full 69-entry table. Note the argument names are ad hoc — only 12 of 37 identifier-carrying tools align mechanically with their REST path parameter, so the name must be **declared per binding**, never derived.

#### 0.10a — The binding key must carry scope (added 2026-08-19)

**As shipped, a binding cannot express a scope-collapsed tool, and a collision is silent.** This is a core-shape defect, so it belongs in P0 rather than P1 — P0's whole rationale is that later phases improve a settled core rather than reshape it, and a binding table that cannot represent 21 of the 69 tools is not settled.

**The mismatch.** REST has two distinct operations per scope — `CreateComponentType` and `CreateClusterComponentType`, different authz actions, different resource types. MCP collapses them into **one** tool that routes on a `scope` argument (`pkg/mcp/tools/scoped_component_types.go:107-133`: `namespace:` → `PEToolset.CreateComponentType`, `cluster:` → `PEToolset.CreateClusterComponentType`). 21 tools do this.

**Why it breaks today.** `OperationDef` carries one `MCPToolName`, and `MCPBindings` builds the table with a bare map write (`operation_def.go:53-57`):

```go
bindings[d.MCPToolName] = MCPBinding{...}   // two defs, one tool name → second wins, no error
```

The adapter then looks up on the tool name alone (`mcpaudit/audit_middleware.go:38`, `bindings[toolName]`), so **both** scopes resolve to whichever def sat later in the slice. The failure is mis-attribution, not omission: a namespaced create would be recorded as `create_cluster_component_type` on `clustercomponenttype` — a platform-wide change that never happened — while the real one becomes invisible. For an audit trail that is worse than silence.

Nothing catches it. `MCPBindings` doesn't error; `definitions_test.go:35` asserts unique operation **IDs**, which differ; and `BuildPatternMap` never sees MCP bindings. It is inert only because none of P0's 6 bindings collide.

**The change.**

1. **Key bindings on `(toolName, scope)`.** Add a `MCPScope` field to `OperationDef` — empty for the 48 non-collapsed tools, `"namespace"` / `"cluster"` for the two halves of each fan-out. Keep the discriminator an **opaque string in core**: `internal/server/middleware/audit` must stay free of MCP-SDK and `pkg/mcp/tools` imports, so the constants stay on the adapter side.
2. **Resolve scope in the adapter before lookup**, using the same raw-argument read `extractResourceArg` already performs. **Apply MCP's own default:** `resolveScope("")` returns `ScopeNamespace` (`pkg/mcp/tools/helpers.go:62-71`), so an absent `scope` must resolve to the namespace binding — otherwise every default-scope call silently misses its binding and emits nothing. Reference `tools.ScopeNamespace` / `tools.ScopeCluster` rather than re-literalling the strings; `mcpaudit` already imports that package for the denial sentinels.
3. **Fail loudly on a duplicate key.** `MCPBindings` currently cannot report one, so give it an error return (or validate in `NewMiddleware`, which already panics on a nil `Operation`) and assert it in a test. Land this **with** the shape change, not after — it is the only thing that will catch the 21 fan-outs being added wrongly later.
4. **Prove it on one fan-out now.** None of P0's 12 operations is scope-collapsed, so add the `create_component_type` pair — or a test-only fixture — so the mechanism has a live case. A shape change with no colliding data is untested by construction.

**Before fixing the key shape, sweep the other 20.** Verified for `create_component_type`: the discriminator is `scope`, values exactly `namespace`/`cluster`, invalid values rejected by `resolveScope`. Confirm no scope-collapsed **write** tool uses a third value or a differently-named argument before committing to a two-value string discriminator.

### 0.11 — Sentinel errors in `pkg/mcp/tools/filter.go`

Export `ErrNotAuthorized` and `ErrNoSubject`; wrap with `%w` at the three denial sites. Without this, `result: denied` is unachievable on MCP.

### 0.12 — MCP adapter

A receiving middleware registered via `AddReceivingMiddleware`. **Corrected post-implementation:** the shipped code uses one call listing audit first — `server.AddReceivingMiddleware(auditMw, filterMw)` — not a second, separately-registered call; within-one-call ordering (first-listed-is-outermost) is what makes it outermost and able to see denials, verified by `server_audit_test.go`. Emits only for `tools/call` on a bound tool; `initialize`, `ping`, `tools/list` and notifications pass through.

**The shared-constructor invariant applies here too.** Deliverable 0.4's guard is real only because production and tests both call `APIMiddlewares`. The MCP adapter needs its own exported constructor, driven by both `pkg/mcp/server.go` and a wiring test.

**Scope resolution belongs here, per 0.10a.** The adapter reads the `scope` argument out of the raw call arguments and looks up `(toolName, scope)`, defaulting an absent value to `namespace` to match `resolveScope`. Note the ordering constraint this creates: scope must be resolved **before** the pre-call resource seed, because the two halves of a fan-out have different `ResourceType` values and the seed is what a denied call reports.

### 0.13 — Resource resolution on both surfaces, and EM-4b

`resource.type` from the operation; `resource.name` from `r.PathValue` (REST) or the declared argument (MCP); `SetResource` as the shared escape hatch. Verified: the ctx the MCP middleware passes to `next` reaches the handler untouched —

```
handleReceive → mh(ctx, …)     $SDK/mcp/shared.go:169
  → callTool(ctx, req)         $SDK/mcp/server.go:732
    → st.handler(ctx, req)     $SDK/mcp/server.go:743
      → th(ctx, req)           $SDK/mcp/server.go:315
        → h(ctx, req, in)      $SDK/mcp/server.go:337
          → mcphandlers.X(ctx, …)
```

— so `audit.SetResource(ctx, …)` works identically on both surfaces. **Result-parsing does not work and must not be relied on:** `mutationResult` (`mcphandlers/transform.go:92-104`) returns only `name`, `action` and `namespace`, with no UID.

Also encode **EM-4b** in the emitter: a denied attempt at a *resolved* operation keeps that operation's category with `result: denied`. *(Revised 2026-08-19: the second half originally read "`access` is only for rejections that never resolve" — there is no `access` category. A rejection that never resolves is distinguished by `result: unauthenticated`, added in P1 task 1.0b.)*

---

## 6. Task breakdown

| # | Task | Repo | Reqs | Dep |
|---|---|---|---|---|
| 0.1 | Extract `APIMiddlewares`; re-wire audit into the OpenAPI chain | oc | PR-1 | — |
| 0.2 | Delete the 19 dead action definitions | oc | PR-1 | 0.1 |
| 0.3 | Fix `extractActor`'s entitlement gate + stale comment; stop an absent `sub` rendering as `"<nil>"`; add `TestExtractActor` | oc | EM-7 *(+ P2's 2.8, pulled forward)* | — |
| 0.4 | Route the test harness through `APIMiddlewares`; add `TestAuditMiddlewareWired` | oc | PR-3 *(partial)* | 0.1 |
| 0.5 | `AuditConfig` toggle, read inside the middleware; config + Helm surface | oc | OP-1 | 0.1 |
| 0.6 | `audit.SetResource` on the six audited handlers | oc | PR-1 | 0.2 |

**Shipped** (the restoration work; folded into `431092a7` by the rebase — see §0). The core-framework tasks below extend P0 per §5b.

| # | Task | Repo | Reqs | Dep |
|---|---|---|---|---|
| 0.7 | Surface-agnostic core: `Operation`, `Envelope`, `Emitter`, `PolicySet` | oc | — | 0.1 |
| 0.8 | Configuration model: `defaults` + ordered `policies` + selector validation | oc | OP-1, OP-3 | 0.7 |
| 0.9 | REST adapter on the core; exact-map resolver on `r.Pattern` | oc | PR-1 | 0.7 |
| 0.10 | MCP bindings for the P0 operation set | oc | PR-4 | 0.7 |
| 0.11 | Sentinel errors in `pkg/mcp/tools/filter.go` | oc | PR-6 | — |
| 0.12 | MCP adapter + exported constructor + wiring test | oc | PR-4 | 0.7, 0.10, 0.11 |
| 0.13 | Resource resolution on both surfaces; EM-4b in the emitter | oc | PR-1, EM-4b | 0.9, 0.12 |

All tasks are in `openchoreo`. 0.1 gates the rest; 0.3 and 0.11 are independent and can land first.

**Closed 2026-08-19.** `431092a7` and `f676c146` shipped 0.7–0.13 (cited as `96fe08ca` and `8c8b33b7` before the rebase — §0); `0ad4c9bd` shipped 0.10a–0.10e (below), closing the remaining core-shape defect. P0 is now fully shipped, and every P0 exit criterion in §8 is met.

| # | Task | Repo | Reqs | Dep | Status |
|---|---|---|---|---|---|
| **0.10a** | Sweep the 21 scope-collapsed write tools to confirm the discriminator is always a `scope` argument with values `namespace`/`cluster` | oc | PR-4 | — | ✅ Confirmed: every scope-collapsed write tool (`authz_role`, `authz_role_binding`, `component_type`, `project_type`, `resource_type`, `trait`, `workflow` × create/update/delete = 21) routes through the shared `registerScopedWriteTool`/`registerScopedSingleResourceTool` helpers in `scoped_helpers.go`, which all call `resolveScope` uniformly. No exceptions, no third value, no differently-named argument. |
| **0.10b** | Add `OperationDef.MCPScope`; key bindings on `(toolName, scope)`, discriminator an opaque string in core | oc | PR-4 | 0.10a | ✅ Landed as `MCPBindingKey{ToolName, Scope}` (a composite struct key, not a nested map) — `internal/server/middleware/audit/mcp_binding.go`. `MCPBindings(defs)` now returns `(map[MCPBindingKey]MCPBinding, error)`. |
| **0.10c** | Resolve scope in the MCP adapter before lookup **and before the resource seed**; default absent → `namespace` per `resolveScope` | oc | PR-4 | 0.10b | ✅ `resolveBinding` in `pkg/mcp/mcpaudit/audit_middleware.go`: tries the unscoped key first (the common case, no scope argument extraction needed), falls back to extracting `scope` and looking up `{ToolName, Scope}`, defaulting absent/unrecognized to `tools.ScopeNamespace`. Runs before the resource seed, satisfying the ordering constraint. |
| **0.10d** | Make a duplicate binding key a loud failure (error from `MCPBindings` or validation in `NewMiddleware`) + test | oc | PR-3 | 0.10b | ✅ `MCPBindings` itself errors on a colliding key (collision is only detectable at map-construction time, before the second def's write would silently overwrite the first — validating post-construction in `NewMiddleware` can't recover the lost information). `apiaudit.MCPBindings()` and `main.go` propagate the error via the same `logger.Error`+`os.Exit(1)` pattern as every other startup check. |
| **0.10e** | Bind one real fan-out (`create_component_type`) so the mechanism has a live colliding case | oc | PR-4 | 0.10c, 0.10d | ✅ Built as a **test-only fixture** (`TestMCPBindings_ScopeCollapsedFanOut`, `TestMCPBindings_DuplicateKeyErrors` in `operation_def_test.go`), not real production coverage — `create_component_type` isn't one of P0's 12 audited REST operations, and wiring it for real would mean adding REST handler `SetResource` calls outside P0's stated scope. The fixture proves collision-detection and scope-based resolution without that scope creep. **Superseded by P1:** `create_component_type` is now genuinely bound to both halves in `internal/openchoreo-api/audit/mcp_bindings.go`, so the mechanism has real production coverage as well as the fixture. |

> **A third hazard, found while implementing 0.10b and not anticipated by 0.10a's sweep
> (added 2026-08-24).** The key-shape fix above closes the *collision* case — two defs writing
> `(tool, scope)` with the same scope. It does **not** close the case of one tool name carrying
> **both an unscoped and a scoped binding**: `{tool, ""}` and `{tool, "cluster"}` are distinct
> map keys, so no collision fires. But the adapter's `resolveBinding`
> (`pkg/mcp/mcpaudit/audit_middleware.go`) deliberately tries the **unscoped key first** as the
> common case, so the scoped binding becomes **silently unreachable** and every call resolves to
> the unscoped operation regardless of the `scope` argument the caller sent. That is the same
> mis-attribution failure 0.10a describes, reached by a different route, and it is exactly what
> a table of 90 binding keys hand-maintained across 21 fan-outs and 10 aliases will eventually
> get wrong.
>
> Fixed as `scopeTracker.checkAndRecord` (`internal/server/middleware/audit/operation_def.go`),
> which rejects the mix in **either arrival order** and is applied at **both** entry points —
> `MCPBindings` and `MergeMCPAliases`. Guarded by `TestMCPBindings_RejectsMixedScopedAndUnscoped`,
> `..._ScopedFirst`, and the two `MergeMCPAliases` variants (against an existing binding, and
> between two aliases in one call).
>
> Worth recording because 0.10a's sweep asked the right question — *"is the discriminator
> always a `scope` argument with values `namespace`/`cluster`?"* — and got a correct answer,
> yet the shape still had a second reachable failure mode that only surfaced once the full
> table was populated. The lesson for later phases: a lookup with a **fallback order** needs
> its own invariant, separate from key uniqueness.
**On the 401-vs-403 conflation.** P0 leaves it in place. `determineResult` maps `401 || 403 → ResultDenied` and `classifyResult` maps `ErrNoSubject, ErrForbidden → ResultDenied`, so "unauthenticated" and "authorization refused" are recorded identically on the 12 audited operations. A `ResultUnauthenticated` value fixes it, and was briefly filed here as `0.14` on the grounds that it corrects shipped behaviour — **moved to P1 (task 1.0b) on 2026-08-19**, since P0 is otherwise closed and the change's main consumer is P1's PR-7 coverage. Reopening a shipped phase for it buys nothing.

This closed task 1.0 in [`openchoreo-audit-logging-p1-plan.md`](./openchoreo-audit-logging-p1-plan.md) — P1 now starts the 69-entry table against a shape that can already hold a scope-collapsed tool.

---

## 7. Demo

```sh
# 1. Audit events emit again, with a populated resource
make k3d.update.openchoreo-api
curl -X POST http://openchoreo.localhost:8080/api/v1/namespaces/default/projects \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"name":"demo","spec":{...}}'
make k3d.logs.openchoreo-api | grep AUDIT-LOG | tail -1 | jq
#    → actor.id = the sub claim (not "unknown"), action = create_project,
#      category = management, result = success, resource.type = project
#      (category was written as `resource` here before the taxonomy was
#       settled — see post-implementation correction 4)

# 2. A token with a present-but-empty entitlement claim is attributed correctly
make test GO_RUN_ARGS="-run TestExtractActor ./internal/server/middleware/audit/..."

# 3. The guard earns its place — remove the audit entry from APIMiddlewares
make test GO_RUN_ARGS="-run TestAuditMiddlewareWired ./internal/openchoreo-api/api/handlers/..."
#    → FAIL: no AUDIT-LOG record emitted for POST .../projects

# 4. The toggle silences emission without changing request handling
#    set openchoreoApi.config.audit.enabled=false, redeploy
#    → the request still succeeds; no AUDIT-LOG record

# ── Core framework (§5b) ──────────────────────────────────────────

# 5. The same operation on both surfaces, one event each
#    drive create_project via REST, then via /mcp with the same token
#    → same action and category, differing only in origin — exactly one
#      event per call, not two, and MCP streaming still works

# 6. One policy applies identically on both surfaces
#    policies: [{ match: {actions: [create_project]}, set: {publish: false} }]
#    → neither surface emits

# 7. MCP denials are distinguishable from failures
#    call a bound tool the subject lacks permission for  → result=denied
#    call one that fails inside the service layer        → result=failure

# 8. Config validation rejects an unsafe policy at startup
#    policies: [{ match: {entitlements: [x]}, set: {publish: false} }]
#    → startup error: entitlement selectors may not suppress

# 9. Concurrency
make test GO_RUN_ARGS="-race -run TestMCPAuditConcurrent ./pkg/mcp/..."

# Pre-PR gate
make code.gen && make lint-fix && make test && make code.gen-check
```

---

## 8. Exit criteria

1. A real request against one of the 6 routes emits a **complete** audit event on `main` — actor, action, category, result and resource all populated.
2. `actor.id` is the `sub` claim regardless of whether the entitlement claim carries values, **and a token with no `sub` records `unknown` rather than `"<nil>"`**.
3. Removing the audit middleware from `APIMiddlewares` **fails a test**, and the test drives the same constructor production uses.
4. `definitions.go` contains only patterns that match a live route, with the P1 hand-off recorded in the file.
5. `audit.enabled=false` silences emission without altering request handling; the middleware remains in the chain.
6. The existing ~15 `*_http_test.go` suites pass through the production chain unchanged.
7. §3's non-goals are recorded where P1's planner will read them.

**Core framework (§5b):**

8. The same operation driven via REST and via MCP emits **exactly one** event each, with the same action and category, differing only in `origin`.
9. One `policies` entry takes effect identically on both surfaces.
10. MCP authz denials emit `result: denied`, distinguishable from `failure`.
11. Config validation rejects at startup: a policy setting `category`; a suppression rule using `actors` or `entitlements`; a `pre_action`/`delivery` rule using `results`; any rule using an arbitrary JWT claim.
12. `resource.type` is present on every event; `resource.name` on both surfaces wherever derivable.
13. Removing **either** adapter from its constructor fails a test.
14. `-race` passes with concurrent MCP tool calls on one session.

**Repos.** openchoreo.

---

## Appendix A — Corrections to the phased plan's P0 section

1. **25 action definitions, not 24.** 6 match a live route, 19 are dead. The earlier `paste`-based count dropped six entries; the `awk` parse in §2.2 is reliable.
2. **Task 0.2 is a deletion, not a restoration-with-edits.** The phased plan says *"restore the handful that match live routes"*, implying editing. The 6 survivors match including parameter names and need no change.
3. **Task 0.4 as specified would guard nothing.** A test in `internal/openchoreo-api/audit/` cannot see `main.go`'s chain, and one that builds its own chain does not guard production. It requires the `APIMiddlewares` extraction and the shared-constructor invariant (§5, 0.1 and 0.4).
4. **Task 0.6 is new.** The phased plan's P0 emits `resource: null` because no handler calls `SetResource`, which contradicts its own demo.
5. **The resolver's `regexp.Compile` silent-`continue` trap is not currently sprung** — all live route parameters are valid Go capture-group names (§2.7). It is a P1 gate requirement, not a P0 blocker.
6. **P2 task 2.8 moves into P0** (§2.3b, deliverable 0.3b). `validateClaims` never requires `sub`, so an absent claim renders as the literal `"<nil>"`. P0's own removal of the entitlement gate is what makes that string reachable as an `actor.id` — turning an honest `"unknown"` into a fabricated identity. Deferring it would ship two phases of audit events carrying a fake actor.
7. **The identity claim is deliberately not configurable** (§2.3a), and audit is the **first and only consumer of `sub`** — `authz.GetAuthzSubjectContext` drops `ID`, so every authorization path ignores it. Worth adding to the scope doc under EM-7: the reason both actor defects survived is that nothing read the field until now.
8. **MCP is a sibling of the API, not a wrapper over it** (§4). `mcphandlers` calls the same service methods as the OpenAPI handlers, so EM-11/PR-4's *"audited once, at the API layer"* does not hold. Note **MCP is enabled by default**, so the unaudited surface ships on.
9. **MCP does not need service-layer emission** — the phased plan's §2.4 conclusion, and an earlier draft of §4 here. A **protocol-level receiving middleware** is the right hook: it observes the authz filter's denials, needs no service-layer plumbing, and cannot double-count. What it does need is a **declared binding table**, because the tool↔operation mapping is not derivable.
10. **P0's scope extended on 2026-08-13** to carry the core framework and both adapters (§5b). The phased plan's P0 section describes only deliverables 0.1–0.6.
11. **`category` is stamped from resource kind and is not operator-configurable.** It is immutable in published records and is what backends key retention on, so reclassifying splits history. Operator flexibility lives in the ordered `policies` list instead — see §5b, deliverable 0.8.

---

## Appendix B — Hand-off to P1

> **P1 has since shipped (2026-08-24).** This appendix is the hand-off as written; four of its
> items were consumed by P1 and are marked below. `openchoreo-audit-logging-p1-plan.md` §0 is
> the authoritative as-built record.

What P1 inherits from P0, stated as facts rather than intentions:

- **Coverage baseline: 12 of 114** state-modifying REST operations audited, plus MCP bindings for those only. P1 adds the remaining 102 and the full MCP binding table. ✅ **Done** — 114 audited (112 generated + `Exec`/`Wirelogs` hand-declared) and 69 tools / 90 binding keys.
- **The core is settled.** `Operation`, `Envelope`, `Emitter` and `PolicySet` exist and are surface-agnostic; P1 adds coverage and the generator, not new machinery.
- **Two shared constructors**, `APIMiddlewares` and the MCP adapter's, are the only places middleware is composed. P1's second, outer HTTP instance for PR-7 belongs in the first.
- **The configuration model exists** with blocks 1–2 implemented and 3–4 reserved. P1 adds no config mechanism; P3 fills information, P5 fills sinks, P10 implements the reserved `pre_action`/`delivery`/`retries` keys.
- **`category` is stamped, not configurable.** ~~P1 moves the stamping into the generator and renames the set to `resource`/`permission`/`access`, dropping `observability`.~~ **Superseded 2026-08-19:** the shipped taxonomy (`management`/`authorization`) is now the permanent one — see the post-implementation corrections note above. P1 moves the stamping into the generator but does **not** rename the category set; there is nothing left to rename to.
- **The regex resolver is gone**, replaced by exact-map lookup on `r.Pattern`. P1's gate therefore does *not* need the "every pattern compiles" and "no shadowing" assertions the phased plan specified — those failure modes no longer exist.
- **Audit sits inside auth**, so token rejections still produce no event. P1, two-instance design. ✅ **Done** — `audit.NewUnauthenticatedMiddleware` outside auth, both instances composed in `APIMiddlewares`, mutually exclusive by construction.
- **`getRequestID` still trusts an inbound `X-Request-ID`.** P1. ✅ **Done** — `audit.RequestIDFromHeader` accepts the header only if it parses as a UUID, and counts rejections (`audit.RequestIDRejections()`).
- **`/exec/` and wirelogs are unaudited**, with reasons in §3. ✅ **Audited** — `responseWriter.Unwrap()` removed the `http.Hijacker`/`Flusher` blocker, so both go through the standard adapter via `NewMiddlewareForRoutes`. **A live exec session and wirelogs tail were never verified end to end** — filed as P1 task 1.14.
- **The review rule in 0.6** — `SetResource` after the error check — applies to every call site P1 adds, and P1 adds ~38 on REST creates. **P1 added 78**, across creates, updates and deletes, and turned the rule's presence half into an AST gate — see 0.6's addendum.
- **P2 task 2.8 is already done** (the `"<nil>"` fix, pulled into 0.3b). P2 should re-scope to adding `actor.subject` / `session_id` / `issuer` as explicit fields on top of it.
- **`actor.id` is `sub` and only `sub`**, by EM-7. If a human-readable label is wanted in the portal, it is a new field in P2/P3, not a configurable identity claim — the reasoning is in §2.3a.
- **`sub` is still not a required claim** at the JWT boundary. P0 makes an absent `sub` record honestly as `unknown`; whether the middleware should instead *reject* a token with no `sub` is a genuine authentication-policy question, out of P0's scope and worth raising with whoever owns `internal/server/middleware/auth/jwt`.
