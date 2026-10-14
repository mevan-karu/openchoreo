# Migrate `observer-api` onto its generated OpenAPI server

**Companion to:** [`openchoreo-audit-logging-phased-plan.md`](./openchoreo-audit-logging-phased-plan.md) §2.6, which first recorded the divergence risk · [`openchoreo-audit-logging-observer-plan.md`](./openchoreo-audit-logging-observer-plan.md) — the audit phase that runs **after** this one, on top of what this plan builds

**Status:** ✅ **Decided — Phase 0 complete, Phase 1 spiked** · **Date:** 2026-09-01 · **Phase:** first; a prerequisite for observer auditing · **Baseline:** `c841b75c`

> **Follow-on, 2026-09-02: the audit phase has landed on top of this.** Two things this plan left in place were
> changed by it, and are noted here so a reader of §5/§7 is not surprised:
>
> - **`ObserverMiddlewares` and `InternalMiddlewares` gained audit**, and `InternalMiddlewares` now returns
>   `([]internalgen.MiddlewareFunc, error)` rather than a bare slice — both composers reject a nil audit emitter at
>   startup. The orderings this plan pinned are otherwise unchanged; audit was inserted inside auth and
>   unauthenticated-audit outside it.
> - **`observermiddleware.Logger` is deleted**, replaced in both composers by the shared `apilogger.Middleware`. That
>   moves the access log from `Debug("HTTP request", …)` to `Info("ACCESS-LOG", …)` and adds `X-Request-ID`
>   validation. `TestInternalMiddlewareOrdering`'s discriminating assertion (Phase 1, item 7) now keys on
>   `"ACCESS-LOG"`; the reversal check that made it discriminating was re-run and still fails on reversal.
>
> Exit criterion 6 ("exactly one definition per server, in a composer function") is what made both edits a
> single-site change. See [`openchoreo-audit-logging-observer-plan.md`](./openchoreo-audit-logging-observer-plan.md).

**Decisions (Phase 0, settled by the maintainer):** §1 → **option B** (make the spec honest) · §2 → **option 1** (split the spec). Rationale and the mechanism that makes B cheap are in §1 and §2 below.

**Problem.** `internal/observer/api/gen/server.gen.go` is generated — `ServerInterface`, `StrictServerInterface`, `HandlerWithOptions`, `NewStrictHandler`, the embedded spec, typed request/response objects, per-operation parameter binding — and **none of it serves traffic**. Observer registers its 18 routes by hand on two `http.ServeMux` instances. So `openapi/observer-api.yaml` is documentation that nothing enforces, and the generated code is dead weight that `make code.gen` keeps regenerating.

---

## §0 — Scope: auditing is explicitly out

**This plan does not mention audit again after this section, and must not be designed around it.**

Observer auditing is a separate, later phase with its own document. The order is deliberate and was chosen by the maintainer: **migrate to the generated server cleanly first, then build auditing on top of the migrated architecture.** Auditing on a hand-registered mux needs machinery — a hand-maintained route table, a test proving registered patterns agree with the spec — that exists only to compensate for the absence of generated routing. Building that machinery and then deleting it is waste; building audit once, on generated routing, is not.

> **§0's original premise is now stale — recorded rather than deleted, because the ordering it justifies is still right.**
>
> This document originally described a staged, uncommitted observer-audit implementation to set aside, and baselined on `92a7e1ba`. Verified against the tree on 2026-09-01: **that staged work no longer exists**, the working tree is clean apart from untracked planning docs, and `92a7e1ba` is **not an ancestor of `HEAD`**. Meanwhile openchoreo-api's audit has **landed** — `internal/openchoreo-api/api/handlers/handler.go:88` (`APIMiddlewares`) composes a live `audit.Emitter` and `apiaudit.GetOperations()`. So the two "architecture-neutral pieces that survive" (`tools/internal/auditgen`, `internal/auditconfig`) are no longer pending prerequisites either. **Baseline is `c841b75c`; nothing needs setting aside.**

---

## §1 — The auth decision: **option B**, and why the split makes it clean

### The divergence, as it stands today

`openapi/observer-api.yaml` declares a global security requirement (line 30) and overrides it to public on **exactly two** operations — `health` (line 41) and `handleAlertWebhook` (line 733), both carrying `security: []`.

It does **not** override it on the four alert-rule CRUD operations. So the spec says `createAlertRule`, `getAlertRule`, `updateAlertRule` and `deleteAlertRule` require a Bearer token, and the generated wrapper agrees: `grep -c 'BearerAuthScopes' internal/observer/api/gen/server.gen.go` → **16**; the two that don't set it are `Health` and `HandleAlertWebhook`.

**In production those four are served with no authentication whatsoever**, on internal port 8081, behind only logger and recovery middleware.

### Why the spec's security scheme is inert today — and why the migration changes that

Worth stating plainly, because it is the crux and it is easy to get backwards.

**Today the spec's `security:` blocks do nothing at all.** Verified:

```
grep -rn 'BearerAuthScopes|OpenAPIAuth' internal/observer/ cmd/observer/   (excluding gen/)
→ only internal/observer/api/finopsadapterclientgen/client.gen.go  (an unrelated adapter client)
```

Authentication comes entirely from where `cmd/observer/main.go` applies `jwtAuth` by hand — `api := routes.With(jwtAuth)` at line 274 for the 12 protected public routes, and **nothing** on `internalRoutes` (line 332: logger + recovery only).

**The migration is precisely what makes the spec load-bearing.** Once routes come from `gen.HandlerWithOptions`, the generated wrapper writes `BearerAuthScopes` into the request context for every operation the spec says needs auth, and `auth.OpenAPIAuth` decides per-request by reading exactly that key:

```go
if r.Context().Value(scopesContextKey) == nil {
    next.ServeHTTP(w, r)   // public
    return
}
protected.ServeHTTP(w, r)  // requires auth
```

That is the point of the migration, not a side effect. But it means that at the moment the spec starts mattering, it had better say the truth.

### The blast radius, named

The caller of the four alert-rule operations is **the control plane's own alert-rule controller**:

`internal/controller/observabilityalertrule/controller.go`
- `defaultObserverInternalBaseURL = "http://observer-internal.openchoreo-observability-plane:8081"` (line 31)
- `getAlertRule` (GET, line 207), `callWriteEndpoint` (POST/PUT, line 269), DELETE on finalizer cleanup (line 437)

It builds requests with `http.NewRequestWithContext(...)` and sets **no `Authorization` header** — verified by reading all three call sites. It has no token source: the reconciler is not acting for a user, and there is no service-account token minting anywhere in that package.

Helm confirms the topology: `observer-internal-service.yaml` exposes port 8081 as a `ClusterIP` service and `observer-config.yaml` sets `INTERNAL_PORT`. It is cluster-internal and **not** routed through the gateway — which is the current, implicit security control.

### Decision: option B — and the split makes it honest rather than embarrassing

The original §1 costed option B as "writes the hole into the published contract", because it assumed one spec needing four apologetic `security: []` lines. **Under §2 option 1 that cost largely disappears:**

- `openapi/observer-api.yaml` keeps the 13 public operations, its global `security: [BearerAuth]`, and `security: []` on `health` **only**. No `security: []` on any state-modifying route.
- `openapi/observer-internal-api.yaml` declares **no `security` block at all**. That is not a lie or an override — it accurately describes an unauthenticated `ClusterIP` service on 8081 that is not gateway-routed.

So the spec becomes load-bearing **and** true in the same change, with **zero behaviour change** and the alert-rule controller untouched.

**Verified, not assumed:** the internal spec generates **zero** `BearerAuthScopes` references (`grep -c BearerAuthScopes internal/observer/api/internalgen/server.gen.go` → `0`). Nothing sets the scopes key, so the internal router needs no `OpenAPIAuth` wrapper and cannot accidentally acquire auth. Auth-neutrality is a property of the generated code, not of wiring discipline.

**Closing the hole later** (phased-plan task 7.22) becomes a single-purpose change: add a security scheme to `observer-internal-api.yaml`, wrap the internal router in `OpenAPIAuth`, and add the controller's token plumbing — all in one reviewable diff, rather than a security-posture change smuggled inside a routing refactor.

> **The decision is recorded in the code, not just here.** `internal/observer/api/spec_topology_test.go` contains `TestInternalSpecDeclaresNoSecurity`, which fails if anyone adds a security scheme to the internal spec, with a message pointing at the controller work that must land alongside it. The spec file itself carries a matching comment block. That is what stops a future reader from "fixing" the hole and silently breaking alert-rule reconciliation.

---

## §2 — Decision: option 1 (split the spec), with shared schemas single-sourced

`HandlerWithOptions` registers **all 18 routes** on whichever mux it is handed. Observer needs them split: 13 on the public server (port 9097), 5 on the internal server (port 8081). Calling `HandlerWithOptions` twice with two `BaseRouter`s puts all 18 on **both** — the public port would expose alert-rule CRUD, and the internal port every query. That is a security regression, not a migration.

Compounding it: `StrictServerInterface` is **one** interface covering all 18 operations, while observer's handlers are **two** types — `*Handler` (13 methods) and `*InternalHandler` (5). Neither satisfies the interface alone.

Note that keeping one spec does **not** dodge this: one spec across two ports forces a hand-maintained port→operation filter (the old option 2), which is a fresh instance of the very drift class this migration removes. The two-port deployment topology is what makes the split the honest representation.

### The mechanism that makes the split cheap — verified with a working spike

The original plan assumed shared schemas must "move to a common component file `$ref`'d by both, or be duplicated with a generation-time check that they agree". **Neither is necessary.** oapi-codegen's `import-mapping` points the internal spec's external `$ref`s at the **existing** `gen` package:

```yaml
# internal/observer/api/cfg-internal-server.yaml
package: internalgen
output: internal/observer/api/internalgen/server.gen.go
generate:
  std-http-server: true
  strict-server: true
  embedded-spec: true
output-options:
  skip-prune: true
import-mapping:
  ./observer-api.yaml: github.com/openchoreo/openchoreo/internal/observer/api/gen
```

`observer-api.yaml` keeps **all** schemas in `components` (it already sets `skip-prune: true`, so unused ones are still generated) and sheds only the 5 internal **paths**. The internal spec `$ref`s them across:

```yaml
$ref: "./observer-api.yaml#/components/schemas/AlertRuleRequest"
```

**Spike results — all verified against the tree, not reasoned about:**

| Check | Result |
|---|---|
| `internalgen` imports the existing package | `externalRef0 "github.com/.../internal/observer/api/gen"` |
| Schema structs duplicated | **None.** `internalgen/models.gen.go` is 18 lines: three `*JSONRequestBody` aliases to `externalRef0` types |
| `gen/models.gen.go` changes | At the time of the spike: only the 3 `*JSONRequestBody` aliases removed (−9 lines). **Superseded in Phase 2** — see the note below |
| Service layer / mocks touched | **None** — the ~15 files using `gen.AlertRuleRequest`, `gen.AlertWebhookResponse` etc. are untouched |
| Operations in `internalgen` | Exactly 5, on exactly the 3 expected paths |
| `BearerAuthScopes` in `internalgen` | **0** (see §1) |
| `internalgen.GetSwagger()` at runtime | **Works.** Generated `PathToRawSpec` composes `gen.PathToRawSpec`, so external refs resolve. Confirmed by a passing test, not by inspection |
| `go build ./internal/observer/...` | Passes |

Two gotchas the spike surfaced, both now handled:

1. **The `import-mapping` key must match the `$ref` string literally** — `./observer-api.yaml`, with the `./`. Keyed as `observer-api.yaml` it fails with `unrecognized external reference './observer-api.yaml'`.
2. **`*JSONRequestBody` aliases come from the *models* generator, not the server generator.** A config generating only `std-http-server` + `strict-server` compile-errors with `undefined: CreateAlertRuleJSONRequestBody`. The fix is `models: true` in the same config — **not** a second config file. (An earlier revision of this document claimed two files were required; that was wrong, and the consolidation is done.)

**Why a separate config at all, rather than passing the second spec to `cfg-server.yaml`:** the two specs must land in two Go *packages*. Both generate `ServerInterface`, `StrictServerInterface`, `HandlerWithOptions`, `NewStrictHandler`, `StdHTTPServerOptions` and `GetSwagger` — one package would be a pile of duplicate declarations. Two packages, two configs.

**Why `models: true` lives in that same config, and why that is the *consistent* choice.** The repo's rule is not "always split models" — it is that models must be generated exactly once per Go package. Sorting every generated package by artifact count makes the pattern plain:

| Package | Artifacts | `models` |
|---|---|---|
| `openchoreo-api/api/gen` | 3 — models, server, client | split (`cfg-models.yaml`; `models: false` on server + client) |
| `observer/api/gen` | 3 — models, server, client | split, same |
| `observer/api/logsadapterclientgen` | 1 — client | **bundled** (`models: true` in the client config) |
| `observer/api/finopsadapterclientgen` | 1 — client | **bundled** |
| `observer/api/internalgen` | 1 — server | **bundled** ← follows the single-artifact precedent |

Splitting `internalgen` would make it the only single-artifact package in the repo carrying a separate models config. The trigger to revisit is a *second* artifact — most plausibly a generated client so the alert-rule controller can stop hand-rolling `http.NewRequestWithContext` — at which point it joins the 3-artifact column and models split out.

> **Superseded: the spec topology changed after Phase 2.** The spike above kept *all* schemas in `observer-api.yaml` and had the internal spec `$ref` six of them across. That left five schemas in the public spec referenced by no public operation — visible in the published contract, and a trap for anyone who deleted the "unused" ones.
>
> The five alert-rule/webhook schemas now live in `observer-internal-api.yaml`, beside the only operations that use them. `ErrorResponse` is the single remaining external `$ref`. Consequences:
>
> - `internalgen` defines those five itself; `gen` no longer does. 13 Go files moved `gen.X` → `internalgen.X` and the mocks regenerated. Several files now import `internalgen` *instead of* `gen`, which is a decent signal the split landed on the right seam.
> - **Removing them renamed an unrelated enum.** oapi-codegen prefixes enum constants with the type name only on collision; `AlertWebhookResponseStatus` also had an `"error"` value, so once it moved out `SpanStatusCodeError` became a bare `Error`. Accepted rather than papered over: `gen` already had bare exported constants (`Created`, `Deleted`, `Synced`, `Failed`), and there was exactly one usage to update. The alternatives were rejected — `always-prefix-enum-values` on `gen` renames ~40 constants including `gen.BadRequest`, and `x-enum-varnames` does not force the prefix. `internalgen` *does* set `always-prefix-enum-values`, where it costs nothing and fixed a worse case (bare `Success`/`Error`).
> - The remaining divergence is deliberate: `gen` prefixes on collision, `internalgen` always. Revisit together if the churn on `gen` ever becomes worth paying.

**Cost that remains:** `observer-api.yaml` is a published contract and the split is externally visible, so it needs a note. That cost is smaller than originally stated — observer's generated `client.gen.go` has **no consumers** (the e2e suites import *openchoreo-api's* `gen`; `occ` hand-rolls `internal/occ/resources/client/obs_client.go`), and the 5 moved operations were never reachable from outside the cluster.

**Rejected:** collapsing to a single port (forces option A, moves the `ClusterIP` service, the controller's base URL and the Helm templates — worth considering on its own merits later, not here), and partial migration (leaves unmigrated exactly the operations where spec and reality diverge).

---

## §3 — Verified current state

Every claim below was re-checked against `c841b75c` on 2026-09-01. The original plan's technical claims all held; only its metadata was stale (see §0).

**Generated, and unused.** `internal/observer/api/cfg-server.yaml` sets `std-http-server: true`, `strict-server: true`, `embedded-spec: true`, `skip-prune: true`. `server.gen.go` (2,604 lines) provides `ServerInterface` (18 methods), `StrictServerInterface`, `ServerInterfaceWrapper`, `Handler`/`HandlerFromMux`/`HandlerWithOptions`, `NewStrictHandler`/`NewStrictHandlerWithOptions`, `BearerAuthScopes`, and per-operation `*RequestObject`/`*ResponseObject` types with `Visit*Response` writers. **Only `GetSwagger()` and the model types are consumed today.**

**Routes: 18, hand-registered inline in `cmd/observer/main.go`** — 13 on the public mux (1 unauthenticated: `GET /health`; 12 behind `jwtAuth`), 5 on the internal mux (none authenticated). They currently agree with the spec, so there is **no drift to repair today**; the migration prevents future drift rather than fixing present drift.

**One spec, two ports.** Of the spec's 16 path items / 18 operations, 3 path items / 5 operations are served *only* on port 8081 and are not reachable from outside the cluster.

**Handler shape.** Method names already match the generated operationIds exactly (`QueryLogs`, `UpdateIncident`, `CreateAlertRule`, …) — a genuine head start. But every one has the raw `func(w http.ResponseWriter, r *http.Request)` signature and reads parameters itself:
- `r.PathValue` at 16 sites across `alerts.go`, `alerts_query.go`, `finops.go`, `traces.go`
- `r.URL.Query()` in `finops.go` (both operations)
- responses written via `baseHandler.writeJSON` / `writeErrorResponse`, not `Visit*Response`

`ServerInterface` wants typed extras (`CreateAlertRule(w, r, sourceType string)`); `StrictServerInterface` wants `(ctx, RequestObject) (ResponseObject, error)`. **Neither matches today's signatures**, so this is a rewrite of all 18, not a re-registration.

**Test blast radius.** 4,740 lines across 13 handler test files — `validations_test.go` (1,210), `alerts_internal_test.go` (626), `metrics_test.go` (405), `alerts_query_test.go` (405), `traces_handler_test.go` (389), `coverage_gaps_test.go` (376), `traces_test.go` (319), then 7 smaller. These drive handlers through `httptest` against the `(w, r)` signature.

**Not in the spec, and staying hand-registered.** `GET /.well-known/oauth-protected-resource` and `/mcp` (the latter with its own chain: `mcpMiddleware → jwtAuth`). openchoreo-api already solves this with `baseMux` (non-spec routes registered first, generated routes layered on) plus a `topMux` for anything needing to sit outside the generated handler — reuse that pattern.

**Generated operationIds are PascalCase in the embedded spec.** `gen.GetSwagger()` returns `CreateAlertRule`, not the YAML's `createAlertRule` — oapi-codegen normalizes to Go names when embedding. Any test asserting on operationIds must expect that.

---

## §4 — Behaviour changes that are not about auth

Three, all client-visible. None is a reason not to migrate; all belong in the PR description.

**1. The error-shape problem exists at *two* layers, not one.**

*Layer one — parameter binding.* `GetComponentCostsParams` declares `StartTime` and `EndTime` as non-pointer required fields. Today `finops.go` reads them with `query.Get(...)` — absent becomes `""` — and `ValidateCostQueryRequest` produces a `400` carrying the `gen.ErrorResponse` JSON shape. Under the generated wrapper a missing `startTime` is caught before the handler runs, producing a `RequiredParamError` routed to `StdHTTPServerOptions.ErrorHandlerFunc`, whose default is `http.Error(w, err.Error(), 400)` — **plain text**. (See the nil-check on `ErrorHandlerFunc` inside `HandlerWithOptions` in `server.gen.go`; cited by symbol because line numbers shift on every regeneration.)

*Layer two — the strict handler.* `NewStrictHandler(h, nil)` installs plain-text defaults too (see `NewStrictHandler` in `server.gen.go`):

```go
RequestErrorHandlerFunc:  func(...) { http.Error(w, err.Error(), http.StatusBadRequest) }
ResponseErrorHandlerFunc: func(...) { http.Error(w, err.Error(), http.StatusInternalServerError) }
```

`RequestErrorHandlerFunc` fires on a **malformed JSON body** — which today returns the `gen.ErrorResponse` shape via `writeErrorResponse` (e.g. `alerts.go:26-29`, `INVALID_REQUEST_BODY`).

> **Both layers need explicit handlers rendering `gen.ErrorResponse`**: `StdHTTPServerOptions.ErrorHandlerFunc` *and* `NewStrictHandlerWithOptions` with both strict funcs. Fixing only the first leaves exit criterion 5 half-satisfied, and no existing test asserts the *shape* of these 400s, so it fails silently.

**2. Non-spec routes would silently lose their middleware.** Today `/health`, `/.well-known/oauth-protected-resource` and `/mcp` all get logger + recovery from `middleware.NewRouteBuilder(mux).With(loggerMiddleware, recoveryMiddleware)` (`main.go:258`). Once the generated routes carry their own chain via `HandlerWithOptions`, anything registered directly on the base mux gets **nothing** unless wrapped explicitly. Losing recovery on `/mcp` turns a handler panic into a dropped connection. openchoreo-api wraps deliberately — `middleware.Chain(mcpLoggerMw, mcpAuth401Mw, jwtMiddleware)(mcpServer)` at `cmd/openchoreo-api/main.go:247`. Mirror that; don't let it fall out. (`/health` moves *into* the generated router, so it inherits the composed chain — only the two non-spec routes need wrapping.)

**3. Unknown-route behaviour per port.** Today each mux carries only its own routes, so a query sent to port 8081 gets that mux's 404. Under the split that stays true — and is now guaranteed by generation rather than by discipline. Worth one assertion per port.

**4. Validation order inverts when both the path and the body are invalid.** The strict handler decodes the body *before* the handler runs, so `POST /rules/unknown` with a malformed body now returns `INVALID_REQUEST_BODY` where it previously returned `INVALID_SOURCE_TYPE` — the handler used to check `sourceType` first because it decoded the body itself. Only the both-invalid case changes; either one alone is unaffected. This is inherent to the strict layer, not a wiring choice.

Alongside it, the malformed-body **message text** changed from `"invalid request body: …"` to `"can't decode JSON body: …"`, since the strict layer produces the error now. Same status, same `errorCode`, different prose.

Both are safe for the one real caller: the alert-rule controller branches purely on status code, and the only body field it reads on error is `message`, used as log text. Nothing matches on `errorCode` or message strings.

**6. Parameter-binding 400s bypass authentication — and the logger.** This is the one place exit criterion 3's "authentication behaviour is unchanged" needs qualifying.

`ServerInterfaceWrapper` calls `ErrorHandlerFunc` and returns *before* the `for _, middleware := range siw.HandlerMiddlewares` loop, so a request rejected during parameter binding never reaches auth, recovery or the access log. Verified against the real chain:

```
GET /costs/namespaces/ns/environments/dev                     (no token) → 400 INVALID_PARAMETER
GET /costs/namespaces/ns/environments/dev?startTime=…&endTime=…  (no token) → 401
```

**Scope is exactly the two FinOps operations** — every other path parameter binds into a plain `string` and cannot fail. What an unauthenticated caller learns is only that a required parameter is missing, which the spec already publishes. The material consequence is that such a caller can drive a `logger.Info` line per request.

**Not fixed, deliberately.** oapi-codegen offers no hook between routing and `ErrorHandlerFunc`, so re-checking auth inside `ParamBindingErrorHandler` would hand-roll exactly the per-route auth the migration removed. The `logger.Info` in that handler stays at Info precisely *because* the access log never fires for these requests — it is the only record they leave.

**5. `errNilServiceResponse` returns 500 where the old code returned 200.** A service returning `(nil, nil)` violates the `AlertRuleService` contract and cannot happen while it holds; previously `writeJSON` would have emitted a literal `null` body with 200. Unreachable, but covered by `TestInternalHandlers_NilServiceResponse` so the five guard branches aren't untested.

---

## §5 — Phases

Sequenced **internal-side-first**: the 5 internal operations are a complete vertical slice — spec split, second gen package, strict handlers, middleware composer, rewired server, tests through the generated router. Every mechanism the public side needs gets proven on the smaller, lower-risk half (one 626-line test file, no auth question) before the 13 public operations and ~4,100 lines of public handler tests. Same end state as the original Phase 1→5; ordered to fail cheap.

### ✅ Phase 0 — Decide and record *(complete)*

§1 → **B**, §2 → **1**, both recorded above with reasoning. No controller prerequisite, because B is auth-neutral.

### ✅ Phase 1 — Internal slice *(complete)*

1. **`openapi/observer-internal-api.yaml`** — 5 operations, no `security` block, external `$ref`s into the public spec's `components`, and a comment block explaining why there is no security scheme.
2. **`cfg-internal-server.yaml`** — one config, with the literal-key `import-mapping`.
3. **`internal/observer/api/internalgen/`** — a single `server.gen.go`. No schema is duplicated: only three `<Operation>JSONRequestBody` aliases pointing back at `gen`.
4. **The 5 internal paths removed from `openapi/observer-api.yaml`**, every schema kept in `components`. Verified afterwards that `skip-prune: true` still emits all six shared types (`AlertRuleRequest`, `AlertRuleResponse`, `AlertingRuleSyncResponse`, `AlertWebhookRequest`, `AlertWebhookResponse`, `ErrorResponse`) even though no public operation references them — this was the one assumption that could have collapsed the whole approach. `models.gen.go`'s only change is the 3 request-body aliases moving to `internalgen`. Split note added to the public spec's `info.description`.
5. **`make code.gen` + `make/lint.mk` wiring.** Generation confirmed idempotent.
6. **`*InternalHandler`'s 5 methods converted to `internalgen.StrictServerInterface`**, with `var _ internalgen.StrictServerInterface = (*InternalHandler)(nil)`. All `r.PathValue` reads gone.
7. **`InternalMiddlewares`** in `internal/observer/api/handlers/wiring.go`, plus all three explicit error handlers from §4.1, and `wiring_test.go` pinning the ordering.

   The ordering is worth the test because it is **inverted** relative to the rest of the codebase: `middleware.Chain` treats the *first* entry as outermost (hence the old `.With(logger, recovery)`), while oapi-codegen wraps its slice in order so the *last* entry ends up outermost — so `InternalMiddlewares` returns `{recovery, logger}` to get `logger → recovery → handler`.

   Getting a genuinely discriminating assertion took two attempts. Asserting "the logger saw the request path" passes in **both** orderings, because `Recovery` logs the path itself. The signal that actually separates them is the Logger middleware's own `"HTTP request"` line: Logger emits it *after* calling `next`, so it survives a panicking handler only when Logger is outside Recovery. Confirmed by reversing the slice and watching both tests fail. Reversed, a panicking request still returns 500 but disappears from the access log entirely — silent, and now caught.
8. **Internal server rewired** in `cmd/observer/main.go` — no hand-written pattern string remains for any internal route.
9. **`alerts_internal_test.go` reshaped** to drive the generated router end-to-end, plus the two internal-handler tests in `coverage_gaps_test.go`.

**Verified:** `go build ./...`, `go vet`, `make lint` (0 issues), the full `./internal/observer/...` suite, codegen idempotency, and that the alert-rule controller's four URLs still match the generated routes exactly with no auth required.

**G.6 is only half-exercised on this port — and that matters for Phase 2.**

The strict layer's `RequestErrorHandlerFunc` is genuinely live and tested: a malformed body returns the `gen.ErrorResponse` shape, asserted down to `Content-Type` and all three fields by `assertErrorResponseShape`.

`ParamBindingErrorHandler` is wired but **unreachable here**. Both internal path parameters bind into a plain `string`, and the spec's `sourceType` enum generates no runtime check, so `BindStyledParameterWithOptions` cannot fail — the handler has no caller and no test. It is forward-wiring for FinOps, whose `startTime`/`endTime` are required non-pointer query params and *will* reach it. Exit criterion 5 is therefore half-satisfied until Phase 2.

> **Phase 2 hazard, verified in the generated code:** `ErrorHandlerFunc` fires *before* the middleware loop — see `ServerInterfaceWrapper.DeleteAlertRule` in `internalgen/server.gen.go`, where the binding error returns at the `siw.ErrorHandlerFunc(...)` call several lines above `for _, middleware := range siw.HandlerMiddlewares`. So parameter-binding 400s bypass both logger and recovery. When FinOps makes them reachable, they will be silently absent from the access log. Fixing it means logging inside `ParamBindingErrorHandler` itself rather than relying on the chain — it already logs at debug level, which should be raised to match whatever the access log expects.

**Two findings worth carrying into the PR description:**

- **The spec was missing a 404 on `updateAlertRule`.** The handler has always returned it on `ErrAlertRuleNotFound`, but the pre-split spec declared only 200/400/500, so the strict interface had no `UpdateAlertRule404JSONResponse` to return. Added to the internal spec, which is exactly the kind of spec/implementation gap a generated server surfaces and a hand-registered mux hides.
- **Empty `ruleName` now returns 404, not 400.** `{ruleName}` matches only a non-empty path segment, so `/rules/` never reaches the handler. Previously the handler validated it and returned `400 INVALID_RULE_NAME`. 404 is the better answer and the handler keeps its defensive check; recorded in `TestGetAlertRule_EmptyRuleName`.

### ✅ Phase 2 — Public slice *(complete)*

**The plan mis-sized this phase, and the correction is the main lesson.** It said Phase 2 "carries the least risk — the compiler checks all 13 against the interface, so the failure mode is a build error." That is true only for signature shape. It is wrong about the request and response *types*, which the compiler cannot check for semantic equivalence:

| | Operations | What was actually needed |
|---|---|---|
| Bodies already `gen.*` | 6 — traces ×3, alerts/incidents ×3 | signature change only |
| Bodies `types.*`, differing from `gen.*` | 4 — logs, events, metrics, runtime-topology | a `gen` → `types` adapter each |
| Params only | 2 — FinOps ×2 | field-mapped adapter each |
| No body | 1 — health | signature change only |
| **Responses** | **all 13** | services return `types.*`, and `QueryMetrics`/`GetComponentCosts`/`GetRecommendations` return bare `any` — none matches a generated response object |

**Responses: one passthrough type, not the generated typed responses.** `apiResponse` implements all thirteen `Visit*Response` methods and writes its value with the same `httputil.WriteJSON` as before, so every response is byte-identical to pre-migration. The generated typed responses were not viable: three services return `any`, `gen.MetricsQueryResponse` is a `oneOf` union, and converting the rest would mean a schema round-trip per response that could silently drop a field. This generalizes what `traces.go` already did — bind the generated request, convert to the internal type, write the service's own response — and it is consistent with §7, whose criteria cover routing, parameters and auth rather than response schemas.

**Requests: marshal round-trip for the four `types.*` bodies.** Both sides describe the same spec schema, so `remarshalJSON` is more faithful than a field mapper — and it is the only approach that preserves `types.SearchScope.UnmarshalJSON`, including its rejection of a `searchScope` mixing `workflowRunName` with `project`/`component`/`environment`. The generated `oneOf` wrapper keeps the scope as raw JSON and performs no such check; re-decoding runs the hand-written discriminator unchanged. FinOps is field-mapped instead, because `types.CostQueryRequest` has no JSON tags.

**The zero-time trap, which would have shipped silently.** The generated time fields have no `omitempty`, so an absent `startTime` decodes to the zero time; formatting that naively gives `"0001-01-01T00:00:00Z"`, which parses fine. `ValidateTimeRange`'s `"startTime is required"` would never fire and callers would get `"query time range cannot exceed 30 days"` instead. `rfc3339OrEmpty` maps zero to `""`. Verified by deleting the guard and watching all four cases fail with exactly that wrong message — the status stays 400 either way, so only asserting the message catches it. `RFC3339Nano` rather than `RFC3339`, because the latter truncates sub-second precision the old `query.Get()` path passed through verbatim.

**Also done:** `ObserverMiddlewares` (logger → recovery → auth → handler, mirroring the pre-migration `Chain(logger, recovery, jwtAuth)`), the public rewire with `/mcp` and `/.well-known/...` explicitly wrapped per §4.2, and ~4,100 lines of handler tests moved onto the generated router via a `serve` helper. `baseHandler.writeJSON` / `writeErrorResponse` are gone — handlers no longer touch a `ResponseWriter`.

**New tests worth naming:** `TestAbsentStartTimeKeepsRequiredMessage` (4 cases), `TestSearchScopeMixedOneOfStillRejected`, `TestRFC3339OrEmpty`, `TestObserverMiddlewaresLeaveHealthPublic` (drives the real chain incl. `OpenAPIAuth`; asserts `/health` → 200 with no token and a query → 401), `TestObserverMiddlewaresRequireAuthMiddleware`, `TestPublicRouterDoesNotServeInternalOperations`, and `TestGetComponentCosts_MissingTimes` / `_UnparseableTime` — the latter two being the first tests to reach `ParamBindingErrorHandler`, closing the half of exit criterion 5 that Phase 1 left forward-wired.

**Two further behavior changes for the PR description**, beyond §4's list: a malformed body now yields `INVALID_REQUEST_BODY` from the strict layer rather than each handler's own message, and FinOps `startTime`/`endTime` are rejected during binding (`INVALID_PARAMETER`) rather than by `ValidateCostQueryRequest`. Same statuses; different messages.

**Content-Type validation, restored rather than dropped.** Seven public operations decoded via `httputil.BindJSON`, which refused a non-JSON `Content-Type`; the strict layer decodes unconditionally, so a form-encoded POST that used to be a 400 briefly became a 200. `RequireJSONContentType` restores it as the innermost entry in `ObserverMiddlewares` — inside auth, so an unauthenticated caller cannot probe it. It is applied to all ten body-bearing operations rather than the original seven: the other three used `json.NewDecoder` and never checked, but all ten declare `content: application/json`, and "seven of ten validate Content-Type" is the sort of inconsistency generated routing exists to remove. An absent `Content-Type` is still allowed, matching `BindJSON`. `BindJSON` and `GetPathParam` are now deleted — both dead once the strict layer decodes bodies and binds path parameters.

### Phase 2 — original plan text

7. **Convert `*Handler`'s 13 methods to `gen.StrictServerInterface`.** The bulk of the work and the least risk — the compiler checks all 13 against the interface, so the failure mode is a build error. Delete the `r.PathValue` / `r.URL.Query()` reads; replace `writeJSON`/`writeErrorResponse` returns with `Visit*Response`. Keep `baseHandler`'s helpers for anything still writing directly.
8. **`ObserverMiddlewares`** composer returning `[]gen.MiddlewareFunc`, mirroring openchoreo-api's `APIMiddlewares`: logger → `auth.OpenAPIAuth(jwtAuth, gen.BearerAuthScopes)` → handler. This is what makes `/health` public and the other 12 protected *from the spec* rather than from hand-picked route groups.
9. **Rewire the public server** — non-spec routes (`/mcp`, `/.well-known/...`) on the base mux first **with explicit middleware wrapping** (§4.2), generated routes layered on, CORS still wrapping the whole mux.
10. **Reshape the remaining ~4,100 test lines**, driving the generated router so routing, parameter binding and handler run together — strictly more coverage than today. `validations_test.go` (1,210) largely tests validators rather than handlers and should need little change.

> **One middleware-composition function per server**, justified on its own terms with no reference to audit: `main.go` supplies dependencies while owning no ordering, a wiring test can drive exactly the chain production runs instead of reimplementing it, and any future cross-cutting middleware gets a single documented insertion point rather than a second place ordering can drift.

### Phase 3 — Verification

`go build ./...`, `go vet ./...`, `make lint`, `make code.gen && make code.gen-check`, the full `./internal/observer/...` suite. Plus a live `k3d` pass: a JWT-protected query, a `/health` call, **an alert-rule create driven by the real controller (must still work unauthenticated)**, a FinOps request with and without `startTime` (asserting the JSON error *shape*), a malformed JSON body (same), a `/mcp` tool call, and a wrong-port request per port.

---

## §6 — Task breakdown

| # | Task | Dep | State |
|---|---|---|---|
| G.0 | Decide §1's auth option and §2's spec topology; record both | — | ✅ B + 1 |
| G.0a | Re-baseline on `c841b75c`; confirm no staged audit work to set aside | — | ✅ |
| G.1 | ~~Control-plane token source~~ — not needed under B | G.0 | ✅ n/a |
| G.2 | `observer-internal-api.yaml`; `cfg-internal-server.yaml`; `internalgen` package | G.0 | ✅ |
| G.2a | Prove `import-mapping` reuses `gen` + `GetSwagger()` resolves | G.2 | ✅ |
| G.3 | Remove the 5 paths from `observer-api.yaml`; split note | G.2 | ✅ |
| G.3a | Move the 5 alert schemas into the internal spec; `ErrorResponse` the only shared `$ref` | G.13 | ✅ |
| G.4 | `make code.gen` + `make/lint.mk` wiring for `internalgen` | G.2 | ✅ |
| G.5 | Convert `*InternalHandler`'s 5 methods to strict | G.3 | ✅ |
| G.6 | All three explicit error handlers rendering `gen.ErrorResponse` (§4.1) | G.2 | 🟡 see below |
| G.7 | `InternalMiddlewares` composer + ordering wiring test | G.5 | ✅ |
| G.8 | Rewire the internal server via `HandlerWithOptions` | G.6, G.7 | ✅ |
| G.9 | Reshape `alerts_internal_test.go` onto the generated router | G.5 | ✅ |
| G.10 | Convert `*Handler`'s 13 methods to strict | G.8 | ✅ |
| G.10a | `apiResponse` passthrough + 6 request adapters (`gen_adapters.go`) | G.10 | ✅ |
| G.11 | `ObserverMiddlewares` composer (incl. `OpenAPIAuth`) + wiring test | G.10 | ✅ |
| G.12 | Rewire the public server; wrap non-spec routes (§4.2) | G.11 | ✅ |
| G.13 | Reshape the remaining ~4,100 test lines | G.10 | ✅ |
| G.14 | Full verification pass | all | 🟡 all but the live `k3d` run |

---

## §7 — Exit criteria

1. Every route observer serves is registered by generated code from a spec — **no hand-written pattern string for any spec operation** remains.
2. The spec and the served routes **cannot** disagree; nothing needs a test to check that they do.
3. **Authentication behaviour is unchanged**, and each spec states the truth about its own port: the public spec requires Bearer on all 13 but `/health`; the internal spec declares no scheme because none is enforced. `TestInternalSpecDeclaresNoSecurity` fails if that changes without the controller work.
4. `/mcp` and `/.well-known/oauth-protected-resource` still work, still outside the generated router, and **still have logger + recovery**.
5. A missing required FinOps parameter **and** a malformed JSON body both return a `400` carrying the `gen.ErrorResponse` shape, not plain text — at both the wrapper and strict layers.
6. Middleware ordering has exactly one definition per server, in a composer function, driven directly by a wiring test.
7. `make code.gen-check` is clean and the generated server is no longer dead code.
8. Build, vet, lint and the full observer test suite are green; a `k3d` pass exercises both ports, including a real controller-driven alert-rule create.

**Repos.** openchoreo.

---

## Appendix — why not simply delete the generated server instead

A fair question, and the answer is what motivates the document. The alternative to migrating is to stop generating the router half (`std-http-server: false`) and keep the hand-registered mux, retaining only models and the embedded spec.

That would be **cheaper and honest** — it removes dead code rather than leaving it — and it is a legitimate choice. But it gives up four things:

- **The spec stops being enforceable as a routing contract**, permanently. Any check would be a hand-written test comparing pattern strings — never parameter names, types, required-ness, or request/response schemas.
- **The auth divergence in §1 stays undetectable by construction.** Nothing would ever have surfaced it; it was found by asking what the generated server *would* do.
- **Auditing observer gets structurally harder**, which is the reason this plan runs first. On generated routing, the audit middleware's pattern map and the mux's patterns derive from one source and cannot disagree. On a hand-registered mux they can, and compensating for that costs a route-table package plus a bespoke agreement gate — machinery whose only job is to detect a hazard that generated routing does not have.
- **P7's north-bound endpoints** (phased plan, P7 item 6) currently plan to hand-register routes and add *another* spec-agreement test, because that is the established observer pattern. Migrating now means P7 inherits generated routing instead of extending the hand-rolled approach.

If the decision is ever revisited, **§1's auth divergence still needs resolving on its own merits** — it is a real finding independent of how routes are registered.
