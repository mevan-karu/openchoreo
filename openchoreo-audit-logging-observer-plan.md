# Extend audit logging to observer-api

**Companion to:** [`openchoreo-audit-logging-scope.md`](./openchoreo-audit-logging-scope.md) — the requirements register · [`openchoreo-audit-logging-phased-plan.md`](./openchoreo-audit-logging-phased-plan.md) — P0–P10 sequencing · [`openchoreo-audit-logging-p1-plan.md`](./openchoreo-audit-logging-p1-plan.md) — the preceding phase · **[`openchoreo-observer-generated-server-plan.md`](./openchoreo-observer-generated-server-plan.md) — the prerequisite**

**Status:** ✅ **Implemented** — the generated-server migration landed first, and this phase is now wired on top of it · **Date:** 2026-08-30 · **Rewritten:** 2026-08-31, against the post-migration architecture · **Revised:** 2026-09-02, to record what the rebase onto the migrated tree actually changed · **Phase:** P1.5b in the phased plan (P1.5a is the migration)

---

## §0 — Read this first: the prerequisite, and what it deletes from this plan

**This plan assumes observer already serves its REST API from its generated OpenAPI server.** That migration is
[`openchoreo-observer-generated-server-plan.md`](./openchoreo-observer-generated-server-plan.md), and it must land
first. Do not start any task here until it has.

### Why the order was chosen

An earlier revision of this document planned observer auditing against the **hand-registered mux** observer used at
`92a7e1ba`. That version needed two artefacts whose only purpose was to compensate for the absence of generated
routing:

- an importable **route-table package** (`internal/observer/api/routes.go`), because nothing else could enumerate
  what patterns were actually registered; and
- **`TestRegisteredRoutesAgreeWithSpec`**, a bespoke gate asserting a 1:1 match between registered patterns and spec
  operations — because `audit.BuildPatternMap` cross-references operationIds against the spec but **never sees the
  mux**, so a one-character drift would mean `r.Pattern` never matches the map key and that operation is silently
  unaudited forever.

Both were built and both are now **deleted**. They were vacuous on generated routing: the router and the audit
pattern map derive from the same generated spec and cannot disagree. Rather than ship machinery whose job is to
detect a hazard the migration eliminates, the maintainer sequenced the migration first. **This document is rewritten
accordingly, and the earlier hand-mux design is not preserved** — it described a shape that no longer exists.

> **What the rebase actually deleted (2026-09-02).** Reconciling the staged audit work with the migrated tree removed
> `internal/observer/api/routes.go` (the three route tables), `internal/observer/api/wiring.go`
> (`BuildMainMux`/`BuildInternalMux` and their middleware structs), `internal/observer/api/wiring_test.go`, and
> `TestRegisteredRoutesAgreeWithSpec` from the coverage gate. `internal/observer/api` is now a test-only package
> holding `spec_topology_test.go`, which is the state the migration already left it in. Audit slots into the
> migration's own `ObserverMiddlewares` / `InternalMiddlewares` composers instead, exactly as the table above
> predicted.

### What that removes from this plan

| Was needed on the hand-registered mux | Status after the migration |
|---|---|
| `internal/observer/api/routes.go` route tables | **Deleted.** The generated router is the route table |
| `TestRegisteredRoutesAgreeWithSpec` | **Deleted.** Drift is structurally impossible |
| A separate `BuildMainMux` / `BuildInternalMux` pair owned by audit | **Absorbed.** The migration introduces `ObserverMiddlewares` / `InternalMiddlewares` composers on its own terms; audit slots into them |
| Reasoning about `RouteBuilder.With`'s append order putting audit outside `jwtAuth` | **Reframed.** Ordering is now a `[]gen.MiddlewareFunc` slice with one documented definition, exactly as openchoreo-api's `APIMiddlewares` — see §2 |

### What survives from the staged work

Two staged pieces are architecture-neutral and should be **reclaimed, not rebuilt** — they are refactors of
*openchoreo-api's* code that happen to be prerequisites for a second consumer, and neither knows how observer
registers routes:

- **`tools/internal/auditgen`** — the spec-walking generator logic extracted behind a `Config` struct.
- **`internal/auditconfig`** — the koanf decode/validate layer extracted behind an explicit `Vocabulary`.

Both are Phase A below. The observer-specific generator output (`internal/observer/audit/definitions.gen.go`,
`exemptions.go`, `definitions.go`) is also spec-derived and architecture-neutral, so it survives too — only its
*wiring* and the drift gate were coupled to the mux.

---

## §1 — Context

P1 built a policy-driven audit-logging framework for `openchoreo-api`: a generator (`tools/auditgen`) deriving one
`audit.OperationDef` per mutating REST route from the OpenAPI spec, a hand-maintained exemption table for routes
deliberately unaudited, a coverage gate failing CI if a mutating operation is neither audited nor exempted, and
REST/MCP middleware emitting off that table. All of it is wired into exactly one binary.

`observer-api` has real, currently-unaudited mutations. Extending the same framework to it is the first genuine test
of **PR-8** — *"adding audit coverage to a new service requires only: declare actions, initialise the logger, add the
middleware"* — which has never been exercised twice.

**Post-migration, observer's shape is openchoreo-api's shape.** Both serve a generated router from an embedded spec,
both compose middleware as a `[]gen.MiddlewareFunc` slice, both have handlers keyed by operationId. That is what makes
this phase small: the core (`audit.Middleware.Handler`, `audit.NewMiddleware`, `audit.NewPolicySet`) is already
surface-agnostic and needs no change at all.

### Verified operation inventory

The migration **took §2 option 1 and split the spec**, so the 18 operations are distributed across two specs and two
gen packages. The totals are unchanged; where each operation lives is not. Verified against
`gen.GetSwagger()` + `internalgen.GetSwagger()` on 2026-09-02.

| | Count | |
|---|---|---|
| Spec operations, total | **18** | 13 in `observer-api.yaml`, 5 in `observer-internal-api.yaml` |
| Non-GET | **14** | the coverage gate's population — **10 public + 4 internal** |
| Public GETs (not in the gate) | **3** | `Health`, `GetComponentCosts`, `GetRecommendations` |
| Internal GETs (not in the gate) | **1** | `GetAlertRule` |
| `Query*` (POST-as-read, all public) | **9** | `QueryAlerts`, `QueryEvents`, `QueryIncidents`, `QueryLogs`, `QueryMetrics`, `QueryRuntimeTopology`, `QuerySpanDetailsForTrace`, `QuerySpansForTrace`, `QueryTraces` |
| Alert-rule writes + webhook (internal port) | **4** | `CreateAlertRule`, `UpdateAlertRule`, `DeleteAlertRule`, `HandleAlertWebhook` |
| Genuinely audited | **1** | `UpdateIncident` (public) |

**The split is why the audit table is filtered per port.** `audit.BuildPatternMap` returns an error for an
operationId it cannot resolve to a route in the spec it was handed, so passing the whole table to either port's
middleware fails observer's startup. `observeraudit.OperationsIn(swagger)` returns just that spec's share, and
`TestEveryDefinedOperationBelongsToExactlyOneSpec` closes the hole filtering opens — an operation matching neither
spec would otherwise be dropped from both ports with no error and no failing test.

**The small coverage number is the point.** The work is the extraction and the wiring, not the operations.

> **`QuerySpanDetailsForTrace` needs its own exemption reason, citing `a65229ba`.** The other eight `Query*` operations
> are self-evidently reads — their paths end in `/query`. This one is `POST /api/v1alpha1/traces/{traceId}/spans/{spanId}`,
> and became a POST purely to carry a request body (*"replace GET method API in Observer used to fetch span details with
> a POST method API"*). It is the only exemption whose justification lives in a commit rather than in the route's own
> shape, so write the commit into the reason string or it will be re-litigated at every review.

### Decisions already settled

- **The internal-port mutations (alert-rule writes + webhook) are exempted**, not audited-as-anonymous — the same
  reasoning as openchoreo-api's `HandleAutoBuild`: no JWT middleware runs there, so there is no real actor to record.
  **Settled: the migration took §1 option B**, so the internal spec declares no security scheme, port 8081 stays
  unauthenticated, and all four exemptions stand. The earlier conditional ("if option A lands, this reverses") is
  resolved and struck. The audit middleware is still wired on that port, so lifting an exemption later — e.g. once
  phased-plan task 7.22 gives it auth — needs no wiring change.
- **Observer has no mutating MCP tools.** All 13 registered tools are `query_*` / `get_*` reads, so MCP audit wiring
  is out of scope. Observer has **no `ToolPermission` registry** (unlike `pkg/mcp/tools`) to classify tools from a
  declared authz action, so the guard is a **name-and-count pin**, not a permission check — say so in the test, or a
  reader will assume a stronger guarantee than it gives.
- **Observer's audit config gets full policy-list support** from day one, matching openchoreo-api's `audit.policies`
  shape, not just an enabled/disabled toggle.

---

## §2 — The one ordering constraint that still matters

Everything about *where* audit sits in the chain survives the migration, because it is a property of how the audit
middleware reads its actor, not of how routes are registered.

`Middleware.Handler` captures `ctx` from `NewAuditContext` **before** calling `next`, and emits from that same
captured `ctx` in its `defer`. The JWT middleware adds its `SubjectContext` to a **child** context created downstream
via `r.WithContext`, which never propagates back to the closure's variable. So if audit sits **outside** auth,
`EmitFromContext → ExtractActor(ctx)` finds no subject and returns `Actor{Type: "anonymous", ID: "anonymous"}` on
**every** audited event — including successful authenticated ones. Nothing fails: the event still emits, with the
wrong actor.

openchoreo-api documents and orders around exactly this, in `OpenAPIMiddlewares` (**not** `APIMiddlewares` — an
earlier revision of this document used the old name):

```
logger → unauthenticatedAudit → auth → audit → webhookRawBody → handler
```

> *"audit sits inside auth so SubjectContext is already populated for it. unauthenticatedAudit sits outside auth — the
> only position that can see a request auth itself rejects, since auth short-circuits and never calls next."*

**Observer's composers use the same order**, and their doc comments carry the same reasoning. As landed:

```
ObserverMiddlewares: logger → recovery → unauthenticatedAudit → auth → audit → contentType → handler
InternalMiddlewares: logger → recovery → audit → handler
```

Observer keeps its own `Recovery` (openchoreo-api has none in this chain), and `RequireJSONContentType` stays
innermost so an unauthenticated caller cannot probe it. The internal chain has no `unauthenticatedAudit`: with no auth
on that port there is no 401 for it to observe, and the pairing only makes sense where a rejection is possible.

> **Note `[]gen.MiddlewareFunc` is applied last-to-first** by `HandlerWithOptions` — the last entry is outermost, as
> `OpenAPIMiddlewares`' comment records. Getting this backwards produces exactly the anonymous-actor failure above,
> with no test failure. `TestUpdateIncidentAuditEvent` therefore asserts `actor.id` **and** `resource.id` on an event
> emitted through the real composed chain, not merely that the middleware is present.

---

## §3 — Phases

### Phase A — Shared libraries (zero behaviour change for openchoreo-api)

**A1. Generator library.** `tools/internal/auditgen` — an importable package holding the spec-walking logic
(`pathTail`, `singularize`, `buildDefinitions`/`deriveDefinition`, `checkNoOrphanCategories`, `renderDefinitions`),
with the currently-hardcoded package vars generalised into a `Config`:

```go
type Config struct {
    ResourceCategories   map[string]string
    ExcludedOperationIDs map[string]bool
    SingularOverrides    map[string]string
    ActionSuffixSegments map[string]bool
    Overrides            map[string]OperationDef // keyed by operationId
}
func BuildDefinitions(swagger *openapi3.T, cfg Config) ([]OperationDef, error)
```

`tools/auditgen/main.go` becomes a thin wrapper over its own `config.go` data file.

**Verification:** run `go run ./tools/auditgen` before and after, and diff
`internal/openchoreo-api/audit/definitions.gen.go` **byte-for-byte** — must be identical.

**A2. Audit-config library.** `internal/auditconfig` — the koanf decode/validate layer from
`internal/openchoreo-api/config/audit.go`, with the package-level `validResources/validOperationIDs/validActions`
generalised into an explicit parameter:

```go
type Vocabulary struct{ Resources, OperationIDs, Actions []string }
func NewVocabulary(ops []audit.Operation) Vocabulary
func (c *AuditConfig) BuildPolicySet(vocab Vocabulary, knownActorTypes []string) (*audit.PolicySet, error)
func (c *AuditConfig) Validate(path *config.Path, vocab Vocabulary, knownActorTypes []string) config.ValidationErrors
```

`internal/openchoreo-api/config/audit.go` becomes a thin re-export. `internal/config` is already generic and unchanged.

**Verification:** the existing `audit_test.go` suite passes unchanged (import paths only), with identical error messages.

> **Both A1 and A2 already exist, staged.** Reclaim them rather than rebuilding — see §0.

### Phase B — Observer's audit package

**B1. One generator serving every API server, not one generator per server.** The plan originally called for a
separate `tools/observerauditgen` binary. That was the wrong shape and is **superseded**: `tools/auditgen` now takes a
`-service` flag and selects from a registry (`tools/auditgen/services.go`), because the only thing that actually
differed between the two mains was *data* — which spec to walk, which operations to exclude, which resource kinds map
to which category — while the generation logic already lived in the shared `tools/internal/auditgen`. Two main
packages over one library were two places for identical boilerplate to drift.

```
go run ./tools/auditgen -service openchoreo-api
go run ./tools/auditgen -service observer
```

A service declares one or more specs; observer declares both of its own. Per-spec `auditgen.Config`s rather than one
shared config, because `BuildDefinitions` runs `checkNoOrphanCategories` against the single spec it is handed — a
resource kind real in one spec would be flagged as an orphan while walking the other. The concatenated tables are
sorted and duplicate-checked in `mergeDefinitions`, since `RenderDefinitions` does not sort and `BuildDefinitions`'
own duplicate check cannot see across specs.

`make audit-gen` loops `AUDIT_SERVICES`; the separate `observer-audit-gen` target is gone. Adding a third service is
a registry entry plus a config file, not a new binary.

> **Walk BOTH specs.** Observer's coverage gate is about its whole served surface. Walking only the public spec would
> leave the four internal operations invisible to the gate, so a new unaudited alert-rule write would pass CI.

> **Read each spec through its `GetSwagger()`, never the YAML.** The raw specs declare camelCase operationIds
> (`createAlertRule`, `updateIncident`); oapi-codegen normalises them to PascalCase in the embedded runtime spec, and
> PascalCase is what the generated routers and `BuildPatternMap` both use. Someone will eventually "fix" the exemption
> keys to match the raw YAML; the package comment is what stops them.

> **The derived resource kind for alert rules is `rule`, not `alertrule`.** `pathTail` takes the last non-parameter
> segment: `/alerts/sources/{sourceType}/rules/{ruleName}` → `rules` → `rule`. Moot while all four are exempted
> (`BuildDefinitions` skips excluded operationIds before `deriveDefinition` runs, so no category is required), which is
> why `observerInternalResourceCategories` is empty. The moment an exemption lifts, generation fails with "no category
> for kind" — the right prompt to decide it deliberately, recorded in that map's doc comment.

**B2.** `internal/observer/audit/exemptions.go` — `RESTExemptions map[string]string` covering the 9 `Query*`
operations ("a read expressed as POST, not a state-modifying action", plus the `a65229ba` note for
`QuerySpanDetailsForTrace`) and the 4 internal-port mutations ("no JWT middleware on the internal port — no real actor
to record"), **subject to which auth option the migration took**.

**B3.** `internal/observer/audit/definitions.go` — `GetOperations()` returning `audit.Operations(generatedOperationDefs())`.
No MCP bindings file: no mutating MCP tools exist.

**B4.** `internal/observer/audit/audit_coverage_test.go` — the CI gate. Re-derive observer's non-GET operationIds from
**both** `gen.GetSwagger()` and `internalgen.GetSwagger()` and assert each is defined-or-exempted, never both, never
neither; every exemption carries a non-empty reason and names a real live operation in either spec; every operation
resolves to `management` or `authorization`; the total is pinned at **14**. Plus
`TestEveryDefinedOperationBelongsToExactlyOneSpec` (see §1) and the MCP pin.

> **The MCP pin was originally a count check that enforced nothing, and its comments claimed otherwise.** Review caught
> it: `MCPToolNames` is a hand-written map, nothing derived it from `registerTools`, and the test only asserted
> `len(...) == 13` — a map literal checked against a constant in the same package. Adding a 14th tool and not touching
> the map passed CI silently, which is the exact failure the comment said it prevented.
>
> The SDK has no way to read tools back off a `*mcpsdk.Server` (verified: no exported accessor, `listTools` is
> unexported). So the pin now connects a real client over `mcpsdk.NewInMemoryTransports()` and issues `tools/list`,
> diffing the result against `MCPToolNames` in both directions. Required exporting `observermcp.NewServer`.
> Verified by adding a bogus entry and watching the gate fail.
>
> It remains a *name* pin, not a permission check — observer still has no `ToolPermission` registry to classify a tool
> as state-modifying — and both comments now say so plainly.

> **No registered-pattern ↔ spec-path assertion.** That gate existed only for the hand-registered mux. On generated
> routing the router and the pattern map come from one source. It is deleted, and the test file carries a comment
> saying so — if someone proposes re-adding it, the answer is that it can no longer fail.

### Phase C — Wiring

**C1. Config.** Add `Audit auditconfig.AuditConfig` to `internal/observer/config.Config`. `audit.policies` is a nested
list observer's flat env-var mapping cannot represent, so load it the way observer already loads `auth.subject_types`:
extend the supplementary YAML load in `config.Load()` to also decode an `audit:` key. Call `cfg.Audit.Validate(...)`
from `(*Config).validate()`, translating `config.ValidationErrors` via the existing `errs.OrNil()`. Derive
`knownActorTypes` from `cfg.Auth.SubjectTypes`.

> **The Go support is only half of OP-1/OP-3 — the chart has to plumb it too, and initially did not.** Review caught
> this: `observer-auth-configmap.yaml` rendered only `auth: subject_types:`, `OBSERVER_AUTH_CONFIG_PATH` points at
> that same file, and `Config.Audit` is tagged `koanf:"-"` so the env-var loader cannot reach it either — not even
> `audit.enabled`. A Helm-installed observer was therefore pinned to `AuditDefaults()` with no way to configure or
> disable audit. Fail-open toward logging, so nothing was broken, but the feature was unreachable.
>
> Fixed by mirroring the control-plane chart: `observer.audit.{enabled,defaults,policies}` in `values.yaml`, rendered
> into the same ConfigMap alongside `auth:`. **Lesson for the next service: "the Go code supports it" is not
> OP-1/OP-3 done — grep the chart before marking a config task complete.**
>
> Left as noted-not-done: audit policies still have to be authored inside the *auth* config file, which is
> surprising. A dedicated `OBSERVER_AUDIT_CONFIG_PATH` falling back to the auth path would be cleaner and is a
> reasonable follow-up.

**C2. Logger.** Replace `observermiddleware.Logger` with the shared, already-hardened
`internal/server/middleware/logger.Middleware` — it generates and validates `X-Request-ID`, which observer's has never
done, keeping audit and access-log correlation intact. Delete the old one once unreferenced.

> **This changes log volume.** Observer's logged at `Debug("HTTP request", …)`; the shared one logs at
> `Info("ACCESS-LOG", …)`. Observer goes from logging no requests at default level to logging every one. That is the
> right behaviour and it is what openchoreo-api does, but it is a visible operational change that belongs in the PR
> description. It also **delivers phased-plan task 4.5** — while leaving **4.3** open, since `initLogger` still returns
> a `slog.NewTextHandler` under `LOG_LEVEL=debug`, so the new access log is correlatable and unparseable at once.

> **The swap is not confined to `main.go`, and that is what made it a merge hazard.** The migration's
> `ObserverMiddlewares` and `InternalMiddlewares` both call `observermiddleware.Logger` while this phase deletes it —
> a conflict git merges cleanly and the compiler then rejects. Both composers now use `apilogger.Middleware`.
>
> It also **rewrites the migration's own ordering assertion.** `TestInternalMiddlewareOrdering` discriminates
> logger-outside-recovery by looking for the logger's post-`next` line surviving a panicking handler; that line was
> `"HTTP request"` and is now `"ACCESS-LOG"`. Changing only the string would silently turn a discriminating test into
> a vacuous one, so the reversal check the migration's Phase 1 describes was re-run: reversing the slice still fails
> exactly that assertion while every other one passes.

**C3. Audit middleware — built inside the composers, exactly as `OpenAPIMiddlewares` does it.** `main.go` builds only
the policy set and the shared emitter, then hands each composer `AuditEmitter` + `AuditEnabled`:

```go
auditPolicies, err := cfg.Audit.BuildPolicySet(
    auditconfig.NewVocabulary(observeraudit.GetOperations()), cfg.Auth.KnownActorTypes())
auditEmitter, err := audit.NewEmitter("observer", auditPolicies, audit.NewLogger(logger))
```

An earlier revision had `main.go` build `auditMw` itself and pass `.Handler` into the composer. That is **not** the
openchoreo-api pattern and is superseded: `OpenAPIMiddlewares` constructs both the audit and unauthenticated-audit
middlewares internally from the emitter, so ordering *and* construction have one definition and `main.go` owns
neither.

**The spec was split, so each gen package gets its own `audit.NewMiddleware`** with its own `GetSwagger` — built by
`newAuditMiddleware` in `handlers/wiring.go` — sharing one emitter so a single configured policy applies across both
ports and `/mcp`. Each is fed `observeraudit.OperationsIn(swagger)` rather than the whole table, for the
`BuildPatternMap` reason in §1.

> **Both composers return an error and observer propagates it.** `BuildPatternMap` fails loudly at startup when an
> operation's `RESTResourceParam` names a path parameter the resolved pattern lacks — reachable, since observer's
> parameter names are hand-written in the spec. Both also reject a nil `AuditEmitter`, which would otherwise reach
> `Emitter.Emit` and panic on the first request instead of at startup
> (`TestMiddlewareComposersRequireAuditEmitter`). `cmd/observer/main.go` reports these with `logger.Error` +
> `os.Exit(1)`, matching `OpenAPIMiddlewares` and observer's own adapter constructors rather than the `log.Fatalf`
> used elsewhere in that file.

**C4. PR-7 parity.** Wire `unauthAuditMw` **outside** auth on the public routes so a 401 is recorded — the hole P1 task
1.6 closed for openchoreo-api's REST routes and PR #4571 then closed for its exec, wirelogs and `/mcp` chains. Observer's
`/mcp` chain gets its own instance stamped `audit.OriginMCP`, so an MCP token rejection is not recorded as REST. Two
instances, never nested — each seeds its own context marker, so an outer instance's marker is never flipped by an inner
one, and a nested pair would double-emit.

**C5. Internal port.** `InternalMiddlewares` builds an audit middleware for `internalgen.GetSwagger` too, even though
every operation there is exempted today — so `OperationsIn` resolves to an empty set and it is currently a
pass-through. Coverage becomes automatic the moment an exemption lifts — e.g. once the internal port gets auth
(phased-plan task **7.22**) — with no wiring change needed.

### Phase D — Handler instrumentation

Only `UpdateIncident` needs a code change: after the error check, `audit.SetResource(ctx, &audit.Resource{ID: id, Name: id})`
using the **validated path parameter**, not `resp.IncidentId` — the path value is always present, so there is no branch
where the event lacks a `resource.id`.

**The hierarchy is fixed in the authz path, and the missing hierarchy turned out to be an authorization defect rather
than an audit one.** This is worth recording carefully, because an earlier revision of this plan got the diagnosis
wrong and proposed a handler-side workaround.

openchoreo-api gets hierarchy for free: `AuthzChecker.Check` calls `audit.SetHierarchy` in the shared authz path,
before the PDP evaluates, so denials carry it too. Observer looked unable to reuse that, because
`alert_incident_authz.go`'s `UpdateIncident` passed an **empty** `authzcore.ResourceHierarchy{}` — described in its
own comment as a deliberate "generic permission check". The wrong conclusion was that the mechanism did not carry
over. The right conclusion is that **the scope-free check was itself the bug**:

- `resourceHierarchyToPath({})` returns the `"*"` wildcard path, so the PDP was being asked whether the subject may
  update incidents *globally*.
- `resourceMatch` is prefix-based, and a request path of `"*"` is matched only by a policy whose own resource is
  `"*"`. So a namespace- or project-scoped grant of `incidents:update` was **silently denied**, and the operation
  effectively required a cluster-wide binding.

Verified against the real matcher rather than reasoned about:

| policy resource | before (empty hierarchy → `"*"`) | after (real hierarchy) |
|---|---|---|
| `*` (cluster-wide) | allow | allow |
| `ns/acme` | **deny** | allow |
| `ns/acme/project/p1` | **deny** | allow |
| `ns/other` | deny | deny |

So the change is safe in the only direction that matters: **nothing permitted today stops being permitted**, and
correctly-scoped grants start working. It is a widening for legitimate callers, not a tightening — and a scoped grant
now confines the caller to its own scope instead of being useless.

**The fix, therefore:**

1. `IncidentPutRequest` names no scope, so the hierarchy has to be read from the stored incident. Added
   `IncidentEntryStore.GetIncidentEntry` (non-locking; sharing one `incidentEntrySelectSQL` + `scanIncidentEntryRow`
   with the update path's loader so an 18-column scan is not maintained twice) and
   `IncidentsUpdater.IncidentScope`.
2. `alertIncidentServiceWithAuthz.UpdateIncident` reads the scope, derives the hierarchy through the same
   `ComponentScopeAuthz` helper the query operations use, and authorizes on it. A failed scope read short-circuits:
   the PDP is never consulted (`TestAlertIncidentAuthz_UpdateIncident_ScopeLookupFails` uses a PDP mock with no
   expectations, so a call fails the test).
3. `observerAuthz.CheckAuthorization` calls `audit.SetHierarchy` — **before** the `pdp == nil` early return, so
   hierarchy is recorded on denials, on PDP errors, and in authz-disabled deployments. This is the exact analogue of
   `AuthzChecker.Check`, and it means every observer operation gets hierarchy without each caller capturing it.
4. **No handler-side `SetHierarchy`.** With the authz path recording it there would be two sources, and the handler's
   would only fire on success.

`TestUpdateIncidentAuditEvent` and `TestUpdateIncidentAuditEventOnDenial` both drive the real authz wrapper with a
mock PDP and assert `resource.namespace`/`project`/`component` — the denial case being the one that proves recording
before the decision was the right call. `TestAlertIncidentAuthz_UpdateIncident_AuthorizesOnIncidentHierarchy`
asserts on the `EvaluateRequest` itself, since a regression to the scope-free form would otherwise still pass.

> **`audit.SetResource` replaces the `*Resource` wholesale rather than merging**, so it also discards the middleware's
> pre-call seed. Anything that must survive has to be in that one call. `SetHierarchy` is a separate field and does not
> collide.

> **The P0/#4571 ordering rule — *after the error check, before the response conversion* — largely dissolves under strict
> handlers**, and that is worth recording rather than silently dropping. The rule exists because openchoreo-api's
> handlers call `convert[...]` into a generated response type *after* the mutation, and a conversion failure returned 500
> with an empty resource. A strict handler returns a typed `*ResponseObject` directly, with no separate conversion step
> that can fail. **The surviving half of the rule still binds:** place the call after the `err != nil` check, never
> immediately after the service call.

### Phase E — Coverage matrix and verification

**E1. One coverage tool serving every API server**, for the same reason as B1 — and `tools/observerauditcoverage` is
**superseded** before it shipped. `tools/auditcoverage` takes `-service` and selects from its own registry
(`tools/auditcoverage/services.go`). Output paths stay `docs/audit/coverage-matrix.md` and
`docs/audit/observer-coverage-matrix.md`.

The split here is narrower than the generator's, and deliberately so:

- **Shared:** the REST table and the exemptions table. Both services derive them from the same `audit.Operation`
  shape, and the two hand-written copies had already drifted (`1 audited operation(s)` vs `audited operations`, and
  two different wordings of the same resource-identity column).
- **Per-service:** the MCP section — openchoreo-api only, since observer registers no mutating tools — and
  `knownNonEvents`, which is hand-written narrative about one service's specific gaps. Collapsing that into a shared
  renderer would produce either a lowest-common-denominator document or a template with a hole for every difference.

`TestRender_Deterministic`, `TestCoverageMatrix_IsFresh` and a new `TestServiceRegistry_IsComplete` all iterate the
registry, so a third service inherits every gate the moment it is registered. `make audit-coverage-matrix` loops
`AUDIT_SERVICES`; `observer-audit-coverage-matrix` is gone.

> **openchoreo-api's committed matrix is byte-identical across the unification** — verified by regenerating and
> diffing, which is the check that keeps this a refactor rather than a content change. Observer's matrix (not yet on
> `main`) changed by three lines: the tool name, the pluralisation fix, and one added non-event about FinOps
> parameter-binding 400s bypassing audit.

**E2.** Full verification:
- `go build ./...`, `go vet ./...` — clean.
- `make code.gen && make code.gen-check` — both generators produce committed output; openchoreo-api's
  `definitions.gen.go` byte-identical to before Phase A.
- `go test ./tools/... ./internal/openchoreo-api/... ./internal/observer/... ./internal/auditconfig/... ./internal/server/middleware/... -count=1` — zero failures, including every pre-existing openchoreo-api audit test (the regression proof for Phase A).
- `make lint` — 0 issues.
- Targeted tests for one exempted operation and for `UpdateIncident` (event emitted, **actor populated**, on success and
  on denial). **For at least one, temporarily break the wiring and confirm the test actually fails** before restoring it —
  the empirical-proof standard applied throughout P0/P1.
- **Write down the PR-8 cost**: what the extraction actually took, so the claim is measured rather than asserted.

---

## §4 — Task breakdown

| # | Task | Reqs | Dep | State |
|---|---|---|---|---|
| — | **Generated-server migration complete** | — | *prerequisite* | ✅ |
| 1.5b.a | Reclaim `tools/internal/auditgen`; byte-identical `definitions.gen.go` gate | PR-8 | — | ✅ |
| 1.5b.b | Reclaim `internal/auditconfig`; openchoreo-api config becomes a re-export | PR-8 | — | ✅ |
| 1.5b.c | ~~`tools/observerauditgen`~~ → **`tools/auditgen -service observer`**; one target for all services | PR-1 | a | ✅ |
| 1.5b.d | `exemptions.go` — 13 exemptions with reasons (option B landed, so 13 not 9) | PR-1 | c | ✅ |
| 1.5b.e | Coverage gate over **both** specs: defined-or-exempted, reasons, categories, count pin, one-spec-each, MCP name pin | PR-3 | d | ✅ |
| 1.5b.f | Observer audit config, loaded from the supplementary YAML, validated at startup | OP-1, OP-3 | b | ✅ |
| 1.5b.f2 | **Helm plumbing** — `observer.audit` in values, rendered into the ConfigMap observer reads | OP-1, OP-3 | f | ✅ |
| 1.5b.g | Logger swap to the shared `logger.Middleware`, incl. both composers and the migration's ordering test | EM-16 | — | ✅ |
| 1.5b.h | Compose audit **inside** auth in `ObserverMiddlewares`, built in the composer; propagate construction errors | PR-1 | c, f | ✅ |
| 1.5b.i | `NewUnauthenticatedMiddleware` outside auth (`OriginAPI`) and on `/mcp` (`OriginMCP`) | PR-7 | h | ✅ |
| 1.5b.j | Audit middleware on the internal server, per-spec-filtered; exemptions lift cleanly later | PR-1 | h | ✅ |
| 1.5b.k | `SetResource` in `UpdateIncident`, after the error check, on the strict handler's `ctx` | PR-1 | h | ✅ |
| 1.5b.k2 | **Fix `UpdateIncident`'s scope-free authz check**; `SetHierarchy` in `CheckAuthorization` | PR-1 | k | ✅ |
| 1.5b.l | ~~`tools/observerauditcoverage`~~ → **`tools/auditcoverage -service observer`**; freshness gate per service | DO-2 | e | ✅ |
| 1.5b.m | Verification pass, including a deliberate-breakage proof | — | all | 🟡 all but the live `k3d` run |
| 1.5b.n | **Write down the PR-8 cost** | PR-8 | m | ✅ see below |

### Open, deliberately not done in this phase

1. **Un-exempting the four internal-port operations is worth a second look.** They are the only genuinely
   state-modifying operations observer has, so as shipped observer's audit trail holds no record that an alert rule
   was created, updated or deleted. The exemption reason ("no JWT middleware, no real actor") is true but only covers
   the *actor* — the action, resource, source IP, request ID and timestamp are all fully recordable and useful with
   `actor: anonymous`. The openchoreo-api precedent cited (`HandleAutoBuild`) is a single webhook; this is a whole
   resource's write surface. Cost to reverse is roughly one `resourceCategories` entry (the derived kind is `rule`)
   plus lifting four exemptions, and the policy layer can suppress the volume if it proves noisy. **Recorded as a
   question for the maintainer, not changed — the plan marks this decision settled.**
2. **A dedicated `OBSERVER_AUDIT_CONFIG_PATH`**, so audit policies need not be authored inside the auth config file.
3. **`KnownActorTypes` divergence between the two services.** Observer reads `st.Type`; openchoreo-api reads its
   `Subjects` map keys. Both now feed the same shared validator, so if a key ever differs from its `Type`,
   openchoreo-api will reject a valid policy. Pre-existing and outside this phase, but the shared package makes the
   two comparable for the first time.

### The PR-8 cost, measured (task n)

PR-8 claimed *"adding audit coverage to a new service requires only: declare actions, initialise the logger, add the
middleware."* Observer is the first second consumer, so here is what it actually took.

**Held up.** The core needed **no change at all**: `audit.Middleware`, `audit.NewEmitter`, `audit.NewPolicySet`,
`audit.NewUnauthenticatedMiddleware` and `audit.SetResource` were already surface-agnostic and were reused verbatim.
The two extractions (`tools/internal/auditgen`, `internal/auditconfig`) were genuinely one-time costs paid by the
first second consumer, and openchoreo-api's generated definitions and coverage matrix both came out byte-identical
across them — the regression proof that they were refactors, not rewrites.

**Cost the claim did not anticipate**, all of it from observer serving **two** specs rather than one:

1. `audit.BuildPatternMap` errors on an operationId absent from the spec it is given, so a two-spec service cannot
   hand its whole table to either port. That needed `observeraudit.OperationsIn` plus a test
   (`TestEveryDefinedOperationBelongsToExactlyOneSpec`) closing the silent-drop hole filtering opens.
2. The generator and the coverage tool each needed a per-spec config and a merge step.
3. The coverage gate had to read both specs, not one.

**Cost the claim did anticipate but understated:** "initialise the logger" was a swap of `observermiddleware.Logger`
for the shared `apilogger.Middleware`, which is a cross-cutting edit — two middleware composers, `main.go`, and a
rewrite of the migration's own ordering assertion, since that test keyed on the old logger's message string.

**Cost the claim missed entirely:** *plumbing the config through the deployment.* The Go work landed and the task was
marked ✅ while the Helm chart still had no `audit:` key, so the feature was unreachable in any real install. PR-8's
three steps should be four — "declare actions, initialise the logger, add the middleware, **plumb the config through
the chart**."

**Also not free, and also not in the claim:** hierarchy. openchoreo-api gets it from the shared authz path; observer's
equivalent check passed an empty hierarchy, so the same call populated nothing — and chasing *why* surfaced an
authorization defect that had nothing to do with audit (see Phase D). Extending audit to a service is where that kind
of thing gets found, which is an argument for doing it, but it is not the "three small steps" PR-8 describes.

**Verdict:** the claim holds for the middleware wiring of a single-spec service. For anything else, add: decide how
the operation table maps onto specs, decide where hierarchy comes from, and plumb the config through the chart. All
three are design steps, not boilerplate.

---

## §5 — Exit criteria

1. Observer emits a correct, **correctly-attributed** audit event for `UpdateIncident` on success, on denial (a 403
   from `ErrAuthzForbidden` → `result: denied`) and on failure (a 500 → `result: failure`) — `actor.id` is the token's
   `sub`, not `anonymous`, and the event carries `resource.id` plus the namespace/project/component hierarchy.

   > An earlier revision had only one non-success test, named `...OnDenial` but driving a 500 and asserting
   > `result: "failure"` — so the denial branch that `determineResult` actually maps from a 403 was untested and this
   > criterion was unmet. Now split into `...OnFailure` and a real `...OnDenial`.
2. Every other non-GET operation is audited or exempted with a real reason; the gate fails if a new mutating operation
   is added unaudited.
3. A 401 on observer's protected routes emits an `unauthenticated` event, stamped `OriginAPI`; a 401 on `/mcp` emits one
   stamped `OriginMCP`; neither double-emits.
4. `openchoreo-api`'s `definitions.gen.go` **and** its committed coverage matrix are **byte-identical** to before
   Phase A and before the tool unification, and its full audit test suite passes unchanged.
5. Observer's access log carries a validated `X-Request-ID`, correlatable with its audit events.
6. Both coverage matrices are generated by one tool, committed, and gated for freshness.
7. Build, vet, lint, `code.gen-check` and the full test suite are green.
8. **The PR-8 cost is written down** — this phase's actual purpose. See §4.
9. **No hand-mux artefact survives.** `internal/observer/api/routes.go`, `api/wiring.go`, `api/wiring_test.go` and
   `TestRegisteredRoutesAgreeWithSpec` are deleted, and nothing reintroduces a route table audit would have to agree
   with.
10. **One audit generator and one coverage tool serve every API server**, selected by `-service`. Adding a third is a
    registry entry, not a new binary. The shared render template asserts nothing service-specific — the
    `mcp_bindings.go` note is a per-service `RenderOptions.DocSuffix`, so observer's generated file no longer claims a
    layout it does not have.
11. **Audit is configurable in a Helm-installed observer**, not just in the Go code — `observer.audit` in values is
    rendered into the ConfigMap observer actually reads (see C1).
12. **Observer refuses to start on a stale generated audit table.** `VerifyOperationsPartition` runs before the
    emitter is built and exits non-zero if any audited operation is claimed by neither served spec — the runtime
    backstop for the one hazard `OperationsIn`'s filtering introduces, matching how the composers already treat a nil
    emitter.

**Repos.** openchoreo.
