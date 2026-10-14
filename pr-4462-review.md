# Review — PR #4462, policy-driven audit logging framework

**Reviewed at:** `56477caa` (PR head == local `feat/audit_logging` HEAD, verified) · **Base:** `main` · 52 files, +3855/−510
**Reviewed against:** [`openchoreo-audit-logging-p0-plan.md`](./openchoreo-audit-logging-p0-plan.md) §5b deliverables 0.7–0.13 and §8 exit criteria
**Build/tests:** `go build ./...` clean; `go test` green on `internal/server/middleware/audit`, `internal/openchoreo-api/{config,audit,api/handlers}`, `pkg/mcp/...`

---

---

# Follow-up pass — 2026-08-19

**Reviewed:** working tree on `feat/audit_logging` (uncommitted, 31 files changed vs `56477caa` + `internal/openchoreo-api/mcphandlers/audit_bindings_test.go`)
**Gates:** `go build ./...` clean · `go test` green across `internal/server/middleware/audit`, `internal/openchoreo-api/...`, `pkg/mcp/...`, `internal/config` · `make lint` → **0 issues**, license + newline checks pass · `make code.gen-check` fails *only* because the tree is dirty (no generated file is modified — verified `git status --porcelain` against `api/gen/`, `config/crd/`, `install/helm/`, `zz_generated`, `samples/`); it will pass once committed.

## Status of the original findings

| # | Finding | Status |
|---|---|---|
| 1 | Audit gated by app log level | **Fixed** — `forceLevelHandler` overrides `Enabled`; `TestLogEvent_NotGatedByAppLogLevel` proves emission at `LevelError` |
| 2 | `resource.type` had four values | **Fixed, better than asked** — `Resource.Type` deleted outright; `Event.ResourceType` stamped only in `buildEvent`; REST seeds via new `Operation.RESTResourceParam` |
| 3 | MCP bindings unvalidated | **Closed in CI** — `TestMCPBindings_MatchRegisteredTools` cross-checks against `toolsets.Register`'s `perms` |
| 4 | `result` diverges across surfaces | **Documented, not fixed** — accepted, with a real root cause (go-sdk discards handler-error identity before the middleware sees it) |
| 5 | Category taxonomy | **Decided** — `management`/`authorization` is permanent; plan §Appendix B rewritten, P1 rename dropped |
| 6 | `panic` on empty `r.Pattern` | **Fixed** — logs and passes through; test inverted to match |
| 7 | Missing `Unwrap()` | **Fixed** |
| 8 | Half the selectors unvalidated | **Fixed** — `resources`/`operations`/`actions` from `apiaudit.GetOperations()`, `actor_types` from new `SecurityConfig.KnownActorTypes()` |
| 9 | `Sink.LogEvent` returns no error | **Not addressed, not recorded** |
| 10 | `pkg/mcp.NewHTTPServer` uncallable externally | **Not addressed, not recorded** |
| 11 | `source_ip` client-controlled | **Documented** — trusted-proxy assumption stated on `SourceIPFromHeader` |

Small items: `"<nil>"` guard + test ✅ · `APIMiddlewares` returns an error, `main` uses `logger.Error`/`os.Exit` ✅ · `NewEmitter` nil-`PolicySet` guard ✅ · `AuditData.Metadata` documented as P3-reserved ✅ · false observer comment corrected ✅ · `defaults.publish: false` allowlist mode documented ✅ · `ErrPDPFailure` now wraps the PDP error ✅ · plan corrections recorded, including the §4 `AddReceivingMiddleware` mechanism ✅.

Independently verified: all four `RESTResourceParam` values (`projectName`, `dpName`, `envName`, `secretName`) match the generated route patterns in `server.gen.go`; all six `MCPToolName`/`MCPResourceArg` pairs still match `pkg/mcp/tools/{project,pe}.go`.

## Residual items

None block merge. Three are new consequences of the fixes; two are the untouched findings.

1. **`forceLevelHandler` loses its force through `WithAttrs`/`WithGroup`.** Empirically confirmed: `NewLogger(appLoggerAtLevelError).slogger.With("k","v").Info("AUDIT-LOG", …)` emits **nothing** — the embedded `slog.Handler`'s `WithAttrs` returns the *inner* handler, dropping the wrapper. Unreachable today (`LogEvent` never derives a logger), but it is a silent-drop trap inside the fix for a silent drop. Six lines to close:
   ```go
   func (h *forceLevelHandler) WithAttrs(as []slog.Attr) slog.Handler {
       return &forceLevelHandler{Handler: h.Handler.WithAttrs(as)}
   }
   func (h *forceLevelHandler) WithGroup(name string) slog.Handler {
       return &forceLevelHandler{Handler: h.Handler.WithGroup(name)}
   }
   ```
2. **`Event`'s JSON tags no longer describe the emitted shape.** Confirmed by marshalling both:
   ```
   json.Marshal(Event) → ..."resource_type":"project","resource":{"id":"uid-1","name":"p1"}...
   Logger.LogEvent     → ..."resource":{"type":"project","id":"uid-1","name":"p1"}...
   ```
   Only the slog sink exists today, so nothing is broken — but a P5 webhook sink marshalling `Event` directly would publish a different shape than the log sink, which is exactly what `buildEvent`'s "sinks can't disagree about one event's identity" comment exists to prevent. Either nest `ResourceType` in the marshalled form or have the logger render `resource_type` flat.
3. **`RESTResourceParam` is unvalidated** — the one field the fix introduced that has no construction-time check, while its two siblings (`ID` via `BuildPatternMap`, `MCPToolName` via the new binding test) both have one. A typo yields an empty `resource.name` on exactly the denied/failed events the seed was added to populate. `BuildPatternMap` already holds the `*openapi3.Operation`, so checking the parameter exists is a few lines in the loop it already runs. `TestAuditMiddlewareWired_DeniedRequestCarriesResource` covers `UpdateProject` only, so `dpName`/`envName`/`secretName` are unguarded. Same argument applies to `MCPResourceArg`, which the new binding test checks the tool name of but not the argument.
4. **`KnownActorTypes()` iterates a map**, so the `must be one of: …` startup error lists actor types in a different order each run — and would flake any test asserting that message. `slices.Sort` on the subject names before returning.
5. **Findings 9 and 10 are still open and now undocumented.** Neither is a merge blocker (the `Logger` sink can't fail; `main.go` is the only `NewHTTPServer` caller), but both are foundation-shaped decisions that get more expensive per phase: the sink contract changes every sink implementation once P5 adds a fallible one, and the `pkg/` → `internal/` dependency decides whether `pkg/mcp` is public surface at all. Worth a line each in the plan's P5/hand-off section recording the decision to defer, so P1's planner doesn't rediscover them.

## Verdict on the pass

Both must-fixes are properly fixed, and finding 2 was fixed at the root rather than at the symptom — deleting `Resource.Type` so there is structurally no field a handler can set inconsistently is a better answer than the "pick one convention" I suggested. The two deliberate non-fixes (4, 5) are each recorded with reasoning I find sound. **Ship it**, with the residual items as follow-ups.

---

# Original review

## Verdict

The structural core is the right shape and the two things that were hardest to get right are right: `BuildPatternMap` replacing the regex resolver is a genuine improvement over the plan (it converts two silent-failure modes into construction-time errors), and deriving `Operations()` and `MCPBindings()` from one `operationDefs()` table is better than the plan's two-table sketch. The shared-constructor invariant is real on both surfaces — `APIMiddlewares` and `NewHTTPServer` are each driven by a wiring test that would fail if the middleware were dropped.

Two defects should be fixed before this lands as the foundation. Both are cheap now and expensive later, because both concern the *published event shape and reachability* — the parts the plan itself argues are immutable once events reach a backend.

---

## Must fix

### 1. Audit emission is silently gated by the application log level

`Logger.LogEvent` emits at `Info`:

```go
// internal/server/middleware/audit/logger.go:80
l.slogger.Info("AUDIT-LOG", attrs...)
```

and `main.go:210` hands the emitter the *application* logger, which is level-filtered:

```go
// cmd/openchoreo-api/main.go:94
logger := logging.NewWithComponent(cfg.Logging.ToLoggingConfig(), version.Get().Name)
// internal/logging/logging.go:26-31 — opts.Level = parseLevel(cfg.Level)
auditEmitter := audit.NewEmitter("openchoreo-api", auditPolicies, audit.NewLogger(logger))
```

`openchoreoApi.config.logging.level` is an operator-facing Helm value (`values.yaml:1891`, default `info`). So:

```yaml
openchoreoApi:
  config:
    audit: { enabled: true }      # audit says on
    logging: { level: warn }      # ...and zero events are emitted
```

No error, no warning, no signal. This is exactly the #2588 failure class the PR exists to close, now reachable through routine config rather than a refactor. It defeats OP-1, and it makes exit criterion 5 misleading — `audit.enabled` is *not* the only kill switch.

The guard tests can't catch it: `audit_wiring_test.go:76` builds `slog.New(slog.NewJSONHandler(&buf, nil))`, whose nil options default to `LevelInfo`.

**Fix direction:** the `Logger` sink should own a handler that is not gated by `cfg.Logging.Level` — either construct it in `main.go` independently of the app logger, or have `NewLogger` wrap the given handler in one that forces the level through. Add a test at `LevelWarn` that still sees the record.

### 2. `resource.type` takes four different values for one field

The same logical field is populated from two different sources with two different conventions:

| Path | `resource.type` | Source |
|---|---|---|
| REST success | `"project"` | handler `SetResource` (`projects.go:89`) |
| REST denied (403) | *`resource` absent entirely* | `NewAuditContext(r.Context(), nil)` — `middleware.go:105` |
| MCP success | `"project"` | handler overwrites the seed (`mcphandlers/projects.go:92`) |
| MCP denied | `"projects"` | plural seed from `op.ResourceType` survives (`audit_middleware.go:51`) |

Two consequences:

- **Policy selectors don't match published events.** `Selector.Resources` is compared against `Operation.ResourceType` (`policy.go:141`), i.e. the plural form, while events carry the singular. An operator who writes `match: {resources: [projects]}` and then greps the emitted stream for `resource.type=projects` finds nothing but denied MCP calls.
- **REST loses the resource on exactly the events that need it most.** The PR description justifies MCP's pre-call seed because "a PDP-denied call never reaches the handler that would otherwise set it." That argument applies verbatim to REST — a 403 from the authz layer emits `resource: null`. Plan deliverable 0.13 specified `resource.type` from the operation and `resource.name` from `r.PathValue` on REST; neither landed, so **exit criterion 12 is not met on REST**.

Note that `audit_wiring_test.go:104` asserts `resource["type"] == "project"`, and the MCP denial test (`server_audit_test.go:304-312`) checks `resource["name"]` but never `resource["type"]` — so the tests lock in the singular form on one path while leaving the divergent path unasserted. Both need updating with the fix.

**Fix direction:** pick one convention for `ResourceType` (I'd take the singular, since it's what 10 of the 12 handler call sites already emit and what reads correctly as `resource.type`), then seed the REST context the way MCP does:

```go
ctx, auditData := NewAuditContext(r.Context(), &Resource{
    Type: op.ResourceType,
    Name: r.PathValue(op.RESTResourceParam),  // declared per operation, like MCPResourceArg
})
```

This also removes the class of bug where a P1 handler forgets `SetResource` and the event silently ships with no resource at all — 102 more routes are coming.

---

## Should fix before this is the foundation

### 3. MCP bindings are never validated against registered tools

REST fails loudly when an `Operation.ID` doesn't match the spec (`pattern_map.go:44`). MCP has the identical failure mode with no equivalent check: a typo or a renamed tool in `MCPToolName` means that tool is silently unaudited forever.

The data to check against is already in hand at the call site:

```go
// pkg/mcp/server.go:70-73
perms, toolToToolsets := toolsets.Register(server)
auditMw := mcpaudit.NewMiddleware(auditOpts)     // nothing cross-references perms
```

I verified all six bindings are correct today (`create_project`/`name`, `update_project`/`project_name`, `delete_project`/`project_name`, `create|update|delete_environment`/`name` — all match `pkg/mcp/tools/project.go` and `pe.go`), so this is an asymmetry to close, not a live bug. But the plan's whole thesis is that silent coverage loss must be structurally impossible, and P1 grows this table to 69 entries.

Worth extending to `MCPResourceArg` too — validating it against the tool's `InputSchema` properties would catch the `env_name`-vs-`name` class of mistake that `server_audit_test.go:362` is currently guarding by hand.

### 4. `result` diverges across surfaces for the same denial

REST classifies by status code, so any 403 becomes `denied` (`middleware.go:139`). MCP classifies only the two sentinels from the MCP-layer filter (`audit_middleware.go:74-77`); a denial raised by the *service* layer surfaces as a tool error and is recorded as `failure`.

Concrete repro: a session opened with `?filterByAuthz=false` skips `filterCallTool` entirely (`QueryParamFilterByAuthz`, `server.go:32`), so every authorization denial in that session is recorded as `failure` on MCP and `denied` on REST. Exit criterion 10 holds only for MCP-layer denials — worth saying so explicitly, or teaching `classifyResult` about the service layer's authorization error type.

### 5. The category taxonomy deviates from the plan, and one value has no producer

The PR ships `management` / `authorization` (`types.go:23-29`). The plan's §3 says P0 keeps `resource`/`auth`/`observability` and P1 renames to `resource`/`permission`/`access` (EM-4a/4b/4c). This is neither. `CategoryAuthorization` has zero producers today.

This needs deciding now rather than discovering later, because the plan's own argument (§5b block 1, Appendix A item 11) is that `category` is immutable in published records and is what backends key retention on. If `management`/`authorization` is the final taxonomy, drop the P1 rename from the phased plan; if not, rename now — otherwise the field gets migrated twice.

### 6. Panicking on an empty `r.Pattern` is the wrong failure mode at request time

```go
// internal/server/middleware/audit/middleware.go:88-99
if r.Pattern == "" {
    m.logger.Error(...)
    panic("audit: request has no matched route pattern (r.Pattern is empty)")
}
```

Fail-loud is the right instinct, but this check sits *before* the "is this route audited?" lookup, so it fires for every request — including the 102 routes with no audit definition. Any `BaseRouter` that isn't a Go 1.22+ `ServeMux` (`gen.StdHTTPServerOptions.BaseRouter` accepts an interface) turns the whole API into 500s on a dropped connection, presenting as an application bug rather than an audit misconfiguration. Tests will never see it, because the helper builds `http.NewServeMux()`.

Construction time is the right place for this class of check, and `BuildPatternMap` already covers it. At request time, `logger.Error` + pass through gives a better operational signal than a severed connection.

### 7. `responseWriter` should implement `Unwrap()`

`middleware.go:62-80` wraps the `ResponseWriter` without an `Unwrap() http.ResponseWriter`, so `http.ResponseController` can't reach `Flusher`, `Hijacker`, or the deadline setters — the same defect that pushed `/exec/` out of this chain. Three lines, and P1 puts ~102 more routes behind this wrapper:

```go
func (rw *responseWriter) Unwrap() http.ResponseWriter { return rw.ResponseWriter }
```

### 8. Half the selector fields are validated, half aren't

`categories`, `origins` and `results` go through `config.MustBeOneOf` (`config/audit.go:129-158`). `resources`, `operations`, `actions` and `actor_types` are passed through raw, so `actions: [create_projct]` yields a rule that silently never matches. The operation table is reachable from `internal/openchoreo-api/config` without a cycle (`config` already imports `internal/server/middleware/audit`; `internal/openchoreo-api/audit` imports only that too), so cross-checking against `apiaudit.GetOperations()` is straightforward. `actor_types` could be checked against the configured `security.subjects` keys.

Directionally this currently fails safe — `defaults.publish` is `true`, so a non-matching suppress rule logs more, not less — which is why I'd file it as consistency rather than a defect. But it becomes a real hole the moment anyone runs allowlist mode (see the one-liner on `defaults.publish` below).

### 9. `Sink.LogEvent` returns no error

```go
type Sink interface { LogEvent(event *Event) }
```

For a framework whose premise is that audit loss must be detectable, an error-less sink interface bakes in undetectable loss. It's fine for the slog sink; it isn't for the webhook and durable-queue sinks P5 adds, and changing the signature then touches every sink and `Emitter.Emit`'s fan-out loop. `Emit` also already takes a `ctx` it explicitly ignores — worth deciding the sink contract (error + ctx) now, even while the only implementation can't fail.

### 10. `pkg/mcp.NewHTTPServer` is no longer callable from outside the module

The new required parameter names an `internal/…` type:

```go
func NewHTTPServer(toolsets *tools.Toolsets, pdp authzcore.PDP, auditOpts mcpaudit.MiddlewareOptions) http.Handler
```

`MiddlewareOptions.Emitter` is `*internal/server/middleware/audit.Emitter`, which an external importer cannot construct, and passing the zero value panics (`middlewares.go:36`). Practical impact today is nil — `main.go:229` is the only caller — so the real question is whether `pkg/mcp` is intended as public surface. If yes, functional options (`NewHTTPServer(toolsets, pdp, WithAudit(...))`) keeps the signature source-compatible and makes audit genuinely optional; if no, the package arguably belongs under `internal/`.

### 11. `source_ip` is client-controlled

`SourceIPFromHeader` (`adapter.go:71-86`) trusts `X-Forwarded-For` and `X-Real-IP` with no trusted-proxy check, so a caller behind a proxy that doesn't overwrite XFF chooses its own recorded source IP. The plan schedules `X-Request-ID` hardening for P1 (task 1.10) but says nothing about this sibling field, which is arguably the more forensically load-bearing of the two. At minimum document the trusted-proxy assumption at the function.

---

## Smaller items

- **The plan's `"<nil>"` defensive guard in `ExtractActor` was dropped.** Plan 0.3 called it non-optional and specified the test case. Only the resolver-side fix landed (`resolver.go:92`), and `ExtractActor` checks just `subjectCtx.ID != ""`. I grepped: `internal/server/middleware/auth/jwt/resolver.go:93` is the only production constructor of `auth.SubjectContext`, so the risk is genuinely low — but record the reasoning somewhere, since the plan's argument was that a fabricated actor ID is undetectable downstream.
- **Constructor error handling is inconsistent three ways in ~20 lines.** `main.go:205-209` does `logger.Error`/`os.Exit(1)` for the policy set; `APIMiddlewares` converts `audit.NewMiddleware`'s error into `panic` (`handler.go:89-91`); `mcpaudit.NewMiddleware` panics outright. Returning `([]gen.MiddlewareFunc, error)` from `APIMiddlewares` and letting `main` handle it uniformly reads better and gives operators a clean startup message instead of a stack trace.
- **`NewEmitter` accepts a nil `*PolicySet`**, which panics inside the deferred `EmitFromContext` — after the response is written, and it escapes the handler-panic `recover` in the same defer. A nil check in `NewEmitter` is cheaper than that failure.
- **`AuditData.Metadata` has no writer** anywhere in the repo since `AddMetadata` was deleted, but is plumbed through `EmitFromContext` → `Envelope` → `Event` → `Logger`. Either restore a setter or note it as reserved for P3.
- **`middleware.go:14` claims "openchoreo-api and observer both use it".** Observer doesn't — `internal/observer/mcp/server.go` has its own unrelated `NewHTTPServer` and no audit middleware at all.
- **`defaults.publish: false` bypasses the empty-match suppression guard.** `validatePolicy` rejects `{match: {}, set: {publish: false}}` as "would silence every event" (`policy.go:113`), but `audit.defaults.publish: false` achieves the same thing unchecked. I assume this is deliberate allowlist mode — if so, say so in the `PolicyDefaultsConfig` doc comment, because right now the two paths read as contradicting each other. If allowlist mode is supported, item 8 (unvalidated selector values) stops being fail-safe.
- **`ErrPDPFailure` discards the underlying PDP error** (`filter.go:186`) — pre-existing, but `%w`-ing both would help debug a PDP outage that now shows up in audit as `failure`.

---

## Plan corrections worth folding back

1. **§4 got the `AddReceivingMiddleware` ordering backwards.** The plan asserts each *call* wraps outside prior ones, and prescribes a second `AddReceivingMiddleware` call after the filter's. The PR uses within-one-call ordering (`server.AddReceivingMiddleware(auditMw, filterMw)`, `server.go:82`) with audit listed *first* to be outermost, and `server_audit_test.go` confirms it observes the filter's denials. Not a defect — the plan's §4 evidence table should be corrected.
2. **Exit criterion 12 is not met** on the REST surface (item 2).
3. **Deliverable 0.13's REST half is unimplemented** — `resource.type` from the operation and `resource.name` from `r.PathValue` (item 2).
4. **Deliverable 0.3's defensive `"<nil>"` guard is unimplemented** (smaller items).
5. **The category taxonomy diverged** from both the P0 and P1 sets the plan names (item 5); Appendix B's hand-off bullet on the P1 rename needs rewriting either way.
6. **Add a P0/P1 note that `audit.enabled` is not the only kill switch** until item 1 is fixed.

## What's better than the plan asked for

- `BuildPatternMap` erroring on an unmatched `operationId` *and* on a pattern collision — the plan only required retiring the regex resolver.
- One `operationDefs()` table deriving both surfaces' views, so a binding can't drift from its operation. The plan sketched these as separate artifacts (0.10 vs. §5's definitions).
- `Resource.ID` carrying the K8s UID, with `audit_wiring_test.go:157` asserting it survives a rename — not in the plan at all, and correct: names are mutable.
- Three sentinels rather than the plan's two, separating `ErrPDPFailure` from a policy denial so a PDP outage isn't recorded as if the user were denied.
- `UnmarshalStrict` on the `audit` subtree, so a typo under `policies[].match` fails startup instead of producing an empty (match-everything) selector.
- `defer`/`recover`/re-`panic` on both surfaces, so a panicking handler still produces a `failure` record.
