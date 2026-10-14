# Audit Read Path — Observer Implementation, Commit by Commit

**Parent:** [`openchoreo-audit-observer-api-plan.md`](./openchoreo-audit-observer-api-plan.md) §1–§4.
This is the commit-level breakdown of that plan's contract, service and authorization work.
**Phase:** P7 · **Status:** all three commits implemented, none pushed · **Date:** 2026-09-11
**Repo:** `openchoreo` only.

> **Three commits, one PR.** The repository squash-merges, so all three land on `main` as a single
> commit — which is what makes the ordering below safe. §0 explains why the ordering is a design
> decision rather than a filing convenience.
>
> **The template is `94209dba5`** (#4658, platform logs). Every file here has a `platform_logs`
> counterpart. Read it before writing the audit one; the shape is the specification, and divergences
> are called out explicitly below.
>
> **Two operations, not one.** `QueryAuditLogs` returns records; `QueryAuditLogFilterValues` returns
> the distinct values one filter can take. They share a service, a permission and an adapter, and
> are separately declinable by a module.

---

## 0. The ordering, and why it is this way

The obvious split is *contract → service → authz*. Taken naively, that leaves a middle commit in
which the endpoints are wired and answering with no permission check — a moment where anyone
authenticated can read the whole audit trail.

**The fix costs nothing: wire the service in the authz commit, not the service commit.**

The platform-logs handler already returns a clean `500` when its service field is nil
(`internal/observer/api/handlers/platformlogs.go:32-40`, `ErrorCodeV1PlatformLogsServiceNotReady`).
Audit inherits that, which means:

| Commit | Endpoint state at runtime | Reachable? |
|---|---|---|
| 1 — contract | `501 not implemented` | Yes, and says so honestly |
| 2 — service | `500 service not initialized` — service exists, `main.go` does not construct it | No |
| 3 — authz | `200` / `403` — constructed **already wrapped** in the authz decorator | Yes, gated |

So the endpoints become live in the same commit that gates them. Commit 2 is fully tested through
its unit tests without ever being reachable over HTTP. **Do not "helpfully" wire `main.go` in
commit 2.**

---

## Commit 1 — `feat(api): define the audit logs query contract on observer and adapter` · **landed**

Contract, generated code, Go interfaces, types, and the AX-5 definitions. The observer answers `501`
on both operations.

### 1.1 The two specs

Both gained `POST /api/v1alpha1/audit-logs/query` and
`POST /api/v1alpha1/audit-logs/filter-values`, with `501` declared on every new operation. Filter
set, casing rule, timeline semantics and
filter-values semantics are in the parent plan §C-1…§C-5; do not re-derive them here.

### 1.2 Codegen

**No `golang.mk` change** — both specs are already inputs to `openapi-codegen` (`:196-204`). The
original plan's "a new spec is not picked up automatically" does not apply: this adds paths to
existing specs.

The generated `StrictServerInterface` gained both methods, and **the build failed until handlers
existed**. That failure is the proof the routes are wired.

### 1.3 Go interfaces and types

| File | Contents |
|---|---|
| `pkg/observability/auditlogs.go` | `AuditLogsAdapter` with `GetAuditLogs` **and** `GetAuditLogFilterValues`, plus params/result/record/timeline/filter-value structs |
| `internal/observer/types/auditlogs.go` | wire types for both operations, filters nested as `AuditLogsActorFilter` / `AuditLogsResourceFilter` |
| `internal/observer/service/interfaces.go` | `AuditLogsQuerier` with both methods — one interface because both reads disclose the same content and so carry the same permission |
| `.mockery.yaml` | `AuditLogsQuerier` under the observer service block |
| `internal/observer/types/errors.go` | `OBS-V1-AL-01…07` |

`AuditLogRecord` mirrors the published event (`internal/server/middleware/audit/types.go`) rather
than inventing a shape, keeping snake_case keys so a response can be compared against an exported
log line key for key. The UI prototype validates this: its detail panel renders the raw wire record
as JSON (`openchoreo-audit-logs-prototype.html:2460`).

Two fields the platform-logs analogue has no counterpart for and which must not be dropped: the
collector provenance (`collector.{namespaceName,podName,containerName}`) that publishing-plan §1.3.2
indexes for the `producer` cross-check, and the preserved raw `log` line.

### 1.4 Handlers that compile and tell the truth

`internal/observer/api/handlers/auditlogs.go` — both methods returning `501`, with distinct codes
(`OBS-V1-AL-05` for records, `OBS-V1-AL-07` for filter values) because a module may serve records
while unable to aggregate.

Two `apiResponse` visitors added in `response.go`, plus both operations in that file's doc-comment
enumeration.

**No `Handler` field and no `NewHandler` parameter yet.** The original plan added them here to avoid
"signature churn across every test helper" — but no test helper calls `NewHandler`; handler tests
build `&Handler{...}` literals directly, so the field belongs with the code that reads it (commit 2)
and the parameter with the wiring (commit 3).

### 1.5 AX-5 — the record read defined, the picker read exempt

The coverage gate required both new operations to be either defined or exempted.

**`QueryAuditLogs` is defined**, under a new `read_` verb and a new `CategoryAccess` — parent plan
§4. `RESTExemptions` carries a comment saying it is deliberately absent, so a future reader does not
add it back as an oversight.

**`QueryAuditLogFilterValues` is exempt on volume, not on sensitivity.** A picker populates on every
interaction that changes a query; auditing it would bury the record reads it leads to, which are
audited. Its exemption reason says exactly that, and says the same permission still gates it — so
nobody reads the exemption as "this one is not sensitive".

The pinned operation count in `audit_coverage_test.go` moved 20 → 22 (two new public operations),
with its breakdown comment updated. Neither original plan mentioned that pin.

### 1.6 Verification

```sh
make code.gen && make lint && make test
```

All green. `TestAuditCoverage` passes with both operations defined; the observer builds; both
endpoints answer `501`.

---

## Commit 2 — `feat(api): implement the audit logs queries in observer` · **done**

Service, adapter methods, real handlers, tests. **No `main.go` change** — see §0.

### 2.1 The adapter methods

`internal/observer/service/audit_logs_adapter.go`, modelled on `platform_logs_adapter.go`.

**Both are methods on the existing `*LogsAdapter`**, not a new adapter type:

```go
var _ observability.AuditLogsAdapter = (*LogsAdapter)(nil)

func (p *LogsAdapter) GetAuditLogs(
    ctx context.Context, params observability.AuditLogsParams,
) (*observability.AuditLogsResult, error)

func (p *LogsAdapter) GetAuditLogFilterValues(
    ctx context.Context, params observability.AuditLogFilterValuesParams,
) (*observability.AuditLogFilterValuesResult, error)
```

This reuses `logs.adapter.url` and follows what #4658 landed. Parent plan §5.1 records why it
revises D-C.

**Map statuses inline to per-signal sentinels**, as platform logs does at
`platform_logs_adapter.go:69-70`:

```go
var ErrAuditLogsNotSupported = errors.New("audit logs are not supported by the configured logs adapter")
var ErrAuditLogFilterValuesNotSupported = errors.New("audit log filter values are not supported by the configured logs adapter")
```

- `501` on either operation → its own not-supported sentinel. **Two sentinels, not one**: a module
  may serve records and decline the aggregation, and a client must be able to tell those apart.
- **No `410`.** It carried an expired cursor, which parent plan §C-3 removed.

Do **not** route any of this through `mapAdapterHTTPError` (`alerts_adapter.go:106`). That function
hard-codes `501 → ErrEventsNotImplemented` under a misleading name; platform logs bypassed it for
that reason, and generalising it is an unrelated cleanup this commit should not absorb.

**Mapping is where the rename bugs live**, which is why the filter groups are nested on both sides:
`params.Actor.IDs` → `body.Actor.Id`, field for field. Preserve nil vs empty on `timeline` — nil
means "the adapter could not compute it", which is not the same answer as an empty bucket list.

### 2.2 The service

`internal/observer/service/audit_logs.go`, modelled on `platform_logs.go`:

- `AuditLogsService` holding `observability.AuditLogsAdapter` and a `*slog.Logger`.
- `var _ AuditLogsQuerier = (*AuditLogsService)(nil)`.
- `QueryAuditLogs` parses `startTime`/`endTime` as RFC3339, calls the adapter, maps the result.
- `QueryAuditLogFilterValues` parses the nested `query`'s window and passes `filter`, `valueSearch`
  and `maxValues` through.
- Error discipline, copied exactly: pass the sentinels straight through; wrap anything else as
  `ErrAuditLogsRetrieval` and log it.

**Three genuine divergences from the template:**

- **Paging.** Platform logs has none; audit pages by time window, which the service does not
  implement — it maps `limit` and `sortOrder` through and the caller closes the window itself.
- **The timeline.** Only ever populated when the request asked for it; an adapter that computes one
  unconditionally must not make every caller carry it. Mirror
  `toTypesPlatformLogFacets`' nil-preserving shape.
- **Filter validation.** `filter` is a closed enum in the spec, so the generated model already
  constrains it — but the service maps it to a stored field path, and an unmapped value must be a
  `400`, not a silently empty result.

### 2.3 The handlers

Replace commit 1's `501` stubs in `handlers/auditlogs.go` with the real implementations, following
`platformlogs.go`:

- Add the `auditLogsService service.AuditLogsQuerier` field to `Handler`.
- Nil-service guard returning `500` + `ErrorCodeV1AuditLogsServiceNotReady` — **this is what keeps
  the endpoints inert until commit 3**, so it is load-bearing, not boilerplate.
- Request validation with `400` on cap violations, in `validations.go` beside
  `ValidatePlatformLogsQueryRequest`. Both operations validate the same filter set, so the record
  query's validator is reused for the nested `query`.
- `auditLogsError` mapping the sentinels: `ErrAuthzForbidden` → `403`, `ErrAuthzUnauthorized` →
  `401`, `ErrAuditLogsNotSupported` → `501`, otherwise `500` + `OBS-V1-AL-04`. Filter values needs
  its own mapping for
  `ErrAuditLogFilterValuesNotSupported` → `501` + `OBS-V1-AL-07`.

**No label-selector parsing.** The platform-logs handler parses a `labels` query parameter by hand;
a POST body arrives decoded, so that block has no counterpart. Do not port it.

**Also fix the `omitempty` fidelity bug here** (parent plan §7 item 4): drop `omitempty` from
`source_ip`, `request_id`, `user_agent` and `producer` on `types.AuditLogRecord`. The emitter writes
all four unconditionally (`eventJSON` gives them no `omitempty`), so keeping it would let the
observer silently drop a key from a record it claims to serve verbatim. It only becomes observable
once the read path serves real records, which is this commit.

### 2.4 Tests

Clone the `platform_logs_*_test.go` set:

| File | Covers |
|---|---|
| `audit_logs_adapter_test.go` | happy path both operations; `501` → each sentinel; a non-501 error asserting `NotErrorIs` the sentinels; nil timeline preserved |
| `audit_logs_test.go` | service over a stub adapter — mapping, time parsing, sentinel pass-through, timeline only when requested |
| `auditlogs_test.go` (handlers) | `400` caps, the nil-service `500`, the error-mapping table for both operations |
| `spec_conformance_test.go` | both response bodies against the spec — they route through the untyped `apiResponse`, so nothing else ties them to the contract |

### 2.5 Verification

```sh
make test GO_RUN_ARGS="./internal/observer/..."
make lint
```

The endpoints still answer `500 service not initialized` over HTTP. That is correct at this commit
and worth asserting in the handler test rather than treating as a gap.

---

## Commit 3 — `feat(api): gate audit log reads on auditlogs:view` · **done**

The action, the decorator, the Helm grants, and the wiring that makes the endpoints live.

### 3.1 Register the action — five sites, and a silent trap

**An action absent from `systemActions` expands to nothing and grants nothing, with no error
anywhere.** That trap is live in the tree: `portal-assistant:invoke` at
`install/helm/openchoreo-control-plane/values.yaml:1409` is dead config no CI check catches.

| # | File | Change |
|---|---|---|
| 1 | `internal/authz/core/actions.go` | `ActionViewAuditLogs = "auditlogs:view"`, beside `ActionViewPlatformLogs` |
| 2 | same file, `systemActions` | `{Name: ActionViewAuditLogs, LowestScope: ScopeCluster, IsInternal: false}` |
| 3 | `internal/observer/authz/constants.go` | `ActionViewAuditLogs Action = "auditlogs:view"` and `ResourceTypeAudit ResourceType = "audit"` |
| 4 | `internal/observer/service/audit_logs_authz.go` | the decorator — §3.2 |
| 5 | `install/helm/openchoreo-control-plane/values.yaml` | the grants — §3.3 |

`ScopeCluster` matches `platformlogs:view` and is what makes the empty-hierarchy check resolvable.

### 3.2 The decorator, and the one comment that is audit's own

`internal/observer/service/audit_logs_authz.go` is a near-verbatim copy of `platform_logs_authz.go`
— same struct shape, same `var _ AuditLogsQuerier` assertion, same `NewAuditLogsServiceWithAuthz`
constructor, same `CheckAuthorization` call with an **empty `authzcore.ResourceHierarchy{}`** —
except it wraps **both** methods. The filter-values read discloses the same content as the record
read, so it carries the same check; gating only one of them is a read-around.

What must not be copied verbatim is the comment. Platform logs justifies its empty hierarchy by
Kubernetes-vs-OpenChoreo namespace collision; audit's reason is different and more important:

```go
// An empty hierarchy is the cluster scope: resourceHierarchyToPath maps it to "*",
// which only a cluster-scoped binding can satisfy. Deliberately not derived from
// the request — an audit query's actor and resource fields are filters, not a
// scope. Deriving a hierarchy from them would let a namespace-scoped binding read
// that namespace's audit trail, which AX-4 exists to prevent: working in a project
// does not confer the right to read its audit trail.
```

**A tenancy filter must never widen authorization.** `resource.namespace: ["foo"]` narrows results;
it does not entitle the caller to `foo`'s trail.

One property that falls out for free: `CheckAuthorization` calls `audit.SetHierarchy` before the PDP
check (`internal/observer/authz/helpers.go:116-122`), and AX-5 already defines both operations, so
**denials are recorded too** with no extra work.

### 3.3 Who gets it

| Role | Grant | Why |
|---|---|---|
| `admin` | automatic via `actions: ["*"]` | no edit needed |
| `platform-engineer` | ✅ add | AX-4 names it. Add beside `platformlogs:view` |
| `sre` | ❌ | Holds broad observability reads, but AX-4 makes audit read a privilege in its own right |
| `rca-agent` | ❌ | Same, and an agent holding the audit trail is a larger blast radius than one holding logs |

> **AX-4's third role does not exist. Do not create it in this commit.** A role definition means
> deciding a full action list and an IdP mapping; that is a platform decision with a blast radius
> well beyond audit. Ship with two roles, record AX-4 as partially met, file the role separately.

Helm values changes require `make code.gen` to regenerate `values.schema.json`, or `code.gen-check`
fails.

### 3.4 Wire it — the commit that makes the endpoints live

`cmd/observer/main.go`, beside the platform-logs wiring:

```go
authzAuditLogsService := service.NewAuditLogsServiceWithAuthz(
	service.NewAuditLogsService(concreteLogsAdapter, logger.With("component", "audit-logs")),
	authzClient, logger.With("component", "authz-audit-logs"))
```

then add `authzAuditLogsService` to the `apihandler.NewHandler(...)` argument list, in the position
this commit adds to the signature.

**Constructed already wrapped** — the bare `NewAuditLogsService` result is never assigned to a
variable that reaches the handler, matching how `authzPlatformLogsService` nests its inner
constructor. The PDP argument is `authzClient`, not `pdp`.

The comment above that block already states the property this relies on: *"Both the API handler and
MCP handler share the same authz-wrapped instances so authorization logic is enforced once, in the
service layer."* That is why an MCP audit tool, if one is ever added, inherits the gate for free.

### 3.5 Tests

`audit_logs_authz_test.go`, cloning `platform_logs_authz_test.go`, plus **two cases the template does
not have and that are the point of this commit**:

- A subject whose only binding is namespace-scoped, querying with
  `resource.namespace: ["that-namespace"]`, is **denied**. This is the filter-is-not-a-scope
  property; without it, §3.2's comment is an unenforced assertion.
- The same denial on `QueryAuditLogFilterValues`. Gating one read and not the other would let a
  caller enumerate every actor and resource name in the trail.

Also: allowed for a cluster-scoped binding, `403` for no binding, `401` for no subject context, and
pass-through of the inner querier's results on success.

### 3.6 Verification

```sh
make code.gen && make lint-fix && make test && make code.gen-check
```

Then on k3d, with the observer pointed at any `observability-logs-*` module — until the module work
lands, that is all of them, so this is testable today:

```sh
curl -s -X POST -H "Authorization: Bearer $ADMIN_TOKEN" \
  http://localhost:11080/api/v1alpha1/audit-logs/query \
  -d '{"startTime":"…","endTime":"…"}'                       # → 501 "not supported by this adapter"

curl -s -X POST -H "Authorization: Bearer $ADMIN_TOKEN" \
  http://localhost:11080/api/v1alpha1/audit-logs/filter-values \
  -d '{"query":{"startTime":"…","endTime":"…"},"filter":"actor.id"}'   # → 501, code OBS-V1-AL-07

curl -s -X POST -H "Authorization: Bearer $DEV_TOKEN" \
  http://localhost:11080/api/v1alpha1/audit-logs/query \
  -d '{"startTime":"…","endTime":"…"}'                       # → 403, and NOT an empty 200
```

**The `501` is the success case here**, not a failure: it proves the whole chain — route, handler,
service, adapter, and AX-7 degradation — while the module side is still parked.

And because AX-5 is defined rather than deferred, the admin's own record query should appear in the
audit destination as `read_audit_log` under `category: access`, and the developer's `403` as the
same action with `result: denied`. The filter-values calls should appear **nowhere** — that one is
exempt on volume, so its absence from the trail is the expected result, not a wiring bug.

---

## 4. What these three commits deliberately leave

| Item | Where it goes |
|---|---|
| **`security-admin` role** | Separate platform change; §3.3 |
| Module query handlers (OpenSearch, then OpenObserve) | Parked with the rest of the community-modules work |
| MCP surface for audit queries | Not requested. The authz decorator wired in `main.go` is what would give MCP coverage for free when it is |
| Attribute filters (AT-9), collapse-on-`audit_id` (AX-2a) | Blocked on P3 and P10a; the schema leaves room and builds neither |
| Per-field substring matching (`field contains x`) | The UI's `op: 'has'` token. `searchPhrase` plus the exact `event_id`/`request_id` filters cover its stated motivation; a general `{field, op, values}` structure is a query language and a much larger contract |
| Generalising `mapAdapterHTTPError` | Unrelated cleanup; §2.1 |
| `METRICS_ADAPTER_URL` missing from `observer-config.yaml` | Pre-existing bug; this work does not touch that file |

---

## 5. Risks

| # | Risk | Handling |
|---|---|---|
| R-1 | **Commit 2 wired in `main.go` by reflex**, opening an ungated audit read | §0. The nil-service `500` is deliberate; a reviewer seeing "service not initialized" in commit 2 should read it as correct, not unfinished |
| R-2 | **Only the record read gets gated**, leaving filter-values as a read-around | §3.2 and §3.5's second denial test. Enumerating every actor and resource name is a disclosure of the trail even without the records |
| R-3 | **The authz comment copied verbatim from platform logs**, losing audit's filter-is-not-a-scope reasoning | §3.2, and §3.5's namespace-scoped denial tests make the property executable rather than aspirational |
| R-4 | **Retired.** Was "cursor semantics invented in the observer" — parent plan §C-3 dropped the cursor for time-window paging, so there are no cursor semantics anywhere |
| R-5 | **One not-supported sentinel for both operations**, so a module that serves records but cannot aggregate looks entirely unavailable | §2.1. Two sentinels, two error codes |
| R-6 | The nested filter mapping renames a field in the adapter copy | §2.1. Nesting on both sides makes it a field-for-field copy; the adapter test's happy path asserts the whole body |
