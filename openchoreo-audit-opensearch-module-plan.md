# `observability-logs-opensearch` — Audit Log Storage and Retrieval

**Repo:** `openchoreo/community-modules`, module `observability-logs-opensearch`
**Parents:** [`openchoreo-audit-publishing-plan.md`](./openchoreo-audit-publishing-plan.md) §2.1 (collection,
destination, retention) and [`openchoreo-audit-delivery-plan.md`](./openchoreo-audit-delivery-plan.md) §2.5
(the read path). This plan supersedes both **for this module**.
**Status:** Draft for review · **Date:** 2026-09-11

---

## 0. The base, the gate, and why this is two phases

### 0.1 Base on PR 606, do not branch from `main`

[`community-modules#606`](https://github.com/openchoreo/community-modules/pull/606) — *"feat: add platform
logs endpoint to observability-logs-opensearch module"*, `akila-i:platform-logs` → `main`, **open**,
`MERGEABLE`, 7 commits — is the base for everything below. It is not merely adjacent work: it touches
**every file this plan touches except the adapter configmap**, and it resolves what was this plan's open
scope question.

| PR 606 changes | Consequence here |
|---|---|
| `internal/platform_handlers.go`, `internal/opensearch/platform_queries.go` (+ tests) | **Platform logs is implemented, so no `501` stub is needed** — the earlier decision to stub it is withdrawn. It also sets the file-naming precedent this plan follows: `audit_handlers.go`, `audit_queries.go` |
| `internal/api/gen/*` regenerated | Already carries `QueryPlatformLogs`. Regenerating for audit then adds **two** methods, not three |
| `Makefile`: `SPEC :=` → `SPEC ?=`, with a comment naming the local-checkout override | The dev-loop override is now sanctioned by the module itself; this plan no longer needs a workaround |
| `helm/templates/fluent-bit/config.yaml`: a required `clusterInstance` and a `modify` filter stamping `openchoreo_cluster_instance` on `kube.*` | **A new ordering constraint on audit routing** — see P1-1 note 3 — and a field the audit index template must map |
| `init/setup-opensearch.sh`: `dynamic_templates` with `path_match`, and `kubernetes.labels` as `"type": "object", "dynamic": true` nested inside a `"dynamic": "false"` mapping | The house precedent that settles half of **D-1** |
| `README.md`: a mandatory-`clusterInstance` section ahead of `## Enable log collection` | The audit README section lands after it and must not restate it |

Rebase onto 606's head while it is in review; if it merges first, this becomes an ordinary branch from
`main`. **Do not start from `main`** — the conflict surface is four files that both changes edit in the same
regions.

### 0.2 The contract gate

`Makefile` pins codegen to the spec on **`main`**. Verified against `upstream/main` (`69e77b9e4`): it carries
`/api/v1alpha1/platform-logs/query` but **no `audit-logs` paths**. Those live only on this checkout's
`feat/audit-observer-contract` (`596f7e464`, `76203033a`, `113078b7c`). So:

| Phase | What | Blocked on |
|---|---|---|
| **Phase 1 — publish, store, retain** | fluent-bit routing, index template, ISM retention, values, README | **PR 606 only.** No contract dependency |
| **Phase 2 — query** | Go adapter implementing the two audit operations | PR 606 **and** the audit contract reaching `openchoreo` **`main`** |

Phase 1 stands alone and is worth landing on its own: it makes the trail *durable* under its own retention,
which is the compliance-relevant half. Phase 2 makes it *readable through the product*.

For local iteration in Phase 2, PR 606's own comment gives the form:

```sh
make openapi-codegen SPEC=../../openchoreo/openapi/observability-logs-adapter-api.yaml
```

The committed default still points at `main`, so the generated files must be regenerated from `main` before
the Phase 2 PR merges.

### 0.3 Corrections to the parent plans, verified in the tree

| Parent says | Actually |
|---|---|
| Publishing plan CM-OS-2b lists `origin` in the event envelope | The field is **`surface`** (`internal/server/middleware/audit/types.go:166`), renamed in `69e77b9e4`. `user_agent` (`:164`) is also missing from that table |
| Delivery plan §2.5: *"through the existing `GenerateIndices` day-walk"* | **The day-walk is wrong for audit** — see §2.3. 365d retention × a day-walk puts ~8 KB of index names in the request line |
| Publishing plan CM-OS-2a: output *"Host/Port/auth/tls identical"* | Also mirror `Generate_ID On`, `Replace_Dots On`, `Suppress_Type_Name On` (`helm/templates/fluent-bit/config.yaml:49,56,57`). `Generate_ID` is the one that dedups on retry |

---

## Phase 1 — Collection, destination and retention

Implements publishing-plan §CM-OS-1…3 with the corrections above. No Go changes; no codegen; no dependency on
the contract.

### P1-1 — Source-gated audit routing in fluent-bit

**File.** `helm/templates/fluent-bit/config.yaml`.

Insert between PR 606's `[FILTER] Name modify` — the cluster-identity stamp — and `[OUTPUT] Name opensearch`.
The ordering is load-bearing in both directions; see notes 3 and 4:

```
{{- if .Values.auditLogs.enabled }}
{{- range $p := .Values.auditLogs.producers }}
    [FILTER]
        Name          rewrite_tag
        Match         kube.var.log.containers.*_{{ $p.namespace }}_{{ $p.container }}-*
        Rule          $log "msg":"AUDIT-LOG"   audit.{{ $p.producer }}   false
        Emitter_Name  audit_emitter_{{ $p.producer }}
{{- end }}

    [FILTER]
        Name          parser
        Match         audit.*
        Key_Name      log
        Parser        audit_json
        Reserve_Data  On
        Preserve_Key  On
{{- end }}
```

plus, in the same ConfigMap's `parsers.conf` (`:62`):

```
    [PARSER]
        Name   audit_json
        Format json
```

Six things that must be right, most expensive first:

1. **The `Match` is the security control; the `Rule` is only a discriminator.** The tag derives from the
   kubelet-written container log filename, which a pod cannot influence. Routing on payload content alone
   would let any workload forge an audit record.
2. **The `Rule` regex must be `"msg":"AUDIT-LOG"`, not the bare token** — the bare token also matches
   `AUDIT-LOG-RENDER-FAILED`, and `Keep false` would then delete that diagnostic from the container-log
   stream while filing it in the audit index. Quote escaping in Fluent Bit's regex engine is **unverified**
   (R-2): test this one line in isolation first.
3. **Order after *both* `kube.*` filters — the `kubernetes` enrichment and PR 606's `modify` stamp.** Neither
   matches `audit.*`, so neither re-runs on a re-emitted record: whatever they attach must already be on the
   record when `rewrite_tag` fires. `kubernetes.*` is what Phase 2 returns as `collector.*`;
   `openchoreo_cluster_instance` is which cluster the producer ran in, which an audit record wants for the
   same reason a platform log does. **An audit record missing the cluster stamp is the symptom of getting
   this order wrong** — and it is silent, which is why §6 asserts on it.
4. **No emitter cycle.** `audit.<producer>` does not match `kube.var.log.containers.*`, so a rule cannot
   re-trigger on its own output. A distinct `Emitter_Name` per producer also keeps the emitters separately
   visible in fluent-bit's metrics, which is how an operator notices a rule that stopped firing.
5. **`Merge_Log` stays `Off` on `kube.*`** (`:41`). Turning it on globally restructures every
   `container-logs-*` document and breaks the adapter's regex log-level extraction
   (`internal/opensearch/types.go:331`). JSON decoding happens only on the audit tag.
6. **No `Time_Key`/`Time_Keep` on the audit parser, deliberately.** The tail input already applies
   `multiline.parser docker, cri` (`:28`), which sets the record time; re-deriving it from `event_time` would
   feed `Logstash_Format`'s **index name** from a re-parsed value, and Go marshals `time.Time` as RFC3339
   with a `Z` that `%z` does not match — a silent mis-file, not an error. `@timestamp` stays
   collector-observed; `event_time` is carried as an ordinary indexed `date`.

> **⚠️ `Reserve_Data On` + `Preserve_Key On` is to-verify, not settled (R-3).** Three things must hold at once
> on the audit tag: parsed fields at top level, `kubernetes.*` retained, raw `log` retained. Confirm
> empirically before wiring the rest. When the parse succeeds and `log` survives, the payload is stored twice
> in the same document — the accepted cost of keeping a verifiable raw line, which Phase 2 returns as the
> record's `log` field.

### P1-2 — The audit output, index template and retention

**Files.** `helm/templates/fluent-bit/config.yaml`, `init/setup-opensearch.sh`,
`helm/templates/opensearch-setup-logs/job.yaml`, `helm/values.yaml`.

**a. Second `[OUTPUT]`**, gated on `auditLogs.enabled`, mirroring the existing block (`:45-60`) and differing
only in `Match` and prefix:

```
    [OUTPUT]
        Name opensearch
        Match audit.*
        Logstash_Format On
        Logstash_DateFormat %Y-%m-%d
        Logstash_Prefix {{ .Values.auditLogs.indexPrefix | trimSuffix "-" }}
        Generate_ID On
        Replace_Dots On
        Suppress_Type_Name On
        ... Host/Port/auth/tls identical to the container-logs output
```

`Replace_Dots On` is carried for consistency with the sibling output and because the `kubernetes.*`
enrichment still flows through this tag; the audit record's own keys are snake_case and dot-free, so it is a
no-op on them. Stated so a reviewer does not read it as cargo-culted.

The existing output's `Match kube.*` already excludes `audit.*`, so nothing double-ships and no second guard
is needed. (This is *not* true of the OpenObserve module, whose output is `Match *` — out of scope here, see
§5.)

**b. `auditLogsIndexTemplate`** in `init/setup-opensearch.sh`, defined beside `containerLogsIndexTemplate`
(`:60`) and added to the `indexTemplates` array (`:248`), which the script's own comment invites. House
convention: `number_of_shards: 1`, `number_of_replicas: 1`, `"dynamic": "false"`.

Fields are derived from `eventJSON` (`internal/server/middleware/audit/types.go:155-171`) and `resourceJSON`
(`:176-186`) — **not from this table**; re-derive at implementation time. Under `"dynamic": "false"`
an undeclared field is stored but unindexed, so anything omitted here is silently unfilterable:

| Group | Fields | Type |
|---|---|---|
| Envelope | `schema_version`, `event_id`, `action`, `category`, `result`, `request_id`, `source_ip`, `user_agent`, `producer`, `surface`, `operation_id` | `keyword` |
| Time | `event_time` · `@timestamp` | `date` (both) |
| Actor | `actor.type`, `actor.id`, `actor.issuer`, `actor.session_id` | `keyword` |
| Entitlements | `actor.entitlements.*` → `actor.entitlements_values` | see D-1 |
| HTTP | `http.method`, `http.path` | `keyword` |
| Resource | `resource.{type,namespace,environment,project,component,resource,uid,name}` | `keyword` |
| Collector | `kubernetes.{namespace_name,pod_name,container_name}`, `openchoreo_cluster_instance` | `keyword` |
| Raw line | `log` | `wildcard` |

`openchoreo_cluster_instance` comes from PR 606's `modify` filter and is mapped here for the same reason it
is mapped on `container-logs-*`: it is the only record-level answer to *which cluster did this come from*.
**The contract has nowhere to return it** — `AuditLogCollectorInfo` carries namespace, pod and container
only. Indexing it now costs nothing and keeps the answer available to anyone querying OpenSearch directly;
adding `collector.clusterInstance` to the adapter contract is **F-5**, and Phase 2 does *not* smuggle it
into the record's `metadata`, which is the producer's own map and must not be overwritten by the collector.

Two mapping decisions to take now rather than defer — both are re-index-or-nothing once records exist:

- **`resource.environment` carries the dual-scoped `{namespace}/{name}` form** while its sibling tenancy
  fields are bare names, stored exactly as authorization evaluated it. `keyword` is right; the README must
  say why it differs, or a consumer filters it wrongly once and works around it forever.
- **`actor.entitlements` is `map[string][]string` with no declared key set**, and the contract requires it to
  be both *filterable across every claim key* and *aggregatable* (it is in the `filter` enum of
  `AuditLogFilterValuesRequest`). Both act on the map's values, so both are served by one declared
  `actor.entitlements_values` keyword field; **D-1** gives the two ways to populate it and which to verify
  first. PR 606 establishes both halves of the pattern in this very file — a `dynamic_templates` entry with
  `path_match`, and an inner `"type": "object", "dynamic": true` nested under a `"dynamic": "false"`
  mapping — so only the `copy_to` step is new.

`metadata` and `resource.metadata` (open `map[string]any`) stay **undeclared**: stored in `_source`,
returned in responses, deliberately not indexed. Record it; do not leave it to be discovered.

**c. `auditLogsIsmPolicy`**, modelled on `containerLogsIsmPolicy` (`:361`) — the same `active → delete`
two-state machine, `ism_template.index_patterns: ["audit-logs-*"]`, `priority: 100` — added to the
`ismPolicies` array (`:436`) with a retention variable beside `:357-358`:

```bash
auditLogsRetention="${AUDIT_LOGS_MIN_INDEX_AGE:-365d}"
```

`reconcile_ism_policy_indices` (`:443`) derives its index pattern from the policy name, so naming the policy
`audit-logs` makes existing indices adopt it with no further change.

**d. Job env**, beside the two existing `*_MIN_INDEX_AGE` vars (`helm/templates/opensearch-setup-logs/job.yaml:41-44`):

```yaml
        - name: AUDIT_LOGS_MIN_INDEX_AGE
          value: "{{ .Values.openSearchSetup.dataRetentionTime.auditLogs }}"
```

**e. Values.** `dataRetentionTime.auditLogs: "365d"` beside `containerLogs: "30d"` and `k8sEvents: "30d"`
(`helm/values.yaml:222-225`). **Audit defaults far longer on purpose — a different retention policy is the
entire reason audit gets its own stream.** See D-4.

### P1-3 — The `auditLogs` values block

**File.** `helm/values.yaml`.

```yaml
auditLogs:
  # Route OpenChoreo audit records to their own index. Requires fluent-bit to be
  # running in the cluster where openchoreo-api / observer run.
  enabled: false

  # Index name prefix. Also read by the adapter, so the two cannot drift.
  indexPrefix: "audit-logs-"

  # The source allowlist. Each entry becomes one rewrite_tag rule matched on the
  # kubelet-written container log filename, so it cannot be satisfied by anything
  # a pod puts in its own log line. Adding an entry grants a workload the right to
  # write into the audit trail — review it as such.
  producers:
    - producer: openchoreo-api
      namespace: openchoreo-control-plane
      container: api-server
    - producer: observer
      namespace: openchoreo-observability-plane
      container: observer
```

**The defaults are verified, not guessed:** container `api-server`
(`install/helm/openchoreo-control-plane/templates/openchoreo-api/deployment.yaml:60`) and container `observer`
(`install/helm/openchoreo-observability-plane/templates/observer/observer-deployment.yaml:63`). Note the
container is `api-server`, **not** `openchoreo-api`. Both namespaces are Helm release namespaces and
overridable.

**Gating: routing is conditional, destination shaping is not.** Gate P1-1 and P1-2a on `auditLogs.enabled`;
leave the index template and ISM policy **ungated**, so the destination is correctly shaped before the first
record could arrive. This is deliberate: every install gets an `audit-logs` template and policy, including
ones that never publish audit. Both are metadata against a pattern matching nothing, so the cost is zero —
and the alternative is exactly the failure the README's existing troubleshooting entry (`README.md:167`)
describes for `container-logs-*`. Say so in the values comment so a reviewer does not "fix" it.

---

## Phase 2 — The query path

Three generated operations to implement. Gated on §0.

### P2-1 — Regenerate, and stub what is not in scope

```sh
make openapi-codegen
```

Regenerates `internal/api/gen/{models,server,client}.gen.go`. On top of PR 606 this adds exactly two methods
to `StrictServerInterface` — `QueryAuditLogs` and `QueryAuditLogFilterValues` — and the module will not
compile until both exist. Land the regeneration as its own commit: a mechanical diff is unreviewable when
mixed with new logic.

### P2-2 — Index resolution: drop the day-walk for audit

`GenerateIndices` (`internal/opensearch/queries.go:393`) enumerates one index name **per day** in the window,
and `Client.Search` (`internal/opensearch/client.go:95`) passes them as `SearchReq.Indices`, which the
client puts in the request path. Container logs retain 30 days so this has never mattered. Audit retains
**365**: a full-window query produces ~365 × ~22 chars ≈ **8 KB of index names in the request line**, against
OpenSearch's 4 KB `http.max_initial_line_length` default. The failure surfaces as a malformed request, not as
a length error.

**So the audit path resolves `audit-logs-*` and relies on the range filter**, and there is a second reason
this is right: a PIT opened against a wildcard resolves concrete indices *at open time*, which is precisely
the "a new daily index cannot appear mid-scroll" property the contract's cursor asks for.

To-verify (R-4): that the shard pre-filter (`can_match`) prunes shards using the `event_time` range so the
wildcard does not cost a full-retention scan on a one-hour query. If it does not, the fallback is a bounded
day-walk — window ≤ N days enumerates, wider falls back to the wildcard — not a return to the unbounded walk.

### P2-3 — Client and query builder

**`internal/opensearch/client.go`** — the existing `Search` discards two things the audit path needs:
`resp.Aggregations` and per-hit `Sort` values. Both are available on the vendored client
(`opensearch-go/v4@v4.7.3/opensearchapi/api_search.go:74` and `:107`). Extend `SearchResponse`
(`internal/opensearch/types.go:15`) with `Aggregations json.RawMessage` and `Hit.Sort []any`, and populate
them in `Search` — additive, so no existing caller changes. `Hits.Total.Relation` is **already** parsed
(`types.go:19`, populated at `client.go:112`) and today thrown away by the handlers; audit is the first
caller that must report it honestly.

Add PIT lifecycle methods: `CreatePIT(ctx, indices, keepAlive)` and `DeletePIT(ctx, id)`, over
`opensearchapi`'s point-in-time API (present in v4.7.3: `api_point_in_time-create.go`, `-delete.go`). A search
carrying a `pit` in its body must be issued with **no** `Indices` — the PIT carries them.

**`internal/opensearch/audit_queries.go`** (new) — mirroring PR 606's `platform_queries.go`, holding:

| Builder | Notes |
|---|---|
| `BuildAuditLogsQuery` | `bool.filter` of `range` on `event_time` + one `terms` per filter group, via PR 606's `addTermsFilter` helper — which already encodes "multi-value ORs within a field, fields AND, empty is not a filter", the exact semantics the audit contract states |
| `searchPhrase` | `wildcard` on `log`, reusing `sanitizeWildcardValue` (`queries.go:16`) |
| Sort | `[{event_time: <order>}, {event_id: asc}]`. `event_id` is a UUID v7 keyword, unique per record — a deterministic tiebreaker, and avoids depending on `_shard_doc` |
| `BuildAuditTimelineAgg` | `date_histogram` on `event_time`, `min_doc_count: 0` + `extended_bounds` from the window (the contract requires contiguous zero-filled buckets), sub-agg `terms` on `result` |
| `BuildAuditFilterValuesAgg` | `terms` on the named field, `size: maxValues`, `order` by `_count` desc then `_key` asc |

`NewQueryBuilder(auditPrefix)` gets its own instance on `LogsHandler` beside `queryBuilder` and
`eventsQueryBuilder` (`internal/handlers.go:23-29`), mirroring how events were added.

**Bucket coarsening.** `timelineInterval` is `<count><unit>`; compute the bucket count over the window and, if
it exceeds 500, coarsen to the next unit until it does not — **coarsen, never reject** — and echo the width
actually used in `timeline.interval`.

**`valueSearch` is case-insensitive substring, and a terms-agg `include` is a Lucene regex with no `(?i)`.**
Verified: `include` accepts a Lucene regex or an exact-value array, anchored, capped at
`index.max_regex_length` (default **1000**). So build a character-class expansion —
`cli` → `.*[cC][lL][iI].*` — which keeps the *original-cased* values in the response, as the contract requires
(a lowercase-normalizer sub-field would return lowercased values and break that). The expansion is ~4× the
input and `valueSearch` allows 256 chars, so a long search can exceed 1000: over-fetch and filter in Go above
a threshold, rather than sending a regex the cluster rejects.

### P2-4 — Cursor: PIT plus `search_after`

Verified against the OpenSearch 3.x docs (`POST /<target>/_search/point_in_time?keep_alive=…`, wildcard
patterns accepted; `pit.id` + `sort` + `search_after` in the search body).

```
cursor := base64( { pit_id, keep_alive, sort: [...last hit's sort values], order } )
```

- **First page:** open a PIT against `audit-logs-*`, search with `sort`, return `nextCursor` **only when the
  page was full** — `nextCursor` absent is the sole end-of-results signal.
- **Subsequent pages:** decode, search with `pit` + `search_after`. Each access extends the PIT lifetime by
  `keep_alive`.
- **Expiry → `410`, never an empty `200`.** An expired or rolled PIT surfaces as a search failure
  (`SearchContextMissingException`); the exact error shape is **to-verify (R-5)** — do not pattern-match on a
  guessed message. `410` is what the observer maps to `ErrAuditLogsCursorExpired`
  (`internal/observer/service/audit_logs_adapter.go:55`); anything else it reports as a generic failure.
- `keepAlive` is configurable (default `5m`) — long enough for human paging, short enough that abandoned
  scrolls release resources. An abandoned PIT is reclaimed on expiry; do not attempt to delete on last page,
  since the adapter cannot know a caller has stopped.

**`total` and `totalRelation`.** Set `track_total_hits` to a bound (the contract caps `limit` at 1000 but says
nothing about the count) and pass OpenSearch's own `relation` through as `eq`/`gte`. Never report a capped
count as exact — an audit consumer reading a capped `total` as exact draws the wrong conclusion about how
much happened.

### P2-5 — Handlers

**`internal/audit_handlers.go`** (new), following PR 606's `platform_handlers.go` shape and reusing the
`derefSlice` / `optional` helpers it adds to the package rather than declaring more.

`QueryAuditLogs`:
1. Body present; `startTime`/`endTime` present.
2. Build query → open or resume PIT → search → map hits to `gen.AuditLogRecord`.
3. `includeTimeline` → add the timeline agg. **If it cannot be computed, omit `timeline` and answer
   normally** — the contract says so explicitly, and an empty timeline is a different claim from an absent one.
4. Map `kubernetes.{namespace_name,pod_name,container_name}` → `collector.*`, and the preserved raw line →
   `log`.

`QueryAuditLogFilterValues`:
1. Apply `query`'s other filters but **not the named filter's own** — a picker must keep offering the
   alternatives to what is already selected.
2. Ignore `query.limit`, `sortOrder`, `cursor`, `includeTimeline`, `timelineInterval` — they carry no meaning
   here and the contract says to ignore, not reject.
3. Honour `query.startTime`/`endTime` and `query.searchPhrase` exactly as a record query would.
4. `totalValues` from a distinct-count; label it `gte` unless it is genuinely exact. Distinct counting on a
   high-cardinality field is approximate, and the contract exists to let an adapter say so.

**Record-shape obligations** (each separately checked by the observer — `audit_logs_adapter.go:52-60`,
`:107-113`): `501` on the two audit operations is **separately declinable**; a `410` is not a `400`; records
carry snake_case keys exactly as published.

### P2-6 — Config and Helm wiring

| File | Change |
|---|---|
| `internal/config.go` | `OpenSearchAuditIndexPrefix` from `OPENSEARCH_AUDIT_INDEX_PREFIX`, default `audit-logs-`, beside the two existing prefixes (`:34-35`); `AuditCursorKeepAlive` from `AUDIT_CURSOR_KEEP_ALIVE`, default `5m` |
| `helm/templates/adapter/configmap.yaml` | `OPENSEARCH_AUDIT_INDEX_PREFIX: "{{ .Values.auditLogs.indexPrefix }}"` beside `:15-16` |

**One value, two renderings.** `auditLogs.indexPrefix` feeds fluent-bit's `Logstash_Prefix` (which needs it
*without* the trailing dash — hence `trimSuffix "-"`) and the adapter env var (*with*, matching the
`container-logs-` convention in `config.go:34`). Template both from the single value so they cannot drift; a
mismatch means records are written where nothing reads them, with no error anywhere.

**Out of scope, stated so it is not assumed:** a separate OpenSearch credential or cluster for the audit
index. The adapter reads audit through the same admin credential it uses for container logs
(`helm/templates/adapter/deployment.yaml:42-51`). Separating them is a real hardening step and a later one.

---

## 3. Documentation — the module side

### DOC-1 — README: `## Enable audit log collection`

**File.** `README.md`, a new section after `## Enable log collection` (`:84`), following the module's
established spine, with four subsections:

**Trusted producers.** Must not read like ordinary configuration. Say, in the operator's terms, that each
entry grants a workload the right to write into the audit trail; that the list defaults to OpenChoreo's own
two producers; and that a non-default release namespace requires editing it **or audit collection silently
stops**. Show the forgery example — a pod writing a well-formed audit line to stdout and it *not* being
routed — because an operator who has seen it will not widen the list casually.

**Configuring the audit destination.** `auditLogs.enabled`, `auditLogs.indexPrefix`, and the single-cluster /
multi-cluster split. PR 606 already covers the control-plane cluster in its remote-install instructions and
makes `fluent-bit.clusterInstance` mandatory — **reference that, do not restate it**; the audit-specific
point is only that audit collection needs fluent-bit in the cluster where openchoreo-api and observer run.

**Retention.** `openSearchSetup.dataRetentionTime.auditLogs`, why it defaults to `365d` rather than `30d`,
and that changing it is reconciled onto existing indices by the setup job on the next upgrade
(`init/setup-opensearch.sh:443`).

**Credentials.** Points at the existing `ExternalSecret` prerequisite (`:13-42`); notes that audit currently
shares the container-logs credential and that this is a known limitation, not a design intent.

### DOC-2 — README: querying, and what the two timestamps mean

Short subsection under the same heading:

- Which observer endpoint serves the trail, and that this module answers it (or `501`, before Phase 2).
- **`@timestamp` vs `event_time`.** `@timestamp` is when the collector saw the line; `event_time` is when the
  audited request was received. The daily index is chosen by the former, all filtering and sorting is on the
  latter. They differ, that is intended, and a consumer filtering on the wrong one gets subtly wrong answers.
- **`resource.environment` is `{namespace}/{name}`**, not a bare name — a bare-name filter matches nothing.
- What is stored but **not indexed**: `metadata`, `resource.metadata`.

### DOC-3 — Troubleshooting entry

In the same class as the existing one (`:167`): an `audit-logs-*` index created **before** the template was
applied carries dynamic mappings instead of the declared ones and returns nothing. Same remedy shape as the
`container-logs-*` entry, with the audit index pattern.

### DOC-4 — Compatibility and version

Add a Compatibility row (`:191-195`) and update the `--version` pins in the README install commands.

**Do not bump `VERSION` unilaterally.** PR 606 carried a bump and then explicitly removed it
(`455fa99aa`, *"remove VERSION bump"*), so release numbering is being handled outside the feature PRs here.
Follow whatever 606 settles on rather than reintroducing the question; the existing drift — README pins
`0.5.3`, `VERSION` is `0.5.4` — is not this change's business either.

### DOC-5 — What does *not* belong here

Module READMEs reach the docs site only through a **runtime `fetch()`** of `raw.githubusercontent.com` — no
build-time sync, no vendored copy. They are unversioned, absent from DocSearch, and missing from the llms.txt
and markdown-export outputs. So anything an operator must find **by searching** belongs in
`openchoreo.github.io`, not here: why audit is on its own destination, the collector trust boundary, and the
ST-7 conformance matrix row. Tracked in §5 as a follow-up on the site repo — this plan does not cover it.

---

## 4. Sequencing

```
PR 606 (platform logs) ──► Phase 1  P1-1 ─► P1-2 ─► P1-3 ─► DOC-1/3/4
                                                                 │
                     contract merges to openchoreo main ─────────┤
                                                                 ▼
                           Phase 2  P2-1 ─► P2-2/3 ─► P2-4 ─► P2-5 ─► P2-6 ─► DOC-2
```

| # | Commit | Phase | Why here |
|---|---|---|---|
| 1 | `feat: route audit records to their own index` | 1 | P1-1 + P1-2a + the `auditLogs` values block |
| 2 | `feat: add the audit logs index template and retention policy` | 1 | P1-2b/c/d/e. Destination shaping lands **before** routing goes live in any real install |
| 3 | `docs: document audit log collection` | 1 | DOC-1, DOC-3, DOC-4 |
| 4 | `chore: regenerate the adapter API stubs` | 2 | P2-1 alone — mechanical, reviewable |
| 5 | `feat: serve audit log queries from opensearch` | 2 | P2-2…P2-5 |
| 6 | `feat: configure the audit index prefix and cursor lifetime` | 2 | P2-6 + DOC-2 |

Commits 1 and 2 are ordered that way for review clarity; **for deployment the reverse holds** — the template
must exist before records arrive, which §P1-3's ungated shaping guarantees regardless of install order.

**The conflict surface with PR 606** is four files edited in the same regions: `fluent-bit/config.yaml`
(filter chain), `values.yaml` (the `fluent-bit` block), `init/setup-opensearch.sh` (the template
definitions), and `README.md` (around `## Enable log collection`). Rebasing on 606 while it is in review
keeps those resolutions in this branch rather than in a merge nobody reviews. If 606 changes shape in
review — particularly the `modify` filter's position — **re-check P1-1 note 3 before landing**.

---

## 5. Risks, open decisions and follow-ups

| # | Item | Disposition |
|---|---|---|
| **R-1** | **#4555 may change the tail input's tag scheme.** P1-1's `Match` assumes `Tag kube.*` over `Path /var/log/containers/*.log` (`config.yaml:27,32`). If a future split changes the prefix, every `Match` matches nothing — **no records, no error** | Not blocking. Re-verify when #4555 lands, and rely on §6's *positive* assertions: every negative test passes when nothing is collected at all |
| ~~**R-2**~~ | **CONFIRMED REAL, then fixed — 2026-09-14, on k3d.** `Rule` is tokenized into its four parts by a splitter that treats `"` as a delimiter, so `"msg":"AUDIT-LOG"` was torn apart: the regex became `msg` and **the new tag became `:"AUDIT-LOG"`**, proved by a temporary catch-all `stdout` output printing that literal tag. Nothing matched it, so with `Keep false` every audit record was removed from `container-logs-*` and then **silently discarded** — an unroutable tag is not an error in Fluent Bit. Fixed by a quote-free `AUDIT-LOG(?!-)`; the negative lookahead is what now excludes `AUDIT-LOG-RENDER-FAILED`. **This was the single highest-consequence risk in the plan and it was real** |
| ~~**R-3**~~ | **CLOSED 2026-09-14, verified on a stored record.** `Reserve_Data On` + `Preserve_Key On` delivers all three at once: parsed audit fields at top level (`action`, `actor`, `event_id`, `event_time`, `result`, …), `kubernetes.*` enrichment retained, and the raw `log` line preserved. `openchoreo_cluster_instance` survived the re-tag too, confirming P1-1 note 3's ordering |
| **R-11** | **`Parsers_File` is resolved against the collector's `--workdir` (`/fluent-bit/etc`), not the config file's directory (`/fluent-bit/etc/conf`).** The chart's existing `Parsers_File parsers.conf` therefore loads the *image's built-in* file, never the ConfigMap's — a latent bug the module has always had, invisible because nothing referenced the one parser it defines | Fixed for audit only: the audit parser lives in its own `audit-parsers.conf` key loaded by absolute path. The pre-existing dead `parsers.conf` is left alone — out of scope, but worth an issue |
| **R-9** | **`track_total_hits: true` can never report a capped count.** Verified on 3.3.0: `true` counts every match and always answers `relation: "eq"`; a bound answers `{value: <bound>, relation: "gte"}` once matches exceed it. `true` is also an unbounded pass over a year-deep index — the cost the wildcard pattern exists to avoid | Fixed during implementation: `track_total_hits` is `auditTotalHitsCap` (10000). A unit test now pins it, because the bug was invisible to a test fed a hand-written `gte` payload the backend would never send |
| **R-10** | **The adapter contract's `ErrorResponse.title` enum had no `gone`**, though the 410 response's own example uses `title: "gone"` — so the spec contradicted itself and the only in-enum choice was the `badRequest` that the distinct 410 exists to avoid | Fixed in `openapi/observability-logs-adapter-api.yaml` (additive, one line). The observer needs no change: it deliberately maps the adapter's 410 onto its own `400` with `ErrorCodeV1AuditLogsCursorExpired` (`auditlogs.go:118-126`) |
| **R-8** | **Regenerating renames a shared enum constant.** The audit `result` enum introduced a `success` value that collides with the alert webhook's, so oapi-codegen disambiguated **both**: `gen.Success` → `gen.AlertWebhookResponseStatusSuccess` | Found and fixed during implementation (3 call sites in `handlers.go`, 1 in `handlers_test.go`). Recorded because it is a pre-existing-code change that a reviewer will otherwise read as unrelated churn |
| **R-4** | **`can_match` pruning over `audit-logs-*`** may not hold, making a narrow query scan full retention | Measure in Phase 2 (§P2-2). Fallback is a *bounded* day-walk, not the unbounded one |
| ~~**R-5**~~ | ~~Expired-PIT error shape is unverified~~ | **Closed 2026-09-14, verified against OpenSearch 3.3.0 in k3d.** A gone PIT answers **HTTP 404** with `search_context_missing_exception` / `"No search context found for id [N]"` in the body, wrapped in a `search_phase_execution_exception`. `isPITExpiredResponse` matches the exception name and that phrase, and maps to `410` |
| **R-6** | **Timestamp semantics.** `@timestamp` (collector) and `event_time` (adapter entry) differ, and the daily index is chosen by the former | Intended. Documented in DOC-2. Do **not** "fix" it by re-parsing `event_time` into the record time — see P1-1 note 6 |
| ~~**D-1**~~ | **CLOSED 2026-09-14 — option (a) verified working on OpenSearch 3.3.0.** A `dynamic_template` on `actor.entitlements.*` with `copy_to: actor.entitlement_values` gives both halves at once: a `terms` filter on `sre` matched the record, and a `terms` aggregation returned `admin`, `platform-admins` and `sre` — values drawn from **two different claim keys** (`groups` and `roles`), which is exactly what the contract asks for. The Lua fallback is not needed. Original analysis: **`actor.entitlements`: how to make it both filterable across claim keys and aggregatable.** Verified from the OpenSearch 3.x docs: `flat_object` supports term/terms/prefix/range/match/wildcard/exists queries and **aggregations are not among them** — so `flat_object` satisfies the record filter and fails the picker. Both operations act on the map's *values*, not its keys, so the answer in either design is one declared `actor.entitlements_values: keyword`. **Two working ways to populate it** — pick on verification, not on preference: **(a)** a `dynamic_template` mapping `actor.entitlements.*` as `keyword` with `copy_to: actor.entitlements_values`, preferred because it is no collector code and because PR 606 already establishes dynamic templates and nested `dynamic: true` in this file, leaving only `copy_to` unproven; **(b)** a Lua filter on the `audit.*` tag flattening the map's values into an array before the OUTPUT — no mapping-engine behaviour relied on, at the cost of collector code in a filter chain P1-1 already establishes. **(b) is the stated fallback, so the decision closes either way** |
| **D-2** | **`metadata` / `resource.metadata` unindexed** | Recommend accept: open maps with no declared shape, returned in responses, not filterable. Record it in DOC-2 |
| **D-3** | **Audit retention default `365d`** | A product decision, not a technical one. It is the number an operator will most often override and it drives OpenSearch storage sizing |
| **D-4** | **PIT `keep_alive` default `5m`** | Trades abandoned-scroll resource hold against how long a human may sit on a page. Configurable; `5m` is a starting point, not a measured one |
| **F-1** | `observability-logs-openobserve` — same contract, and its `[OUTPUT] Match *` is a **silent authorization hole** for audit (records would also land in the shared default stream) | Out of scope. Open as its own issue; the `Match *` narrowing is the highest-risk line in that module |
| **F-2** | The other five `observability-logs-*` modules need a trivial `501` | Out of scope. They break only on their next regeneration; open proactive issues rather than leaving landmines |
| **F-3** | Site docs: the audit-logging hub page, the collector trust boundary, the ST-7 conformance matrix row | Out of scope (DOC-5). The PR that adds audit support to a module updates its matrix row |
| **F-4** | ~~`QueryPlatformLogs` real implementation~~ | **Closed by PR 606** (§0.1). No stub, no follow-up |
| **F-5** | **The contract has no place to return the collecting cluster.** `AuditLogCollectorInfo` carries namespace, pod and container; PR 606 stamps `openchoreo_cluster_instance` and platform logs returns it as `clusterInstance` | Index it in Phase 1 so the answer exists; propose `collector.clusterInstance` on `observability-logs-adapter-api.yaml` as a separate contract change. Do **not** smuggle it into the record's producer-owned `metadata` |
| **R-7** | **PR 606 is open, not merged.** Its `modify` filter position, its `clusterInstance` requirement and its `dynamic_templates` block are all load-bearing here | Rebase, don't branch from `main`. Re-verify P1-1 note 3 and the D-1 precedent against 606's final state before landing |

---

## 6. Verification — the definition of done

### Phase 1, on k3d

Install the observability plane and this module with `auditLogs.enabled=true`, then perform any audited
operation.

Six assertions. **The order matters — the two negative ones are worth writing first, because a content-only
rule passes every positive check:**

| # | Assertion | Catches |
|---|---|---|
| 1 | The record is in `audit-logs-YYYY-MM-DD` | The `Match` patterns resolve against the real tag (R-1) |
| 2 | Repeat for an `observer` operation | A typo in one of the two rules, invisible in the other's result |
| 3 | It is **absent** from `container-logs-*` | `Keep false` |
| 4 | **Negative — over-broad marker.** A line containing `AUDIT-LOG-RENDER-FAILED` lands in `container-logs-*`, **not** `audit-logs-*` | The P1-1 note 2 bug |
| 5 | **Negative — forgery.** An ordinary data-plane workload writes a well-formed audit record to stdout claiming `"producer":"openchoreo-api"`. It appears **only** in `container-logs-*` | The whole point of the source gate |
| 6 | A stored record carries `kubernetes.container_name: api-server` beside `producer: openchoreo-api`, **and `openchoreo_cluster_instance` is set**, and `log` is present and searchable | Filter ordering (note 3) against *both* `kube.*` filters, and that `Preserve_Key On` is paired with its mapping |

Retention:

```sh
curl -s localhost:9200/_plugins/_ism/policies/audit-logs | jq   # 365d, distinct from container-logs
```

### Phase 2

```sh
make unit-test     # in the module
```

Unit coverage for: filter→DSL mapping per field group; cursor encode/decode round-trip; bucket coarsening
above 500; `totalRelation` passthrough; the `valueSearch` character-class expansion, including the
over-length fallback. Follow `platform_queries_test.go` / `platform_handlers_test.go` for shape.

End to end against the observer:

| Check | Expected |
|---|---|
| Query with no cursor, page through to the end | `nextCursor` absent on the last page — and never an empty page carrying one |
| Page with a deliberately expired cursor | `410`, and the observer surfaces "restart the query" — never an empty `200` |
| `includeTimeline` over a 7-day window at `15m` | Coarsened to ≤500 buckets, actual width echoed, **zero-count buckets present** |
| Filter values for `actor.id` under a `result=denied` query | Values reachable under the rest of the query; `actor.id`'s own selections ignored |
| `valueSearch: "CLI"` against mixed-case values | Case-insensitive, original casing returned |
| The audit read itself | Appears in the trail — the read path is audited |
| Stop the adapter | A clear error from the observer, not an empty success |
