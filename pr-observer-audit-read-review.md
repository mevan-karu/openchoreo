# Review — audit read path on the observer

**Scope:** `596f7e464` (contract + service) and `76203033a` (authorization), reviewed against
[`openchoreo-audit-observer-api-plan.md`](./openchoreo-audit-observer-api-plan.md) and
[`openchoreo-audit-observer-impl-plan.md`](./openchoreo-audit-observer-impl-plan.md).
**Verified:** `go test ./internal/observer/... ./tools/auditgen/...` — all green.

---

## What the plans got right, confirmed

**The §0 / R-1 invariant holds.** This is the highest-stakes property in the impl plan and it is
intact: `cmd/observer/main.go` is touched only in `76203033a`, in the same commit that adds the
decorator and the Helm grant. There is no commit in the branch's history where the endpoints are
reachable over HTTP and ungated. The filing changed — the plan's commits 1 and 2 landed together as
`596f7e464` — but that does not weaken the property, because the merge was of contract into service,
not of service into wiring.

**§7 item 4 (`omitempty` fidelity) is resolved.** `types.AuditLogRecord` gives `request_id`,
`source_ip`, `user_agent` and `producer` no `omitempty`, with a comment explaining why
(`internal/observer/types/auditlogs.go:151-156`). The tags remaining on
`pkg/observability.AuditLogRecord` are correct — that is the internal domain type, not the served
shape.

**The grant landed where §3.3 said.** `auditlogs:view` appears exactly once in
`install/helm/openchoreo-control-plane/values.yaml:1612`, inside the `platform-engineer` block that
opens at `:1513`. `sre` (`:1456`) and `rca-agent` are excluded, as decided.

**`values.schema.json` needs no regeneration.** The role `actions` field is a free-form
`array of string` (`values.schema.json:3718-3724`), so `make code.gen-check` will not fail on the
one-line grant, contrary to impl plan §3.3's warning.

**Test coverage is genuinely good** — adapter status mapping, the two distinct not-supported
sentinels, nil-timeline preservation, the nil-service 500, the cluster-scope assertion, and both
tenancy-filter-is-not-a-scope denial cases (§3.5's two additions) are all present and executable.

---

## Findings, most severe first

### 1. The `category` filter enum excludes `access`, the category this change introduces

Both specs still declare the filter as a two-value closed set:

- `openapi/observer-api.yaml:1572` — `maxItems: 2`, `:1578` — `enum: ["management", "authorization"]`
- `openapi/observability-logs-adapter-api.yaml:975` / `:981` — the same
- record-side descriptions: `openapi/observer-api.yaml:1996` and
  `openapi/observability-logs-adapter-api.yaml:1318` — *"Event category. `management` or
  `authorization` at schema 1.0."*

Meanwhile the same commit adds `CategoryAccess = "access"`
(`internal/server/middleware/audit/types.go`) and emits `QueryAuditLogs` records under it
(`internal/observer/audit/definitions.gen.go:19-22`). Querying the trail now produces records the
trail's own `category` filter cannot name.

There is **no runtime 400** — I grepped for `OapiRequestValidator` and the observer does not mount
one, and the hand-written validator at `internal/observer/api/handlers/validations.go:363` already
accepts `"access"`. So the break is entirely at the contract boundary, which is worse in this case
rather than better:

- The generated `AuditLogsQueryRequestCategory` constants (`internal/observer/api/gen/models.gen.go:77-80`)
  have only `Authorization` and `Management`. No generated client can express the filter.
- A community module validating its request body against the adapter YAML will reject a forwarded
  `category: ["access"]` — and `auditLogsRequestBody` will happily send it, since the observer's own
  validator waved it through.
- `maxItems: 2` means the full vocabulary cannot be requested even once `access` is added to the
  enum.

This defeats the reason the category was created. API plan §4: `CategoryAccess` exists so `category`
works *"as a filter for telling disclosure apart from change, which is one of AX-2's required
axes."* The filter that makes that axis usable does not accept the value.

**Fix — six edits, all mechanical:** the two `enum` lines, the two `maxItems: 2` → `3`, and the two
record-schema descriptions.

### 2. Audit queries inherit the logs endpoints' 30-day window cap

`ValidateAuditLogsQueryRequest` calls `ValidateTimeRange`
(`internal/observer/api/handlers/validations.go:194`), which enforces
`maxQueryTimeRange = 30 * 24 * time.Hour` (`:25`) — a constant shared with the component-logs and
platform-logs endpoints.

This is not a taste question; it contradicts the contract in the same PR. The observer spec's own
description of the endpoint says the audit trail is *"served from the audit destination, which has
its own retention independent of the log retention behind the other endpoints."* The implementation
then bounds it with the log-oriented constant. A compliance query over a quarter — the ordinary case
for an audit trail — has to be hand-paged into four windows, and nothing in either plan discusses
the cap.

**Fix:** give audit its own `maxAuditLogsTimeRange`, or state in the spec that queries are windowed
at 30 days and why. Either is fine; silently inheriting a log constant is not.

### 3. No `spec_conformance_test.go` case for either audit response

Impl plan §2.4 names the file and API plan §8 repeats the requirement. `TestResponsesConformToSpec`
(`internal/observer/api/handlers/spec_conformance_test.go:106`) has no audit case — I grepped the
file for `audit` and got nothing.

The file's own doc comment states exactly why this matters: *"handlers return `types.*` and `gen.*`
values through the untyped `apiResponse` passthrough, so nothing else ties a response body to the
contract."* Both audit handlers return `jsonResponse(http.StatusOK, result)` with a hand-written
`types.*` struct. Nothing in the build or the test suite currently checks that
`types.AuditLogsResponse` serialises to something `AuditLogsResponse` accepts — and that is a
hand-maintained struct of ~20 fields with a deliberate snake_case/camelCase split, which is the
highest-risk shape in the change.

### 4. Adapter response values reach the wire unvalidated, and can violate the observer's own contract

`AuditLogsResponse.totalRelation` is `required` with `enum: [eq, gte]`
(`openapi/observer-api.yaml`, `AuditLogsResponse`). The observer forwards whatever the adapter said:

```go
TotalRelation: string(result.TotalRelation),   // internal/observer/service/audit_logs.go:58, :100
```

A module that omits the field, or spells it `"exact"`, turns into an observer response that breaks
the observer's published contract — and the observer is the trusted side of that boundary. The same
applies to `nextCursor` (`maxLength: 4096` in the spec, passed through unbounded) and to the
`AuditLogRecord` required fields.

Given the whole phase is designed around *"an empty success is indistinguishable from nothing
happened"*, silently emitting a malformed `totalRelation` is the same class of mistake. A default
to `gte` on an unrecognised value, or a `500` with the retrieval code, is a few lines and keeps the
observer's output honest regardless of module quality.

### 5. Four spec enums are mirrored by hand in Go, with nothing enforcing the mirror

`internal/observer/api/handlers/validations.go:340` (`auditLogsFilterPaths`, 19 entries) and `:363`
(`auditLogCategories` / `auditLogResults` / `auditLogSurfaces`) duplicate spec enums. The comment on
the first asserts *"Kept in sync with the `filter` enum in openapi/observer-api.yaml"* — an
invariant with no mechanism behind it.

This is the root cause of finding 1: the Go side gained `access` and the spec did not, and nothing
noticed. It is also cheap to close, because codegen already emits what the validators need:
`AuditLogFilterValuesRequestFilter` has all 19 path constants (`gen/models.gen.go:48-68`), and the
three enum types are generated too. Either range over the generated constants, or add a test that
cross-checks the maps against `gen.GetSwagger()`. The second is less invasive and would have caught
this.

### 6. The collector cross-check was shipped as a comment, not as behaviour

API plan §7 item 3 said of the `producer` / collector mismatch: *"**Flag and return** rather than
drop; decide before the fields become decorative."* They are now decorative.
`AuditLogCollectorInfo` is threaded through all three layers, and its doc comment in
`pkg/observability/auditlogs.go` states *"It cross-checks Producer: the two disagreeing means a
record's claimed origin and its actual one differ"* — but nothing anywhere performs that comparison,
and no response field reports it.

A doc comment describing behaviour the code does not have is worse than an absent feature: the next
reader assumes the check exists. Either implement the comparison and surface it, or reword the
comment to say the fields are carried for a client to compare.

### 7. `maxValues` defaults from `defaultLimit`, recoupling what C-5 decoupled

```go
req.MaxValues = defaultLimit   // internal/observer/api/handlers/validations.go:467
```

`defaultLimit` is the shared record page-size constant. The spec argues the opposite case in its own
words: *"Named `maxValues` rather than `limit` to keep it distinct from `query.limit`, which is a
page size for records and is ignored here."* Bump `defaultLimit` for log paging some day and the
filter-values default moves with it, silently.

`defaultAuditLogMaxValues = 100` is one line and matches the spec's declared `default: 100`.

### 8. Two stale comments

- `internal/observer/api/handlers/validations.go:322-325` — a doc comment for
  `maxAuditLogsTimelineBuckets`, a constant that does not exist. It sits above
  `auditLogsTimelineIntervalPattern`, so it reads as documenting the wrong thing.
- `internal/observer/api/handlers/audit_wiring_test.go:102` — *"drives the one audited operation"*.
  `QueryAuditLogs` is now a second one, and it has no wiring test.

On that second point: an end-to-end test mirroring `TestUpdateIncidentAuditEvent` — driving
`QueryAuditLogs` through the composed chain and asserting a `read_audit_log` event lands with
`category: access` and a real `actor.id` — would make AX-5 executable instead of a manual `curl`
step in §8. Given that "querying the trail appends to it" is the headline claim of §4, it is worth
the ~40 lines.

### Nit

`ErrorCodeV1AuditLogsAuthzInternal` (`OBS-V1-AL-02`) is declared and never used — the 401/403 paths
pass an empty code. Worth noting only because it mirrors `ErrorCodeV1PlatformLogsAuthzInternal`
(`OBS-V1-CL-02`), which is equally unused; this is precedent-following, not a defect introduced here.

---

## Suggested order

1. Finding 1 — contract bug, blocks any client from using the category axis.
2. Finding 5 — add the sync test, so 1 cannot recur.
3. Finding 3 — the conformance cases.
4. Findings 2, 4 — decide the window cap and the pass-through hardening.
5. Findings 6, 7, 8 — comment and naming cleanups.
