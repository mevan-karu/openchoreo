# Audit Read Path — Observer API Implementation Plan

**Parent:** [`openchoreo-audit-delivery-plan.md`](./openchoreo-audit-delivery-plan.md) §2. This is the
per-task breakdown of the **read path** — contract, Observer, authorization.
**Sibling:** [`openchoreo-audit-publishing-plan.md`](./openchoreo-audit-publishing-plan.md) §1, the publishing half.
**Phase:** P7 · **Status:** contract and observer implemented, none pushed · **Date:** 2026-09-11
**Repo:** `openchoreo` only.

> **Community-module work is parked.** Phase E of the delivery plan — the OpenSearch and OpenObserve
> query handlers — is deliberately out of scope here, and so is the collection work in
> `openchoreo-audit-publishing-plan.md` §CM. This plan lands the contract and the Observer side
> **against an adapter that answers `501`**, which is a complete, shippable, testable state: the
> endpoints exist, are permission-gated, and degrade intelligibly.
>
> The consequence to accept openly: **until a module implements the queries, this delivers a working
> API that returns "no audit backend configured".** The value is that the contract is frozen before
> anyone implements against it, and the portal (P8) can start.

---

## 0. What changed, and why

Nine decisions moved after this plan was first written. Each is recorded here so none gets
re-litigated from the older text.

| Then | Now |
|---|---|
| #4658 *"in review"*, naming unsettled | **Merged.** Naming settled as `platform-logs`, action `platformlogs:view`. Audit mirrors the noun, not the verb |
| §2.3: *"a fifth adapter pair in `AdaptersConfig`"* — a new `audit.adapter.url` | **D-C revised.** `GetAuditLogs` is a method on the existing `*LogsAdapter`, reusing `logs.adapter.url`, as `GetPlatformLogs` is. §5.1 |
| §2.2: *"generalise `mapAdapterHTTPError`"* | **Moot.** Audit checks status inline and returns per-signal sentinels, as platform logs does |
| AX-5 *"a new pattern rather than one more entry"* — and the POST-vs-GET argument built on it | **Both obsolete.** `7140fa08f` (#4703) made read classification explicit: an unclassified GET now *fails generation*, and an `Overrides` entry can audit a GET. The gate is verb-neutral, so "POST makes AX-5 free" is no longer a reason for anything. POST still wins on request shape — §C-2 |
| The record carries `origin: api\|mcp` | **`surface: rest\|mcp`**, and a new always-written `user_agent` (`69e77b9e4`, #4706). MCP wraps the same API, so the REST value is not `api` |
| Filters are flat and renamed — `actorId`, `namespace`, `resourceType` | **Filters mirror the record.** Nested `actor` / `resource` groups, named exactly as the record names them. §C-2 |
| §C-4: *"add the new spec output to `make/golang.mk`"* | **Moot.** Both specs are already wired at `golang.mk:196-204`; this adds paths to existing specs, not a spec file |
| Facets: return every filter's values on the record query | **Rejected on load.** One aggregation per filter, recomputed on every query change, is enough to exhaust a backend. Replaced by a dedicated operation — §C-5 |
| AX-5 deferred to a follow-up PR | **Done in the contract commit.** The record read is *defined*, not exempted. The filter-values read is exempt on volume — a picker fires on every query change — and says so — §4 |
| `auditlog:view` | **`auditlogs:view`.** Every observability signal action is plural — `logs:view`, `platformlogs:view`, `events:view`, `traces:view`, `alerts:view`. The singular ones are CRD resources |

**One thing the original plan got right and worth restating**, because it is the highest-consequence
line in the phase: every new adapter operation **must declare `501` from day one**, or every existing
`observability-logs-*` module becomes contractually obliged to implement audit on its next unrelated
regeneration.

---

## 1. The contract — landed

Two specs, one commit. Modules implement against this, so it is frozen before §2 continues.

### C-1 — `POST /api/v1alpha1/audit-logs/query` on the adapter contract

**File.** `openapi/observability-logs-adapter-api.yaml`.

- **Declares `501`.** Non-negotiable, per above.
- **No `410`.** It existed for an expired cursor; §C-3 removed the cursor.
- Request `AuditLogsQueryRequest`, response `AuditLogsResponse`.
- The description states that a module which does not implement it must answer `501` rather than an
  empty `200` — an empty success is indistinguishable from "nothing happened", the one answer an
  audit API must never fabricate — and that a forward-only deployment reaches `501` too.

### C-2 — `POST /api/v1alpha1/audit-logs/query` on the observer contract

**File.** `openapi/observer-api.yaml`. Modelled on the observer's eight existing read-as-POST
endpoints rather than on `GET /platform-logs`.

> **POST, not GET.** Read the observer's endpoints by request *shape*, not verb count: all eight
> `POST …/query` endpoints are filter-body reads; the four GETs are path-addressed reports
> (`{namespace}/{environment}`, `{traceId}/{spanId}`). `platform-logs` is the only filter-query on a
> GET, and this plan's own text called it the outlier. Audit's filter set is the POST shape.
>
> **The audit argument for POST is dead and must not be repeated.** It rested on `auditgen` skipping
> every GET before the `Overrides` check. Since #4703 the order is exclusions → GET-requires-override
> → derive, so a GET can be audited and an unclassified one fails generation. Surviving reasons:
> request shape, a filter set that grows, symmetry with the adapter.
>
> **What this gives up:** no bookmarkable API URL, the reason #4658 gave for GET. It costs P8 nothing
> — the portal syncs filter state to its own route, so its deep-linking never depended on the verb.

**Request — shaped like the record it filters.** Every filter is named and nested exactly as the
field it matches in `AuditLogRecord`, so a client needs no translation table and a picker built from
the record's field paths maps on mechanically. This is the change the UI prototype forced: its whole
vocabulary is record field paths (`COLUMNS` at `openchoreo-audit-logs-prototype.html:1472` — *"Columns
name the field path they render, so the table header and the filter vocabulary stay the same words"*).

| Group | Filters |
|---|---|
| `actor` | `id`, `type`, `issuer`, `session_id`, `entitlements` |
| `resource` | `type`, `namespace`, `environment`, `project`, `component`, `name` |
| top level | `action`, `category`, `result`, `producer`, `surface`, `operation_id`, `request_id`, `event_id`, `source_ip`, `user_agent` |
| envelope | `startTime`, `endTime` (required), `searchPhrase`, `limit`, `sortOrder`, `includeTimeline`, `timelineInterval` |

**The casing rule is deliberate and documented in the schema:** record-derived filters keep the
record's snake_case (`operation_id`, `source_ip`); the query's own controls stay camelCase
(`startTime`, `includeTimeline`). The casing marks which of the two a field is. Without that
sentence a reader hitting `event_id` beside `startTime` reads it as sloppiness.

Field-level decisions, each recorded rather than implied:

- **`actor.issuer` exists because `actor.id` alone is wrong.** The record's own description says an
  `id` is unique only within an `issuer`. Filtering on `id` across two identity providers conflates
  two subjects — on the endpoint whose job is attributing actions to people. This is a correctness
  filter, not a convenience one.
- **`actor.entitlements` matches across claim keys.** The key varies by subject kind — `groups` for
  a user, `sub` for a service account — and a caller asking "what did everyone in
  `platform-engineer` do" should not have to know which.
- **No `resource.uid`.** `uid` is absent on deletes and non-CRUD mutations, so a UID filter silently
  excludes exactly the operations an investigator most often wants. Filter by name, read `uid` off
  the record.
- **No `resource.resource`.** The fourth hierarchy level is populated only by the `Resource` CRD
  service, where it equals `resource.name` (`services/resource/service_authz.go:118-122`), so a
  filter would be a second spelling of one already present.
- **`resource.environment` takes the dual-scoped `{namespace}/{name}` form**, because that is how
  the value is stored — recorded exactly as authorization evaluated it. Said in the parameter
  description, or every consumer gets it wrong once.
- **`maxItems` on every array**, enforced with a `400`. A body removes the URL-length ceiling, not
  the reason to bound a filter set.
- **No attribute filters, no `audit_id` collapse.** AT-9 needs P3's declared attributes and AX-2a
  needs P10a's `audit_id`; neither exists. The schema leaves room and builds neither.

Responses: `200`, `400`, `401`, `403`, `500`, `501`.

### C-3 — Time-window paging, as on the other log endpoints

**Reversed: there is no cursor.** The earlier design had the module mint an opaque continuation
token, with a `410`/`OBS-V1-AL-06` path for expiry. Two things killed it:

- **Some observability backends have no cursor concept at all**, and the contract made `nextCursor`
  optional — so a conforming module that cannot mint one returns page 1 with the field absent, which
  this contract defines as "last page". A cursor-less backend would therefore report the first page
  as the whole trail. The optionality read as a graceful escape and was really a trap.
- It asked every module for point-in-time support to serve a read that `/logs` and `/platform-logs`
  serve with a time window and nothing else.

**Paging is by window.** A page is at most `limit` records ordered by `event_time`; a caller
continues by closing the window up to the last record it received — `endTime` down when descending,
`startTime` up when ascending. No token, no expiry, no per-scroll state in a module.

AX-2 still holds: window continuation is pagination, and stable ordering comes from sorting rather
than from a token.

- **The accepted cost.** `endTime` is exclusive, so records sharing the boundary's exact
  `event_time` beyond a page's `limit` are not reached by the next query. Documented on the
  operation rather than left for each client to rediscover; the remedy is to narrow the window.
  Option B in the P7 discussion — keyset on `(event_time, event_id)` — closes this and remains
  available additively if boundary ties ever bite.
- **`totalRelation: eq | gte`**, so a capped count stops silently claiming to be exact, and a client
  can tell whether a window holds more than one page.

The UI still fits: the prototype's table appends with "Load N more" rather than jumping to page N
(`:2225-2230`), which a window walk serves as well as a cursor did, and it renders
"Showing 30 of 1,204" from `total` — which is what `totalRelation` makes honest.

### C-4 — Timeline

`includeTimeline` plus `timelineInterval` (`^[1-9][0-9]*[mhdw]$` — this spec's existing
`<count><unit>` convention, extended with `m` because a one-hour window wants minute buckets).

Response `timeline` carries `interval` (**the width actually used**) and `buckets`, each
`{startTime, total, counts}` where `counts` is an open map keyed by `result`.

Four properties that are requirements, not implementation details:

- **No page of records can be bucketed into a timeline.** A page is the newest or oldest `limit`
  records, not a sample of the window, so a client that wants a histogram must ask for one. The
  prototype buckets client-side over *every* matching record (`:2000-2016`), which only works
  because it holds the whole dataset in memory.
- **Empty buckets are present with zero counts.** A sparse array lets a client draw a continuous
  chart across a gap in activity — the same class of mistake as answering an unimplemented query
  with an empty `200`.
- **A width exceeding 500 buckets is coarsened, not rejected**, and `interval` reports the truth.
  `1m` over `30d` is 43,200 buckets; an adapter with no stated ceiling invents its own silently.
- **`counts` is an open map, absent key means zero**, so a `result` value added in a later schema
  needs no spec bump.

### C-5 — `POST /api/v1alpha1/audit-logs/filter-values`

**Facets were rejected and this replaces them.** Returning every filter's values on the record query
means one aggregation per filter rather than one in total, recomputed on every query change — enough
load to exhaust a backend on a busy trail. A dedicated operation keeps the cost proportional to what
a client actually needs.

- **One filter per request**, named by its path in the query vocabulary. This matches the UI's real
  access pattern: `parseDraft()` resolves to a single attribute and `suggestions()` opens one picker
  at a time.
- **`query` is the whole `AuditLogsQueryRequest`, nested.** A client already holds one query state
  and asks what values a field takes under it. `startTime`, `endTime` and `searchPhrase` are
  honoured; `limit`, `sortOrder`, `includeTimeline`, `timelineInterval` are ignored.
- **The cap is `maxValues`, not `limit`**, so it cannot be confused with `query.limit` — a record
  page size. A client setting `query.limit: 1000` hoping for more values would otherwise get nothing
  with no error.
- **Self-exclusion applies to the named filter only, not its group.** Values come back under the
  query's *other* filters, so selecting a namespace narrows the actions on offer, while the named
  filter's own selections are ignored — otherwise a picker mirrors the selection instead of offering
  the alternatives to it. Nesting invites group-level exclusion, so the schema says this explicitly.
- **`valueSearch` narrows the values; `query.searchPhrase` narrows the records.** The distinction is
  what makes a high-cardinality filter usable — the prototype's substring row (`:1683-1685`) exists
  for exactly this.
- **`totalValues` + `totalRelation`**, not a boolean. The prototype renders *"N more values — keep
  typing to narrow"* (`:1707-1709`), which needs a number, and counting distinct values exactly is
  itself expensive.
- **`actor.session_id` is offered; `event_id` and `request_id` are not.** A near-unique field becomes
  usable once `valueSearch` exists — an investigator narrowing to a session has a prefix to type.
  Event and request IDs are filtered by an exact value the client already holds, from a log line or
  another record.
- **`501` is separately declinable**, with its own code `OBS-V1-AL-07`. A module can serve records
  before it can aggregate, and a client seeing this should fall back to free-text entry on that
  filter rather than conclude the trail is unreadable.

### C-6 — Codegen wiring

Nothing to add: both specs are already inputs to `make/golang.mk`'s `openapi-codegen`
(`:196-204`). `make code.gen` regenerates `internal/observer/api/gen/` and `logsadapterclientgen/`.

---

## 2. The Observer

`platform_logs` is the template for every file below; read it first, the shape is the specification.
Divergences are called out in the impl plan.

| File | Clone of | Contents |
|---|---|---|
| `internal/observer/types/auditlogs.go` ✅ | `types/platformlogs.go` | request/record/response/timeline/filter-value types |
| `pkg/observability/auditlogs.go` ✅ | `pkg/observability/logs.go:142` | `AuditLogsAdapter` with **two** methods, params/result structs |
| `internal/observer/service/audit_logs_adapter.go` | `platform_logs_adapter.go` | both methods on the existing `*LogsAdapter`, with inline status→sentinel checks |
| `internal/observer/service/audit_logs.go` | `platform_logs.go` | `AuditLogsService`, mapping and sentinel translation |
| `internal/observer/service/audit_logs_authz.go` | `platform_logs_authz.go` | the decorator — §3 |
| `internal/observer/api/handlers/auditlogs.go` ✅ stub | `platformlogs.go` | both handlers plus `auditLogsError` |

Plus edits to five existing files: `internal/observer/service/interfaces.go` ✅, `.mockery.yaml` ✅,
`internal/observer/api/handlers/handler.go`, `cmd/observer/main.go`,
`internal/observer/api/handlers/response.go` ✅.

**The generated server is the safety net.** Since P1.5a the observer serves both surfaces through the
generated strict handler, so a spec path with no method **fails the build**. The inherited worry
about hand-registered routes and silent spec drift does not apply, and P7 tasks 7.11/7.12 are struck.

---

## 3. Authorization

`auditlogs:view`, cluster-scoped, registered exactly as `platformlogs:view` was — and gating **both**
operations, because the filter-values read enumerates actors, resource names and source addresses
from the trail. It discloses the trail whether or not records come with it, so a caller who cannot
read the records must not be able to read that either.

**Five edit sites** — and an action absent from `systemActions` **expands to nothing and grants
nothing, silently**. That trap is live in the tree: `portal-assistant:invoke` at
`install/helm/openchoreo-control-plane/values.yaml:1409` is dead config no CI check catches.

1. `internal/authz/core/actions.go` — the const, beside `ActionViewPlatformLogs`.
2. The same file's `systemActions` table — `{Name: ActionViewAuditLogs, LowestScope: ScopeCluster, IsInternal: false}`.
3. `internal/observer/authz/constants.go` — the mirror const and a `ResourceTypeAudit`.
4. `internal/observer/service/audit_logs_authz.go` — the decorator.
5. `install/helm/openchoreo-control-plane/values.yaml` — the bootstrap role grants.

**The decorator's comment is the load-bearing part, and it is audit's own, not platform logs':**

```go
// An empty hierarchy is the cluster scope: resourceHierarchyToPath maps it to "*",
// which only a cluster-scoped binding can satisfy. Deliberately not derived from
// the request — an audit query's actor and resource fields are filters, not a
// scope. Deriving a hierarchy from them would let a namespace-scoped binding read
// that namespace's audit trail, which AX-4 exists to prevent: working in a project
// does not confer the right to read its audit trail.
```

**A tenancy filter must never widen authorization.** `resource.namespace: ["foo"]` narrows results;
it does not entitle the caller to `foo`'s audit trail.

### Who gets it

| Role | `platformlogs:view` | `auditlogs:view` | Why |
|---|---|---|---|
| `admin` | via `*` | via `*` | automatic |
| `platform-engineer` | ✅ | ✅ | AX-4 names it |
| `sre` | — | ❌ | Holds broad observability reads, but AX-4 makes audit read a privilege in its own right — the trail discloses activity across every team |
| `rca-agent` | — | ❌ | Same, and an agent with the audit trail is a larger blast radius than an agent with logs |

> **⚠️ AX-4's third role does not exist, and this plan does not create it.** The bootstrap roles are
> `admin`, `platform-engineer`, `sre`, `developer`, `cluster-reader`, `namespace-reader` and five
> service identities — **there is no `security-admin`**.
>
> Creating one is not a grant, it is a role definition: someone has to decide its whole action list
> and its IdP mapping, a platform decision with a blast radius well beyond audit. **So phase 1 grants
> `admin` + `platform-engineer` and records AX-4 as partially met**, with `security-admin` filed
> separately. Adding the action to a new role later is one line; inventing the role inside an audit
> PR is how a half-considered role definition ends up permanent.

---

## 4. AX-5 — auditing the audit read · **done**

The record read is **defined, not exempted**. Querying the trail appends to it.

```go
// tools/auditgen/config_observer.go
{ID: "QueryAuditLogs", Action: "read_audit_log", ResourceType: "auditlog",
 Category: "CategoryAccess"},
```

It needs an override rather than derivation for two independent reasons: `deriveDefinition` has no
verb for a read — it maps POST to `create_` and then rejects the mismatch against the operationId's
leading word — and the resource-kind segment it would parse from this path is `query`, which names
no resource. An override replaces the derivation wholesale and skips the verb-consistency check
with it.

**`QueryAuditLogFilterValues` is exempt, and the difference is volume rather than sensitivity.** A
filter picker populates on every interaction that changes a query, so auditing it would bury the
record reads it leads to — and those are audited, which is where the disclosure that matters is
already captured. The values it returns are the trail's vocabulary, not its contents. The same
`auditlogs:view` permission still gates it, so the exemption is about what gets recorded, never about
who may call it.

**Two vocabulary additions, both new:**

- **verb `read_`** — every other action is a mutation verb.
- **`CategoryAccess`** (`internal/server/middleware/audit/types.go`) — the other two categories mean
  state change and authorization change. A read of the trail is neither, and filing it under
  `management` would make `category` useless as a filter for telling disclosure apart from change,
  which is one of AX-2's required axes.

`ResourceType` is `auditlog` — singular, as every audit `ResourceType` is (`project`, `incident`,
`component`), where the authz action noun is plural to match its siblings. Each follows its own
convention rather than one following the other. `action` stays 1:1 with `operation_id` as it is
everywhere else — a property the UI prototype documents (`FIELD_ANNOTATIONS`: *"1:1 with
operation_id today"*).

`internal/observer/audit/audit_coverage_test.go`'s category switch admits `CategoryAccess`;
openchoreo-api's stays at two, so a read-audit operation added there fails with a clear message
rather than landing silently.

One consequence for the schema reference: **the read event is published to the audit destination**,
so querying the trail appends to it. Intended, and P0's policy vocabulary can suppress it if volume
becomes a problem. `CheckAuthorization` calls `audit.SetHierarchy` before the PDP check
(`internal/observer/authz/helpers.go:116-122`), so **denials are recorded too**, with no extra work.

---

## 5. Configuration and degradation

### 5.1 Adapter URL — follow the precedent, revising D-C

**D-C is revised: reuse `logs.adapter.url`, as platform logs does.** The evidence changed —
`GetPlatformLogs` is a method on `*LogsAdapter`, and audit's queries live on the *same adapter
contract*, served by the same module process.

- A separate key with only one legal value is inert configuration, and this repo has an explicit
  convention against that (`policy.go:33-36`).
- Diverging from a precedent that landed the same week costs a reader an explanation for no
  behaviour.
- **Adding it later is additive**, not breaking: `audit.adapter.url` defaulting to `logs.adapter.url`
  is backward-compatible whenever someone actually needs the split.

> *Unrelated but adjacent:* `METRICS_ADAPTER_URL` is **absent** from
> `install/helm/openchoreo-observability-plane/templates/observer/observer-config.yaml` although the
> config requires it, so it silently falls back to its default. Pre-existing; this plan does not
> touch that file.

### 5.2 AX-7 — degrade intelligibly

| State | Response |
|---|---|
| Module implements the query | `200` with records |
| Module answers `501` on records | `501`, *"audit logs are not supported by this adapter"* |
| Module serves records but cannot aggregate | `501` on filter-values only, code `OBS-V1-AL-07`; records keep working |
| Window holds more than one page | `total` + `totalRelation`; the caller narrows the window to continue |
| Adapter unreachable | `500` with a retrieval code, **never** an empty `200` |

This is also how scope §4.2's ST-6-vs-AX-1 contradiction resolves: **forward-only is an explicitly
read-path-disabled configuration**, recorded rather than discovered. That sentence is in the `501`
description, so the contradiction is answered where a reader meets it.

---

## 6. Breakdown

| # | Title | Contents | State |
|---|---|---|---|
| **1** | `feat(api): define the audit logs query contract on observer and adapter` | C-1…C-6, both operations, timeline, AX-5 definitions, regenerated models/server/clients; observer answers `501` | **landed** |
| **2** | `feat(api): implement the audit logs queries in observer` | §2's service/adapter/handler and tests | **done** |
| **3** | `feat(api): gate audit log reads on auditlogs:view` | §3's five edit sites, decorator tests, Helm grants, `main.go` wiring | **done** |

PRs 2 and 3 stay separate so the grant list is a reviewable decision rather than a diff detail —
AX-4's exclusions (`sre`, `rca-agent`) are exactly the kind of call that should be visible.

---

## 7. Open decisions

| # | Question | Recommendation |
|---|---|---|
| 1 | AX-4 names `security-admin`, which **does not exist** as a bootstrap role | **Ship without it**, record AX-4 as partially met, file it separately |
| 2 | `sre` and `rca-agent` — in or out? | **Out**, stated explicitly in the PR so it is a decision rather than an omission |
| 3 | Does the read path surface a `producer` / `kubernetes.container_name` mismatch? | Publishing-plan §1.3.2 indexes both. The record carries `collector` for it. **Flag and return** rather than drop; decide before the fields become decorative |
| 4 | `types.AuditLogRecord` has `omitempty` on four fields the emitter always writes | **Fix in the implementation.** `eventJSON` gives `source_ip`, `request_id`, `user_agent` and `producer` no `omitempty`, so a module returning `source_ip: ""` would have the key dropped — contradicting the verbatim-record claim in that struct's own doc comment |

---

## 8. Verification

**PR 1** — landed. `make code.gen && make lint && make test` clean; the generated
`StrictServerInterface` gained `QueryAuditLogs` and `QueryAuditLogFilterValues`, and the build failed
until both handlers existed. That build failure *is* the test that the routes are wired. The audit
coverage gate passes with both operations **defined**; the pinned operation count is 22.

**PR 2** — clone the `platform_logs_*_test.go` set: adapter tests for the happy path, `501` →
sentinel, and a non-501 error asserting `NotErrorIs` the sentinel; service
tests over a stub adapter; handler tests for the `400` caps and the nil-service `500`. Add a
`spec_conformance_test.go` case, since both responses route through the untyped `apiResponse`.

**PR 3** —

```sh
curl -s -X POST -H "Authorization: Bearer $ADMIN_TOKEN" \
  http://localhost:11080/api/v1alpha1/audit-logs/query \
  -d '{"startTime":"…","endTime":"…","result":["denied"]}' | jq
curl -s -X POST -H "Authorization: Bearer $DEV_TOKEN" \
  '…/audit-logs/query' -d '{…}'                                  # → 403, not an empty 200
```

Three assertions that are easy to omit and are the point of the phase:

- **A namespace-scoped binding does not satisfy either query**, even with `resource.namespace` in the
  body set to the namespace it holds. The decorator's empty hierarchy enforces this, and it is the
  difference between a filter and a scope.
- **Filter-values is gated too.** A caller denied the records gets `403` here as well, not a list of
  every actor in the trail.
- **A module answering `501` produces `501` with the "not supported" message**, not an empty `200`.
  Point the observer at any `observability-logs-*` module that has not implemented audit — until
  Phase E, that is all of them, so this is testable today.

**And the read appears in the audit destination**, since AX-5 is defined rather than deferred:
querying the trail emits `read_audit_log` under `category: access`.

Before every PR: `make code.gen && make lint-fix && make test && make code.gen-check`.
