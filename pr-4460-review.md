# Review — PR #4460, audit every state-modifying operation and enforce it in CI

**Reviewed at:** `6b54b8c4` (branch `feat/audit-logging-p1` HEAD) · **Base:** `a7f1d9c1` (PR #4462) · 86 files, +4880/−272
**Reviewed against:** [`openchoreo-audit-logging-p1-plan.md`](./openchoreo-audit-logging-p1-plan.md) — §0 as-built record, §5 deliverables B1–B9, §8 exit criteria
**Date:** 2026-08-28

**Gates run:**

| Check | Result |
|---|---|
| `TestAuditCoverage` (7 assertions, 157/69 pins) | ✅ pass |
| `TestHandlers_BodyCarriedCreatesCallSetResource` / `_UpdatesCallSetResource` | ✅ pass |
| `go run ./tools/auditgen` → byte-diff vs committed `definitions.gen.go` | ✅ in sync |
| `TestCoverageMatrix_IsFresh`, `TestRender_Deterministic` | ✅ pass |
| `go test` across `tools/audit*`, `internal/server/middleware/{audit,logger}`, `pkg/mcp/...`, `internal/openchoreo-api/{audit,config,mcphandlers,api/handlers}` | ✅ pass |
| `go test -race ./internal/server/middleware/audit/... ./pkg/mcp/mcpaudit/...` | ✅ pass |

> Local note: `go` was failing with `compile: version "go1.26.0" does not match go tool version "go1.26.3"` because a stale `GOROOT` env var pointed at the gvm `go1.26` directory while the `go` binary on `PATH` was the 1.26.3 toolchain. `env -u GOROOT go test …` works. Not a repo problem.

---

# Follow-up pass 3 — 2026-08-30

**Reviewed:** working tree on `feat/audit-logging-p1`, 15 files changed vs `6b54b8c4` (+582/−244)
**Gates:** all tests green across `tools/...`, `internal/server/middleware/...`, `pkg/mcp/...`, `internal/openchoreo-api/...` · `auditgen` output byte-identical to the committed `definitions.gen.go` · `TestCoverageMatrix_IsFresh` green · **`golangci-lint` → same 6 `lll` issues, unchanged**

## HandleAutoBuild — exemption restored, accepted

Finding 1 is reverted by decision: `HandleAutoBuild` is back in `excludedOperationIDs` and `RESTExemptions`, and the per-action machinery built for it (`EmitEvent`, `SuppressDefaultEmit`, `AuditData.RequestID`/`SourceIP`, the generalized `operationOverrides`, `webhook_processor.go`'s `triggerAndAudit`, `NewWebhookProcessor`'s error return, `main.go`'s emitter hoist) is gone with it. The rationale recorded in the code — every event on that path carried `actor: anonymous/anonymous`, so the added framework surface wasn't worth it until the webhook payload's pusher identity is threaded through — is a reasonable call, and the comment marks it a deferral with a revisit condition rather than a settled exemption. Not relitigating it.

**The revert is clean.** `grep -rn "EmitEvent\|SuppressDefault" internal/ pkg/ cmd/ tools/` returns only the three prose mentions inside the exemption comment; no orphaned exported surface survives. Generated defs are back to 112, `definitions_test.go`'s 114/`notInSpec == 2` pins still hold untouched, and only `SetResult` — which exec needs independently — remains from that work. N1 (the bogus matrix row for `HandleAutoBuild`) and N2 (zero-component webhook, zero events) are both resolved by the revert.

### One thing to fix: the published reason string still doesn't survive `k8sClient.Create`

The Go comment above the entry is accurate and useful. The **reason string itself is unchanged**:

```go
"HandleAutoBuild": "An HMAC-authenticated CI webhook trigger, not a user action — there is no " +
    "meaningful actor identity to record.",
```

That string is what the gate checks and what `docs/audit/coverage-matrix.md:129` publishes — and it is the only part a reader of the matrix sees. It says "not a user action" and stops there, so someone auditing the audit coverage would reasonably conclude the endpoint doesn't change state. It does: `ProcessWebhook` → `TriggerWorkflow` → `k8sClient.Create` on a WorkflowRun (`workflowrun/service.go:940`).

The fix is one line, and it's the same criterion as before — the reason has to survive the `Create`, whichever way the decision goes. Something like:

> *"Creates a WorkflowRun per affected component, but is HMAC-authenticated rather than JWT-authenticated, so every event would carry `actor: anonymous`. Deferred until the webhook payload's pusher identity can be recorded — see the comment above."*

Related, minor: that comment names `audit.EmitEvent` and `audit.SuppressDefaultEmit` in the past tense, and neither exists in the tree any more. A reader who greps for them finds only this comment. Worth phrasing as "an earlier revision of this branch" so it doesn't read as a pointer to live code.

## Everything else from pass 2

| # | Finding | Status |
|---|---|---|
| 1 | `HandleAutoBuild` | **Exempted by decision** — reason string still needs the one-line fix above |
| 2 | `/exec/` (a) always `success` / (b) emitted at session end | ◐ **(a) fixed** via `SetResult`; **(b) still open and still undocumented** |
| 3 | Wirelogs namespace | ✅ **Fixed** — see N4 below on placement |
| 4 | Generated action names | ✅ **Fixed** |
| 5 | `logger.responseWriter` missing `Unwrap` | Open |
| 6 | Unauthenticated events publish by default | Open |
| 7 | MCP has no PR-7 outer instance | Open — now documented in the matrix |
| 8 | `init()` panic | Open |
| 9 | bindings→perms gate check | ✅ **Fixed** |
| N1 | Matrix row for `HandleAutoBuild` | ✅ Resolved by the revert |
| N2 | Zero-component webhook, zero events | ✅ Moot |
| N3 | `splitPascalCase` acronym mis-split | Open (minor) — nil-`emitter` half is moot |

**2(b) remains the one substantive gap.** `exec.go` still emits only when `ServeHTTP` returns from `<-done`, so there's no record while a shell is open and a mid-session `openchoreo-api` restart loses it. It isn't in *Known non-events* either — the three P3 entries landed, but this one didn't. Fix it or record it; leaving it in neither place is the only state that's wrong.

## N4 — wirelogs `SetResource` runs before the input validation

New this pass; I missed it in pass 2, when I checked only that the call existed.

`wirelogs.go:118` sets the resource immediately after `parseWirelogsPath`, but the length and RFC1123 checks on `namespace` and `environment` are at `:128-134`. So on the three malformed-input 400s, whatever the client put in the path lands verbatim in `resource.namespace` / `resource.name` — bounded only by `http.Server`'s `MaxHeaderBytes`, ~1MB by default.

This PR already treats that exact class as worth bounding elsewhere: `logger/middleware.go`'s `maxLoggedRequestIDLen = 64` and `truncateForLog` exist precisely so a client can't put an arbitrarily large attacker-controlled string into a log line on every request.

Cleanest fix is to move the `SetResource` call below the four validation blocks. The malformed-input 400s would then carry no namespace — which is exactly what the matrix already documents for every spec route (*"a malformed path parameter never reaches audit on REST"*), so it's surface parity rather than a loss. The 403 ownership mismatch and the `authzChecker == nil` 500, which are the exits that actually motivated the fix, both sit below the validation and keep their namespace either way.

## Lint — unchanged, still failing

Same six `lll` violations as pass 2, all pre-existing at `6b54b8c4` (verified there directly), none introduced or fixed since:

```
pkg/mcp/tools/types_test.go:19, :24
tools/auditgen/generator_test.go:224, :241, :242, :259
```

`lll` has no autofix, so `make lint-fix` will not clear them. `make lint` — and therefore the stated pre-PR gate — fails today.

## Verdict on the pass

The revert is disciplined: no dead surface, counts and pins back to baseline, generated output in sync. Of the four original blockers, three are fixed and the fourth is a recorded, reasoned deferral.

Remaining before merge: the six line wraps, the `HandleAutoBuild` reason string, N4's call placement, and a decision on 2(b) — fix or document.

---

# Follow-up pass 2 — 2026-08-28

**Reviewed:** working tree on `feat/audit-logging-p1`, 21 files changed vs `6b54b8c4` (+1149/−341)
**Gates:** all tests green (`tools/audit*`, `middleware/{audit,logger}`, `pkg/mcp/...`, `internal/openchoreo-api/...`) · `-race` clean on `middleware/audit` and `services/autobuild` · `go run ./tools/auditgen` → byte-identical to the committed `definitions.gen.go` · `TestCoverageMatrix_IsFresh` green · **`golangci-lint` → 6 issues, see below**

## ⚠️ `make lint` fails — and it has since the original commit

```
pkg/mcp/tools/types_test.go:19:1     lll  125 chars > 120
pkg/mcp/tools/types_test.go:24:1     lll  127 chars > 120
tools/auditgen/generator_test.go:224 lll  127 chars > 120
tools/auditgen/generator_test.go:241 lll  127 chars > 120
tools/auditgen/generator_test.go:242 lll  140 chars > 120
tools/auditgen/generator_test.go:259 lll  141 chars > 120
```

Verified directly against the committed HEAD (`git stash -u` → lint → `stash pop`): **all six reproduce at `6b54b8c4`.** This pass neither introduced nor fixed them.

`lll` has no autofix, so `make lint-fix` will not clear these — they need manual wrapping. The stated pre-PR gate `make code.gen && make lint-fix && make test && make code.gen-check` therefore fails today, and has for two commits. That's the more useful signal than the six line numbers: lint has not been run on this branch.

## What landed

**Finding 1 — `HandleAutoBuild` — fixed, and the design is right.** Removed from `excludedOperationIDs` and `RESTExemptions`; added to a generalized `operationOverrides` table (`Action: "trigger_autobuild"`, `ResourceType: "workflowrun"`). The mechanism chosen is better than a plain definition would have been: because one webhook can trigger a build for zero or more components, a single per-request event genuinely cannot represent it. So `AuditData` gains `Result`, `RequestID`, `SourceIP` and `SuppressDefault`; `EmitEvent` lets the service layer emit per action using ctx alone; `SuppressDefaultEmit` stops the middleware also emitting a resource-less top-level event; and `webhook_processor.go`'s new `triggerAndAudit` emits one event per component — success carrying the created WorkflowRun's `{Namespace, UUID, Name}`, failure carrying the attempted component's identity.

Three details worth calling out as correct rather than incidental:
- `markEmitted(ctx)` is called **before** the `SuppressDefault` early return (`middleware.go:176-182`), so suppressing the inner event does not cause the outer unauthenticated middleware to fire a spurious `unauthenticated` record.
- The panic path deliberately ignores `SuppressDefault`, with the reasoning written down — a crash after some per-action events still gets signalled.
- `NewWebhookProcessor` resolves its `*Operation` at construction and returns an error if absent, so a renamed `HandleAutoBuild` fails startup rather than going silently unaudited. `main.go` wires it and exits on error.

**Finding 3 — wirelogs namespace — fixed.** `wirelogs.go:111-118` now calls `SetResource` explicitly, with a comment explaining why the middleware's `namespaceName` seed can't reach it. `middleware.go:138-147`'s false invariant comment is corrected to name the `NotInOpenAPISpec` exception. `TestWirelogsHandler_SetsResourceNamespaceFromParsedPath` asserts both `resource.namespace` and `resource.name` through the `authzChecker == nil` 500 path — a real pre-authz exit, not a proxy for one.

**Finding 4 — action names — fixed.** `actionFromOperationID` word-splits the operationId's PascalCase, so `create_clustercomponenttype` becomes `create_cluster_component_type`, matching the MCP tool names for the same operations. `ResourceType` deliberately keeps its concatenated spelling, and the doc comment explains that Action and ResourceType are now not substrings of one another by construction — which is the right call, since `resource.type` is what `Selector.Resources` and published events key on.

The new `strings.HasPrefix(action, verb+"_")` cross-check is a good addition: the method/path-derived verb and the operationId's leading word are computed independently, and a disagreement now fails generation rather than producing a quietly wrong action.

**Finding 9 — hole closed.** The `schemas[key.ToolName]` lookup now precedes the `ResourceArg == ""` continue (`audit_coverage_test.go`), so all 4 empty-`ResourceArg` bindings get the registration check too.

**P3 — all three matrix entries landed** (`renderKnownNonEvents`): `NewSTDIO`, `?filterByAuthz=false`, and the MCP HTTP-401 gap, each with a real explanation.

## Still open

**Finding 2 is half fixed.** `SetResult` (new, `context.go:43-52`) covers both post-upgrade failure paths in `exec.go` — `buildGatewayExecURL` and the gateway dial — and `middleware.go` prefers it over `determineResult` when set. `TestMiddleware_Handler_SetResultOverridesStatusCode` proves it. **2(a) is closed.**

**2(b) is untouched.** The event still fires only when `ServeHTTP` returns from `<-done`, so there is no record while a shell is open, and an `openchoreo-api` restart mid-session still loses it entirely. If that's a deliberate deferral, it belongs in *Known non-events*; right now it isn't recorded anywhere.

Findings 5, 6, 7, 8 and the Minor list are unchanged. Finding 10 remains withdrawn.

## New findings from this pass

### N1 — the matrix row for `HandleAutoBuild` is wrong in three ways

`docs/audit/coverage-matrix.md` renders its resource identity as:

> *"handler-supplied on success (body-carried create; unavailable on a denied/failed request)"*

All three clauses are false for this operation: `triggerAndAudit`'s failure branch **does** carry the component's identity; it is not a body-carried create; and one request yields zero-or-many events rather than one. This is `renderRESTSection`'s generic `default` branch firing — its three-way switch (`RESTResourceParam != ""` / `NotInOpenAPISpec` / default) has no case for the new "handler emits its own events" shape.

Add a fourth case. The cleanest discriminator is a flag on `OperationDef` (`EmitsOwnEvents bool`), which would also give the AST-test exception something to key on instead of a second hand-maintained ID list.

### N2 — a webhook matching zero components now emits zero audit events

`SuppressDefaultEmit(ctx)` is called unconditionally after the loop (`webhook_processor.go:115`). If `affectedComponents` is empty, no `EmitEvent` fires and the middleware's own event is suppressed, so an HMAC-authenticated webhook call that reached the service layer and triggered nothing produces **no record at all**.

Not a regression — it was fully exempt before — but it is a genuine "one request, zero events" case: it's untested, and it's absent from *Known non-events*. Given that the gate now requires a written reason for every exemption, this shape deserves the same treatment. Either move the `SuppressDefaultEmit` call inside `if len(affectedComponents) > 0`, letting the middleware record the no-op call, or document it.

### N3 — two small things worth one line each

- **`NewWebhookProcessor` doesn't nil-check `emitter`.** Safe today (`main.go` always passes a real one, and `EmitEvent` no-ops when there's no audit data in context), but the constructor already validates `auditOp`, so guarding the other dependency is consistent and cheap.
- **`splitPascalCase` mis-segments acronym runs.** The doc comment says so, and I verified no current operationId contains one. The failure mode on the next `CreateOIDCProvider`-shaped ID is a silently ugly action string (`create_o_i_d_c_provider`), not an error — unlike every other derivation in this generator, which fails loudly. A check for two consecutive uppercase letters returning an error would match the file's own conventions.

## Status of the original findings

| # | Finding | Status |
|---|---|---|
| 1 | `HandleAutoBuild` creates WorkflowRuns while exempted | ✅ **Fixed** |
| 2 | `/exec/` always `success` (a) / emitted at session end (b) | ◐ **(a) fixed, (b) open** |
| 3 | Wirelogs `{namespace}` vs hardcoded `namespaceName` | ✅ **Fixed** |
| 4 | Generated action names | ✅ **Fixed** |
| 5 | `logger.responseWriter` missing `Unwrap` vs `findFlusher` | Open |
| 6 | Unauthenticated events publish by default, unnarrowable | Open |
| 7 | MCP has no PR-7 outer instance | Open (now documented in the matrix) |
| 8 | `init()` panic in a widely-imported package | Open |
| 9 | No bindings→perms check in the gate | ✅ **Fixed** |
| 10 | Discarded `Register` error | Withdrawn — was wrong |
| — | Minor list (14 items) | Open |
| N1 | Matrix row for `HandleAutoBuild` is wrong | New |
| N2 | Zero-component webhook emits zero events | New |
| N3 | nil `emitter`; acronym mis-split | New |

## Verdict on the pass

All four blocking findings are addressed or half-addressed, and the mechanism built for finding 1 — per-action `EmitEvent` plus `SuppressDefaultEmit`, with the panic path deliberately exempt — is a better answer than the one I proposed. The new tests exercise the fixed behaviour rather than restating it.

What stands between this and merge is smaller than what was fixed: `make lint` (pre-existing, six manual line wraps), N1's one-line matrix bug, and a decision on 2(b) and N2 — fix, or record in *Known non-events*.

---

# Follow-up pass 1 — 2026-08-28

**Reviewed:** working tree on `feat/audit-logging-p1`, one file changed vs `6b54b8c4` — `internal/openchoreo-api/mcphandlers/audit_coverage_test.go` (+86)
**Gates:** `TestAuditCoverage` green, all 8 subtests · `./bin/tools/golangci-lint run ./internal/openchoreo-api/mcphandlers/...` → **0 issues**

## What landed

**New gate assertion 7 — "every ResourceArg names a real tool argument."** `toolInputSchemaProperties` stands up a real `mcp.Server`, connects an in-memory client, and reads each tool's input schema back over `ListTools` — the same schema a real MCP client sees. `assertResourceArgsMatchToolSchemas` then checks every binding's non-empty `ResourceArg` against it.

This is a good addition and it closes a gap I did not flag. `RESTResourceParam` has had this check since P0 (`BuildPatternMap`, `pattern_map.go:64-67`), for the stated reason that a typo'd path parameter is invisible until it silently produces an empty `resource.name` on exactly the denied/failed events the seed exists to populate. `MCPResourceArg` is the same field on the other surface and had no equivalent check. Reading the schema back over the wire rather than re-deriving it is the right call — `AddTool` infers the schema by reflection, so the wire is the only place the real argument names exist.

It also partially closes **finding 9**: the `props, ok := schemas[key.ToolName]; if !ok` branch errors when a binding names a tool that isn't registered.

**Residual hole in that coverage:** the `binding.ResourceArg == ""` early `continue` skips the registration check too. Four bindings have an empty `ResourceArg` — `create_workload`, `create_release_binding`, `create_workflow_run` (`mcp_bindings.go:53,60,75`) and the `trigger_workflow_run` alias (`:177`) — so a stale tool name on any of those four is still caught only indirectly, via the `len(perms) == 157` pin. Moving the `schemas[key.ToolName]` lookup above the `ResourceArg == ""` check closes it, and finding 9 with it.

## Status of the original findings

| # | Finding | Status |
|---|---|---|
| 1 | `HandleAutoBuild` creates WorkflowRuns while exempted | **Not addressed** — blocking |
| 2 | `/exec/` always audited `success`, emitted at session end | **Not addressed** — blocking |
| 3 | Wirelogs `{namespace}` vs hardcoded `namespaceName` | **Not addressed** — blocking |
| 4 | Generated action names (`create_clustercomponenttype`) | **Not addressed** — blocking |
| 5 | `logger.responseWriter` missing `Unwrap` vs `findFlusher` | Not addressed |
| 6 | Unauthenticated events publish by default, unnarrowable | Not addressed |
| 7 | MCP has no PR-7 outer instance | Not addressed |
| 8 | `init()` panic in a widely-imported package | Not addressed |
| 9 | No bindings→perms check in the gate | **Mostly closed** by assertion 7; hole on the 4 empty-`ResourceArg` bindings |
| 10 | "`toolsets.Register`'s error is discarded" | **Withdrawn — the finding was wrong.** See below |
| — | Minor list (14 items) | Not addressed |

## Correction: finding 10 was wrong

`Toolsets.Register` returns **no error**. Its signature (`pkg/mcp/tools/register.go:333-336`) is:

```go
func (t *Toolsets) Register(s *mcp.Server) (
	perms map[string]ToolPermission,
	toolToToolsets map[string]map[ToolsetType]bool,
)
```

So `perms, _ := toolsets.Register(server)` discards a *map*, not an error, and there is nothing to check. My finding 10 — and the "partial registration silently under-reports the matrix" failure scenario built on it — does not exist. `golangci-lint` agreeing (0 issues, with `errcheck` enabled) is consistent with that, not with a suppressed warning.

The bare `toolsets.Register(server)` at `audit_coverage_test.go:45` is therefore fine as written; the discarded second return is genuinely unused there.

## One note on the new helper, not a finding

`ListTools` is paginated in the go-sdk, and the assertion reads only the first page (`toolsResult.Tools`, no `NextCursor` follow). `DefaultPageSize` is 1000 (`go-sdk@v1.4.1/mcp/server.go:36`) and the server is constructed with `nil` options, so all 157 tools land in one page and this is correct today. Worth knowing only because the failure mode if that ever changes is a confusing one: tools on later pages would be reported as *"no such tool is registered"* rather than as a pagination problem. A `t.Fatal` on a non-empty `NextCursor` would make it say what actually went wrong.

Two smaller things in the helper, both immaterial: it re-registers all 157 tools against a second server rather than reusing `registerAllToolsets`'s work, and the `serverSession` returned by `server.Connect` is never closed (only `clientSession` is). Neither matters in a test that runs once.

## Verdict on the pass

The one change is a real improvement, correctly implemented, and green. But it addresses roughly half of one finding from the "should fix" tier and none of the four blocking ones. If the intent was to respond to the review, the four blocking items — and finding 1 in particular, which contradicts the PR's own title — are still open.

---

# Original review

## Verdict

The enforcement machinery is well built and does what the plan claimed: generation is deterministic, `resourceCategories` is genuinely exhaustive in both directions, `MCPBindings`/`MergeMCPAliases` fail loudly on collisions and on scope-mixing, and `code.gen-check` will catch a new route. The comments carry real reasoning rather than restating the code.

The defects cluster in one place, and it is not the generated middle: **coverage decisions inherited from the plan's early census, and the routes wired in late.** The gate enforces that a decision was *made*, not that it was right, so those are exactly where it cannot help. Four items block.

---

## Blocking

### 1. `HandleAutoBuild` creates WorkflowRuns and is exempted on a reason that doesn't cover that

**Files:** `tools/auditgen/generator.go:26-29`, `internal/openchoreo-api/audit/exemptions.go:17-18`

The exemption reads:

```go
"HandleAutoBuild": "An HMAC-authenticated CI webhook trigger, not a user action — there is no " +
    "meaningful actor identity to record.",
```

Traced call path:

- `internal/openchoreo-api/api/handlers/webhook_handler.go:111` → `AutoBuildService.ProcessWebhook`
- `internal/openchoreo-api/services/autobuild/webhook_processor.go:89` → `workflowTrigger.TriggerWorkflow`
- `internal/openchoreo-api/services/workflowrun/service.go:823` `TriggerWorkflow` — its own doc comment: *"creates a new WorkflowRun from a component's workflow configuration. This is used by both the authorized API handler path and the webhook path (unauthz)"*
- `internal/openchoreo-api/services/workflowrun/service.go:940` — `s.k8sClient.Create(ctx, workflowRun)`

**Why it matters.** This is an externally reachable, JWT-unauthenticated endpoint that creates cluster resources which then pull and build arbitrary source. The *same* `TriggerWorkflow` reached via `CreateWorkflowRun` **is** audited, so the one path an operator would most want in the trail — "a webhook created a build" — is the one path with no record. And the reason is a non-sequitur: an absent *user* argues for recording `actor.type: "webhook"`, not for dropping the event. The framework already has the vocabulary (`ExtractActor` falls back to `anonymous/anonymous` for exactly this shape).

**Note the asymmetry in how this was reached.** `GenerateRelease` got precisely this scrutiny: the plan's B3 originally exempted it, someone checked, found `k8sClient.Create`, and gave it a real definition via `generateReleaseOverride` — with a doc comment explaining why. `HandleAutoBuild` inherited §2.2's *"HMAC webhook, not a user action → exempt"* and was never re-checked the same way.

**Fix.** Remove from `excludedOperationIDs` and `RESTExemptions`; add a definition (`Action: "trigger_autobuild"`, `ResourceType: "workflowrun"`, `Category: CategoryManagement`) plus a `SetResource` call from what `ProcessWebhook` returns. If the exemption is kept deliberately, the reason string must survive the fact that it calls `Create`.

`Evaluates`, the other REST exemption, is correct — `handlers/authz.go` batch-evaluates the PDP with no writes.

### 2. `/exec/` is always audited as `success`, and only when the session ends

**Files:** `internal/openchoreo-api/api/handlers/exec.go:192-264`, `internal/server/middleware/audit/middleware.go:154,161-177`

**(a) Result is always `success`.** `execUpgrader.Upgrade(w, r, nil)` (`exec.go:192`) hijacks the connection and writes the 101 directly to the raw conn — it never calls `WriteHeader` on the wrapped writer. `rw.statusCode` stays at its `http.StatusOK` default (`middleware.go:154`), so `determineResult` returns `ResultSuccess`. Every post-upgrade failure audits identically to a working shell: `buildGatewayExecURL` failing (`:200`) and the gateway dial failing (`:211`) both call `writeWSError` and return.

**(b) Emitted at session end, not session start.** `ServeHTTP` blocks on `<-done` (`:263`); the middleware's `defer` only runs after `next.ServeHTTP` returns. So there is **no audit record while a shell is open** — "who is in a production pod right now?" is unanswerable from the log — and an `openchoreo-api` restart mid-session loses the record entirely. On the most privileged operation the platform offers.

Task **1.14** (live-verify exec through the wrapper) would very likely have caught (a). This is the concrete reason not to close #4460 on it.

**Fix.** Emit a session-opened event immediately after `Upgrade` succeeds and let the existing defer emit session-closed. Add a way for a hijacking handler to set the result explicitly, since the status code can no longer change after the hijack.

### 3. Wirelogs audit events lose `resource.namespace` on every pre-authz exit path

**Files:** `internal/server/middleware/audit/middleware.go:136-150` vs `internal/openchoreo-api/api/handlers/exec_wirelogs_audit.go:20`

The middleware hardcodes the namespace path-parameter name:

```go
ctx, auditData := NewAuditContext(r.Context(), &Resource{
    Namespace: r.PathValue("namespaceName"),
    Name:      r.PathValue(op.RESTResourceParam),
})
```

with this comment three lines above:

> *"Every audited route is nested under `/namespaces/{namespaceName}/...`, so the path parameter name is fixed across all of them."*

**This commit falsifies that invariant.** `WirelogsRoutePattern` is `GET /api/v1/namespaces/{namespace}/environments/{environment}/wirelogs` — `{namespace}`, not `{namespaceName}`. Every OpenAPI route does use `{namespaceName}` (68 occurrences in the spec), so the hardcode was safe until wirelogs arrived. `r.PathValue("namespaceName")` now returns `""` for every wirelogs request.

`BuildPatternMap`'s guard against exactly this class of typo (`pattern_map.go:63-67`) is skipped, because `Wirelogs` carries `NotInOpenAPISpec: true` and returns early at `pattern_map.go:48`.

The only recovery is `withHierarchyNamespaceFallback`, which needs `AuthzChecker.Check` to have run. `wirelogsCheckRequest` does set `Hierarchy.Namespace` in all three branches, so success and PDP-denial events are fine — but the handler has **six exits before that check**:

- three 400s on name-regex validation (`wirelogs.go:118-133`)
- 400 on `resolveComponentProject` failure (`:152`)
- **403 on `ownerProject != project`** (`:158`) — a real authorization denial
- 500 on `authzChecker == nil` (`:143`)

Each emits with `resource: {type: environment, name: <env>}` and **no namespace**. The 403 is the one that matters: a cross-project wirelogs probe is precisely the event you want scopeable by tenant, and it is the one that arrives untenanted.

No test catches it — `TestExecWirelogsAuditMiddleware_ResolvesBothRoutes` asserts only `action` and `result`.

**Fix.** Either add `audit.SetResource(ctx, &audit.Resource{Namespace: namespace, Name: environment})` right after `parseWirelogsPath` (mirroring `exec.go:86`), or add a `RESTNamespaceParam` field to `OperationDef` defaulting to `"namespaceName"` and have `BuildPatternMap` validate it the way it already validates `RESTResourceParam`. Fix the now-false comment at `middleware.go:141-143` either way, and assert `resource.namespace` in the wirelogs test.

### 4. Generated action names are the operator-facing selector contract, and they read badly

**File:** `internal/openchoreo-api/audit/definitions.gen.go`, produced by `tools/auditgen/generator.go:196-217`

`create_clustercomponenttype`, `delete_observabilityalertsnotificationchannel`, `create_authzrolebinding`.

These are what an operator writes in `audit.policies`'s `actions:` selector and what appears in every emitted record. They are inconsistent with the MCP tool names bound to the *same* operations (`create_component_type`) and with P0's hand-written defs.

Not cosmetic, and not cheap to defer: once a deployment has a policy or a downstream index keyed on `create_clustercomponenttype`, changing it is a breaking change. The generator already has `singularOverrides` as the hook for non-mechanical naming — an `actionOverrides` map, or deriving the action from a word-split of the operationId (`CreateClusterComponentType` → `create_cluster_component_type`), settles it now at zero cost.

**This is the item most worth fixing before merge rather than after**, because it is the only one whose cost grows with adoption.

---

## Should fix

### 5. The `findFlusher` workaround treats the symptom; the broken wrapper is `logger.responseWriter`

**Files:** `internal/server/middleware/logger/middleware.go:31-47`, `internal/openchoreo-api/api/handlers/wirelogs.go:41-52`

`logger.responseWriter` implements only `WriteHeader` and `Write` — no `Unwrap() http.ResponseWriter`, no `Flush`, no `Hijack`. It therefore terminates any `http.ResponseController` walk and defeats any `w.(http.Flusher)` assertion.

Verified against the actual dependency (`go.mod: github.com/modelcontextprotocol/go-sdk v1.4.1`): the SDK flushes with a bare assertion at `mcp/event.go:57` and `mcp/streamable.go:1047` — `if f, ok := w.(http.Flusher); ok`. `cmd/openchoreo-api/main.go:229,246` wraps the MCP streamable-HTTP server in `apilogger.LoggerMiddleware`. The assertion fails silently, so MCP SSE frames buffer rather than flushing.

**Pre-existing, not introduced here.** But this commit added `findFlusher` — a hand-rolled reimplementation of `net/http`'s unexported `rwUnwrapper` walk — rather than the four-line `Unwrap()` on the wrapper that actually breaks the chain. The audit wrapper already does it correctly (`audit/middleware.go:101-103`). The codebase now has one wrapper that unwraps, one that doesn't, and a handler-level workaround for the second.

Exec and wirelogs are unaffected today: `main.go:305-315` hands `topMux` straight to `server.New`, so the logger middleware is not in their chain, and gorilla/websocket uses `http.NewResponseController(w).Hijack()` which does follow the audit wrapper's `Unwrap`.

**Fix.** Add `func (rw *responseWriter) Unwrap() http.ResponseWriter { return rw.ResponseWriter }` to the logger wrapper, delete `findFlusher`, and use `http.NewResponseController(w).Flush()` — which needs no capability pre-check. Add a test that a streaming handler behind `logger.Middleware` can flush.

### 6. Unauthenticated events publish by default on every route, and cannot be narrowed

**Files:** `internal/openchoreo-api/config/audit.go:70-77`, `internal/server/middleware/audit/unauthenticated_middleware.go:57-65`, `emitter.go:93-98`, `policy.go:138-149`

`AuditDefaults()` is `Publish: true`, and `NewUnauthenticatedMiddleware` sits in `APIMiddlewares` in front of every generated route including all GETs. Any scanner or expired-token client generates one published record per request, carrying `actor: anonymous/anonymous`, `action: ""`, `category: ""`, and a spoofable `source_ip` as the only distinguishing field.

The plan's B5 note already traced why an operator cannot narrow it: every operation-derived selector short-circuits on `rc.Operation == nil`, leaving `results: [unauthenticated]` as the only lever — which also silences the legitimate 401s the *inner* middleware records against resolved operations.

**Fix — the option the plan considered and dropped.** Give the outer instance a synthetic operation (`ID: "AuthenticationRejected"`, `Action: "authentication_rejected"`, `Category: CategoryAuthorization`) declared in `nonSpecOperationDefs`. It then flows through selector validation, the gate and the coverage matrix like everything else; `actions: [authentication_rejected]` becomes a precise suppression lever; and the record stops shipping two empty required fields. The plan rejected this over the concern that *"an `actions:` selector could then match a value no operation declares"* — declaring it in `nonSpecOperationDefs` is exactly what removes that objection.

Related, and worth stating in `adapter.go:102-133`: the `source_ip` doc comment's defence — *"Actor identity comes from a validated JWT sub, so nothing forensically load-bearing rests on source_ip today"* — stops being true for the events this commit adds. On an auth-rejection record `source_ip` is the **only** attribution field, and it is fully client-chosen. Either update the comment to name the exception, or state the gateway's `xffNumTrustedHops` requirement as a deployment prerequisite in `docs/audit/`.

### 7. MCP has no equivalent of the PR-7 outer instance; an HTTP 401 on `/mcp` emits nothing

**File:** `cmd/openchoreo-api/main.go:246`

```go
mcpHandler := middleware.Chain(mcpLoggerMw, mcpAuth401Mw, jwtMiddleware)(mcpServer)
```

`NewUnauthenticatedMiddleware` is composed only into `APIMiddlewares` (`handler.go:110`). So an expired token produces an `unauthenticated` audit event on REST and **silence** on `/mcp`. `mcpaudit.classifyResult`'s `ErrNoSubject → unauthenticated` covers only the case where a request already got past HTTP auth and the MCP filter finds no subject in the session.

Given how much of this plan rests on surface parity, this is the one asymmetry written down nowhere. If fixed by reusing the middleware, note it hardcodes `OriginAPI` (`unauthenticated_middleware.go:66`), so origin would need to become a parameter.

### 8. `init()` panic in a widely-imported package

**File:** `internal/openchoreo-api/audit/mcp_bindings.go:215-220`

```go
func init() {
	all := append(generatedOperationDefs(), nonSpecOperationDefs...)
	if err := validateEnrichmentKeys(all, mcpEnrichment); err != nil {
		panic(err)
	}
}
```

The condition is worth guarding (a typo'd key silently unbinds a tool), but this package is imported by handlers, config, the coverage gate and both codegen-adjacent tools. Rename an operation in the spec, run `make code.gen`, and `go run ./tools/auditcoverage` panics before it can report what changed — as does every test binary in every importing package, with a stack trace instead of a diagnosis.

The same file already handles the identical class of problem correctly: `MCPBindings()` returns an error, with a doc comment explaining why. Follow that precedent rather than diverging within one file.

### 9. The gate never checks that a bound MCP tool name still exists

**File:** `internal/openchoreo-api/mcphandlers/audit_coverage_test.go`

Assertion 2 walks `perms → bindings`. Assertion 4 checks that exemptions name real tools. Nothing walks `bindings → perms`. A stale `ToolName` in `mcpEnrichment` leaves a binding that can never fire — and `validateEnrichmentKeys`'s `init()` validates only the *operationId* key, not the tool name.

Caught indirectly today: a rename trips assertion 2 on the new name, a removal trips the `len(perms) == 157` pin. Both are inference, and the pins are the part most likely to be "fixed" by bumping the number.

Same root cause makes `tools/auditcoverage/main.go:152-153` arithmetically loose — it computes state-modifying tools as `len(boundToolNames)+len(exempted)`, mixing a count derived from the binding table with one derived from `perms`.

**Fix.** Add the direct assertion (four lines, and it says what it guards, which is the standard the rest of this gate holds itself to).

### 10. ~~`toolsets.Register`'s error is discarded in both the gate and the tool~~ — WITHDRAWN

**This finding was wrong.** `Toolsets.Register` returns no error — its second return value is `toolToToolsets map[string]map[ToolsetType]bool` (`pkg/mcp/tools/register.go:333-336`). `perms, _ := toolsets.Register(server)` discards a map, not an error, and there is nothing to check. See the follow-up pass above.

---

## Minor

- **`IsReadOnly`'s `:view` suffix rule** (`pkg/mcp/tools/types.go:648-660`) returns true for `logs:view` — the exact action wirelogs authorizes against. This commit decided wirelogs is a read worth auditing (`audit/exec_wirelogs.go:22-24`); the MCP-side rule would wave the equivalent tool through with no binding and no exemption. No such tool exists today, so latent — but the two surfaces now disagree on the same question. Consider a denylist of sensitive actions that are never read-only regardless of verb.
- **`pathTail` can index `segs[-1]`** (`tools/auditgen/generator.go:230-243`). After `i--` for an action suffix and `i--` for a path param, nothing bounds `i >= 0`. Unreachable from the current spec, but this is a generator that produces a careful named error for every other malformed input; a bounds check costs two lines.
- **Dead code: `extractResourceArg`** (`pkg/mcp/mcpaudit/audit_middleware.go:202`). Only caller is its own test at `audit_middleware_test.go:395`; production uses `argFromParsed(args, binding.ResourceArg)`. Delete both.
- **`RequestIDRejections()` has no reader** (`adapter.go:67-70`). B8 asked to "count rejections"; the counter exists and is tested, but nothing exports it as a metric or logs it. Compounding: `logger.Middleware` now normalizes the header first, so on every REST and MCP route the audit-path counter can never increment — only exec and wirelogs can, and they have no logger middleware. As shipped it is a two-route counter with no exporter. Wire it to the metrics registry or drop it and let the logger's WARN be the signal.
- **`buildEvent` calls `eventID.String()` before checking `err`** (`emitter.go:74-78`). Harmless — the result is overwritten — but it reads as a bug and will draw a reviewer's eye every time.
- **Two render paths diverge on a nil resource.** `Event.MarshalJSON` emits `"resource":null` (no `omitempty` on `eventJSON.Resource`); `Logger.LogEvent` omits the group entirely. The plan's E.5 guard test exists for exactly this drift and should cover the nil case. (Both paths *do* correctly include the `|| e.Hierarchy != (Hierarchy{})` clause — checked.)
- **exec and wirelogs bypass `logger.Middleware`** (`main.go:305-315`), so they produce no access-log line and their audit `request_id` correlates to nothing. Pre-existing, but P1 is what made those routes emit records that imply a correlation ID.
- **An auth-layer 403 is unaudited (latent).** `NewUnauthenticatedMiddleware` returns without emitting unless `rw.statusCode == 401`; `auth/jwt/middleware.go:175` writes a **403** when `config.SuccessHandler` returns an error and short-circuits, so the inner middleware never runs either. Dead today — `SuccessHandler` is never set anywhere in `cmd/`, `internal/`, or `pkg/` — but B5's rationale ("a 403 means auth succeeded and the PDP refused a known subject, which the inner instance already audits") is only true because that field happens to be nil. Either comment the branch or widen it to `401 || 403`; the emitted-marker already prevents the double-audit that narrowing was meant to avoid.
- **`pattern_map.go:33-41` is last-write-wins on a duplicate operationId.** `for method, op := range item.Operations()` iterates a map, so if two spec entries shared an operationId, which pattern wins is nondeterministic. oapi-codegen rejects such a spec first, and `auditgen` catches it in `buildDefinitions` (`generator.go:155-161`), so defense-in-depth only — but a two-line "already seen" error would match the collision handling three lines below it.
- **`format.Source`'s error is returned unwrapped** (`generator.go:321`), so a template bug surfaces as a bare `expected declaration, found …` with no indication it came from `renderDefinitions`.
- **`callsAuditSetResource` is branch-insensitive** (`audit_setresource_ast_test.go:143`), so a handler calling `SetResource` only on a 500 error path satisfies the gate. The doc comment *is* honest about the mechanism (*"anywhere in its syntax tree, including inside nested blocks, closures, and branches"*); only the implication is left for the reader. Worth one sentence.
- **`TestRender_Deterministic`'s claim is stronger than the test** (`tools/auditcoverage/main_test.go:26-39`). Two `render()` calls in one process will usually agree even with a map-order bug, since Go's per-map seed is stable within a process. `TestCoverageMatrix_IsFresh` is the real gate; downgrade the comment.
- **`NewAuditContext(r.Context(), nil)` at `unauthenticated_middleware.go:64`** discards the returned context purely to obtain an empty `*AuditData`. `&AuditData{}` is clearer and allocates one object instead of two.
- **Unbounded WARN rate** (`logger/middleware.go:69-72`): every request with a malformed `X-Request-ID` writes a WARN. The value is truncated (good), but the line itself isn't rate-limited, so a client can generate one log line per request.
- **`audit-gen` depends on `openapi-codegen`, which is also a direct prerequisite of `code.gen`** (`make/golang.mk:205`, `make/lint.mk:160`). Fine serially; under `make -jN` the two could interleave writes into `api/gen/`. Order-only prerequisites (`|`) would make the intent explicit.

---

## Plan conformance

### P1 — 1.15 was folded into the PR the plan said to keep it out of

§0.7 states: *"Lands: follow-up commit on `feat/audit-logging-p1`, after PR #4460 is reviewed — deliberately **not** folded into it"*, with the rationale *"the schema change should not reopen a 73-file PR already in review."*

The hierarchy work (`Hierarchy` type, both setters, the `services/authz.go` capture, both render paths, the MCP seed) is in this commit. The reasoning that produced that decision still applies — a wire-schema change is a different review than a coverage change. Either split it back out, or delete the "deliberately not folded" line so the record stops contradicting itself.

### P2 — `Closes #4460` while the plan's own exit criteria 3 and 8 are unmet

§0.3 grades criterion 3 (*"each demonstrated by a deliberate breaking change, not merely asserted in code"*) and criterion 8's live verification as ◐, tracked as tasks **1.13** and **1.14**. §0.4 calls 1.14 *"the one open item with a real chance of finding a defect."*

Blocking finding 2 is that defect. `TestFindFlusher_SeesThroughAuditWrapper` covers `Flusher`; nothing exercises a hijacked connection, which is what B7 explicitly said not to take on trust. Hold the `Closes`, or land 1.14 first.

### P3 — Three gaps the plan assigned to the coverage matrix are missing from it

`docs/audit/coverage-matrix.md`'s *Known non-events* has five bullets and mentions none of `stdio`, `filterByAuthz`, `unauthenticated`, or `401` (grepped):

| Gap | Where the plan assigns it |
|---|---|
| `pkg/mcp.NewSTDIO` registers no middleware | §2.6, verbatim: *"Belongs in the coverage matrix's Known non-events"* |
| `?filterByAuthz=false` records a service-layer denial as `failure`, not `denied` | E.10 — the qualifier on exit criterion 4 |
| MCP HTTP-level 401 emits nothing (finding 7 above) | not previously recorded |

The matrix is the DO-2 deliverable. Its freshness gate proves it matches the code, not that it is complete — that distinction is worth a line in the file itself.

### P4 — §0.1's commit citations are stale again

`0ad4c9bd` and `8d257ca9` do not resolve against the squashed `6b54b8c4`. That is the exact failure §0.1 was written to warn about; the section should now cite the PR only, per its own policy.

### Design notes (no change required)

- **`SetHierarchy` replaces the whole struct rather than merging fields.** The "first Check wins → last Check wins" fix is correct and its doc comment earns its length. A field-wise merge (fill empty fields only) would have fixed the original `create_component` bug *and* preserved the MCP argument seed when a downstream `Check` carries a thinner hierarchy. I looked for a live case — `create_workload` (`workload/service_authz.go:46`), `create_resource_release_binding` (`resourcereleasebinding/service_authz.go:42`), `create_project_release_binding` — and in each the service-layer `Check` is at least as rich as the seed, so there is no defect today. Recorded because the doc comment already names the inverted version of this risk (a wrapped service calling another wrapper *after* its own `Check`) and nothing enforces the convention it depends on.
- **Gate assertions 3 and 6 cannot fail, and the mitigation is a magic number.** §0.5 owns this honestly. Note only that `wantTools = 157` and `wantBoundToolNames = 69` will be bumped by hand by whoever adds a tool — i.e. by the person the pin exists to catch. The failure messages are good enough that they will read them, but a comment at each constant saying *"bumping this is a decision, not a chore"* would help. Deriving them as an invariant (`len(perms) == len(bound) + len(exempted) + len(readOnly)`) would be better still.

---

## What's good

1. **The seed-then-override resource model is correct**, and `withHierarchyNamespaceFallback` (`emitter.go:113-126`) returning a *copy* rather than mutating the shared `*Resource` is the right call for the right reason — the comment names the exact aliasing hazard it avoids.
2. **Reading the spec through `gen.GetSwagger()` rather than the raw YAML** (`tools/auditgen/main.go:11-17`) is the non-obvious correct choice, and the reason — oapi-codegen PascalCases operationIds, and that form is what `BuildPatternMap` matches at request time — is documented where the next person will find it.
3. **`emittedMarker`** (`context.go:90-123`) solves outer/inner mutual exclusion properly, and explains why the obvious alternative (re-reading `r.Context()` after `ServeHTTP`) cannot work. `TestUnauthenticatedMiddleware_AuthenticatedRequestEmitsExactlyOnce` drives it through a `fakeAuth` shaped like the real JWT middleware rather than pre-seeding the subject — the difference between a test that catches the bug and one that doesn't.
4. **`resourceCategories` exhaustive in both directions.** `deriveDefinition` errors on a kind with no entry, `checkNoOrphanCategories` errors on an entry with no kind. That is what makes gate assertion 5 meaningful rather than decorative.
5. **`MCPBindings` / `MergeMCPAliases` / `scopeTracker`** turn three silent-overwrite failure modes into construction-time errors, including the non-obvious one (mixing an unscoped and a scoped binding for one tool, which does not collide on the map key but makes the scoped entry unreachable).
6. **Both `SetResource` AST gates.** Machine-enforcing the presence half of a review rule is the right response to 78 call sites.
7. **`TestCoverageMatrix_IsFresh`** closes the gap left by keeping the matrix out of `code.gen` — a generated document that cannot silently go stale.

---

## Recommended order

**Before merge**
1. Finding 1 — audit `HandleAutoBuild`, or write a reason that survives `k8sClient.Create`
2. Finding 2 — exec's always-`success` classification (and ideally the session-start event)
3. Finding 3 — wirelogs namespace parameter
4. Finding 4 — generated action names, while changing them is still free
5. Reconcile P1 and P2 — split 1.15 back out, or update the plan; drop `Closes #4460` until 1.13/1.14 land

**Next PR**
Findings 5–8, plus the residual hole in 9 (move the `schemas[key.ToolName]` lookup above the `ResourceArg == ""` check), plus P3's three matrix entries. Finding 10 is withdrawn.

**Backlog**
The Minor list, and the two design notes.
