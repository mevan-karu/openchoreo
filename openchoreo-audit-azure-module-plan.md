# Audit logs module for Azure — implementation plan

Status: draft for review · 2026-09-22

Target repo: `openchoreo/community-modules`
Reference implementations: `observability-logs-opensearch` (0.6.0), `observability-logs-openobserve`
Existing Azure module: `observability-logs-azure-loganalytics` (0.1.3, **no audit support**)

---

## 0. Decisions at a glance

| # | Decision | Why |
|---|---|---|
| 1 | Dedicated custom table `OpenChoreoAuditLog_CL`, **not** a KQL filter over `ContainerLogV2` | Retention and read access are both *table* properties — §2 |
| 2 | `TimeGenerated = event_time`, plus a separate unclamped `EventTime` column | `TimeGenerated` is the partition column; Azure clamps back-dated values — §3.2, §3.2.1 |
| 3 | Where the table lives and who may read it is the **operator's** decision. The adapter keeps its single existing identity; the module adds one config field. | Separate workspace vs table-scoped ABAC is a role-assignment topology, not a code path — §4 |
| 4 | Ingest by splitting the **Container Insights DCR** (two complementary data flows); fluent-bit is the documented fallback | No new collector, no client secret, unforgeable AMA-stamped provenance — §5 |
| 5 | Parse into typed columns **at ingest**, not at query time | Cheap bounded queries, and row-level ABAC only works on real string columns — §3.3, §5.1 |
| 6 | Interactive retention covers the entire served window; search jobs are an **operator runbook**, never adapter code | Search jobs need table-write, lack `summarize`, and bypass granular RBAC — §6 |
| 7 | Ship audit **inside** `observability-logs-azure-loganalytics`. **No core-repo change required.** | The observer reaches audit via `LOGS_ADAPTER_URL`, and one component correctly has one identity — §7.2 |
| 8 | Provision Azure resources by documented ARM/Bicep, not a Helm hook | The module owns none of its Azure resources — §10 |

Three things must be verified in a real subscription before implementation starts: transformation
billing granularity, whether the AKS addon reconciles away the added data flow, and whether
`TimeGenerated` can be overridden on the `Microsoft-ContainerLogV2` stream. See §12.

---

## 1. The question this plan exists to answer

OpenSearch and OpenObserve give the audit trail a **separate index / stream**. That separation is
not cosmetic — it is the mechanism that delivers two properties:

- **Retention.** Audit records live for `365d` while container logs live for `30d`
  (`openSearchSetup.dataRetentionTime.auditLogs: "365d"` vs `containerLogs: "30d"`). The
  OpenSearch module's own comment says it outright: *"Needing a different retention is the reason
  audit is a separate stream rather than a filter over the container logs."*
- **Security.** A distinct storage object is the unit an access rule can name, so an
  operational-log reader is not automatically an audit reader.

So the first question for Azure is: **what is the equivalent of an index, such that retention and
access can be set on it independently?**

Answer: a **dedicated custom table** (`*_CL`) in a Log Analytics workspace. Azure sets both
retention *and* read access per table. The rest of this plan follows from that choice.

### 1.1 Concept mapping

| OpenSearch / OpenObserve | Azure Monitor Logs | Notes |
|---|---|---|
| `audit-logs-*` index / stream | Custom table `OpenChoreoAuditLog_CL` | The isolation unit |
| Index template (field mappings) | Table schema (typed columns) + DCR `streamDeclarations` | Declared up front, not dynamic |
| ISM policy → delete at `min_index_age` | `retentionInDays` + `totalRetentionInDays` on the table | No policy object; it is a table property |
| Daily index rollover (`Logstash_DateFormat`) | **Not needed** — Azure partitions internally | Removes a whole class of failure |
| `dynamic: false` mapping (unmapped ⇒ unfilterable) | Column must exist in the table, else the value is dropped | Same failure mode, harder failure |
| `copy_to: actor.entitlement_values` | `extend` in the DCR transform | Must be built at ingest |
| Index-pattern read role | Granular RBAC (ABAC) condition on the table, **or** a separate workspace | See §4 |
| Write user scoped to the audit index | `Monitoring Metrics Publisher` on a **single DCR** | Stronger than OpenSearch's role |
| Producer allowlist (kubelet log filename) | `PodNamespace` + `ContainerName` in the DCR transform | Same unforgeable-provenance property |
| `rewrite_tag ... false` (don't keep under original tag) | Two complementary `dataFlows` filters | See §5.3 — must not drift |

### 1.2 Where Azure is better, and where it is worse

**Better:**

- **Write isolation is physical, not policy.** The ingesting identity holds
  `Monitoring Metrics Publisher` on one DCR, and that DCR declares exactly one output stream. The
  writer *cannot* target another table even if compromised. In the OpenSearch module today,
  fluent-bit writes with the shared `admin` credential — it could write anywhere.
- **Append-only by construction.** The Logs Ingestion API has no update or delete verb. Erasing
  audit data requires `Microsoft.OperationalInsights/workspaces/purge/action` (the `Data Purger`
  role), which we deliberately leave unassigned. There is no OpenSearch equivalent of that
  separation — `admin` can `DELETE /audit-logs-*`.
- **Retention to 12 years** (`totalRetentionInDays` up to 4383) without a second system.
- **Row-level access** is expressible (ABAC on a column), which OpenSearch's module does not
  attempt at all.

**Worse / needs care:**

- **Two-tier retention.** Only *interactive* retention (max 730 days) is reachable by the
  synchronous query API. Data past that is reachable only by an asynchronous **search job**, which
  requires `workspaces/tables/write` + `workspaces/searchJobs/write` (Log Analytics Contributor),
  does not support `summarize`, and is not yet covered by granular RBAC. **A read-only audit
  adapter therefore cannot and must not run search jobs.** See §6.
- **ABAC is additive with no deny.** A table-scoped condition is defeated by any broader
  assignment the same principal holds. Not a module concern, but the sharpest edge an operator can
  cut themselves on, so the README must name it — §4.1.
- **Schema is closed at ingest.** A field absent from the table schema is silently dropped, not
  merely unindexed. Worse than OpenSearch's `dynamic: false`, where the value at least survives in
  `_source`.
- **Transformation billing.** Dropping more than 50% of a data flow's input is billed. See §5.5 —
  this is a real cost item for the recommended ingestion path.

---

## 2. Decision 1 — dedicated custom table, not a filter over `ContainerLogV2`

**Rejected alternative:** leave audit records in `ContainerLogV2` and have the adapter recognise
them in KQL (`PodNamespace in (...) and ContainerName in (...) and LogMessage has 'AUDIT-LOG'`).
Zero new Azure resources; tempting.

**Why it fails the requirement — both halves of it:**

- Retention is a **table** property. Audit records in `ContainerLogV2` inherit container-log
  retention. You cannot have 365-day audit and 30-day application logs in one table.
- Access control is a **table** property. Anyone granted read on `ContainerLogV2` — which is every
  developer using the logs feature — can read the full audit trail, including the
  `category: access` records about who read the audit trail. That inverts the point of the feature.

**Decision:** dedicated custom table. This is also the pattern Microsoft documents for exactly this
purpose: *"Send sensitive records to an alternate table with different role-based access control
configuration."*

Consequence to accept: audit records must be **removed** from `ContainerLogV2`, not merely copied
out of it, or the isolation is theatre. That is what §5.3 is about.

---

## 3. Decision 2 — table schema

### 3.1 Authoritative field list

The published record is defined by `eventJSON` / `resourceJSON` in
`openchoreo/internal/server/middleware/audit/types.go` (verified; `SchemaVersion = "1.0"`). **That
file is the source of truth, not the OpenSearch index template** — reconcile against it on every
schema change.

Proposed columns. Azure custom-table column rules applied: max 45 chars, must start with a letter,
`[A-Za-z0-9_]` only, and `Type`, `id`, `TenantId`, `UniqueId`, `Title`, `_ResourceId`,
`_SubscriptionId` are **reserved**.

| Column | Type | Source | Note |
|---|---|---|---|
| `TimeGenerated` | datetime | `event_time` | **The partition column — see §3.2.** Azure may clamp it (§3.2.1) |
| `EventTime` | datetime | `event_time` | Verbatim, never clamped. `TimeGenerated != EventTime` ⇒ the record was clamped |
| `CollectedTime` | datetime | AMA's original `TimeGenerated` | Collector clock; skew detection |
| `SchemaVersion` | string | `schema_version` | |
| `EventId` | string | `event_id` | UUID v7 |
| `Action` | string | `action` | |
| `Category` | string | `category` | management \| authorization \| access |
| `Result` | string | `result` | success \| failure \| denied |
| `RequestId` | string | `request_id` | |
| `SourceIP` | string | `source_ip` | |
| `UserAgent` | string | `user_agent` | client-supplied, unverifiable |
| `Producer` | string | `producer` | producer's *claim*; compare with `CollectorContainer` |
| `Surface` | string | `surface` | rest \| mcp |
| `OperationId` | string | `operation_id` | |
| `ActorType` | string | `actor.type` | **not** `Type` — reserved |
| `ActorId` | string | `actor.id` | unique only within `ActorIssuer` |
| `ActorIssuer` | string | `actor.issuer` | |
| `ActorSessionId` | string | `actor.session_id` | |
| `ActorEntitlements` | dynamic | `actor.entitlements` | arbitrary keys; `map[string][]string` |
| `ActorEntitlementValues` | string | flattened from the above | ABAC/filtering surrogate — see §3.3 |
| `HttpMethod` | string | `http.method` | |
| `HttpPath` | string | `http.path` | |
| `ResourceType` | string | `resource.type` | |
| `ResourceNamespace` | string | `resource.namespace` | |
| `ResourceEnvironment` | string | `resource.environment` | |
| `ResourceProject` | string | `resource.project` | |
| `ResourceComponent` | string | `resource.component` | |
| `ResourceResource` | string | `resource.resource` | |
| `ResourceUid` | string | `resource.uid` | |
| `ResourceName` | string | `resource.name` | |
| `ResourceMetadata` | dynamic | `resource.metadata` | |
| `Metadata` | dynamic | `metadata` | no writer today; plumbed for the future |
| `CollectorNamespace` | string | AMA `PodNamespace` | **collector-stamped, trust boundary** |
| `CollectorPod` | string | AMA `PodName` | |
| `CollectorContainer` | string | AMA `ContainerName` | |
| `ClusterInstance` | string | **literal, per cluster's transform** | analogue of `openchoreo_cluster_instance` — see §5.4 |
| `RawRecord` | string | the producer's raw JSON line | preserves anything the schema drops |

`RawRecord` is the deliberate escape hatch for Azure's closed schema: a `schema_version` bump that
adds a field is not silently lost, and it backs free-text search the way OpenSearch's `log`
(`wildcard`) field does.

**Trust boundary, carried over from OpenObserve:** `Collector*` columns come from AMA and are
never read from the producer's line. `Producer` is the producer's own claim. They are separate
columns precisely so an investigator can compare them.

### 3.2 `TimeGenerated` must carry `event_time`

`TimeGenerated` is the partition column. The `azlogs` SDK's `Timespan` filters on it, and it is
what makes a bounded audit query cheap rather than a full scan of the retention window.

So the transform sets `TimeGenerated = todatetime(event_time)` and moves AMA's arrival time to
`CollectedTime`. This is a documented transformation pattern — Microsoft's own ASIM normalisation
sample does `project TimeGenerated = timestamp`, and the parse sample does
`project TimeGenerated = todatetime(d[0])`.

**This matches both reference modules, and for the same stated reason.** OpenObserve's fluent-bit
config does `Copy event_time _timestamp` with the comment *"OpenObserve takes a record's time from
`_timestamp`"*, and the module's PR description says it is *"so time windows and sorting are exact
to the microsecond."* OpenSearch keeps the two apart (`@timestamp` = collector read time,
`event_time` = request receipt) and ranges on `event_time`. Azure's partition column forces
OpenObserve's choice, so this plan follows it — with the clamp caveat below.

**To verify before implementation (§12):** that overriding `TimeGenerated` is honoured on the
`Microsoft-ContainerLogV2` stream specifically, not just on custom streams.

*Fallback if it is not:* every audit query must be double-bounded — a generous `Timespan` on
`TimeGenerated` (widened by a configurable collection-lag allowance) **plus** an exact
`where EventTime >= s and EventTime < e`. This changes the KQL for every endpoint, so settle it
first. It also breaks cheap sorting, because the contract's sort key would no longer be the
partition column.

### 3.2.1 Azure clamps `TimeGenerated` — keep `EventTime` as well

Azure Monitor will **silently overwrite** `TimeGenerated` with the actual received time if the
submitted value is more than **2 days older** than receipt or more than **1 day in the future**.

This is the exact Azure analogue of OpenObserve's `ZO_INGEST_ALLOWED_UPTO` coupling, which that
module's README calls out under Limitations: raise audit retention or ship to a foreign instance
without raising the ingest window and *late records are dropped*. Azure does not drop them — it
rewrites their timestamp, which is worse, because the record survives looking legitimate.

In steady state this never fires: collection lag is seconds. It fires on **replay** — fluent-bit
resuming from a buffer after a multi-day outage, or a backfill. An audit trail whose ordering
silently changes after an incident is precisely the wrong failure.

Mitigation, and the reason `EventTime` is a column in its own right rather than folded into
`TimeGenerated`:

- `EventTime` is written verbatim from the record and is never clamped.
- `TimeGenerated != EventTime` is therefore a **detector** for clamped records. Surface it in the
  README's troubleshooting section and consider it a candidate for an operator alert.
- The adapter keeps ranging on `TimeGenerated` (cheap), but returns `EventTime` as the record's
  `event_time` to the caller, so consumers see the true time even for a clamped record.

Accept the residual: a clamped record sorts and pages by its received time. That is a property of
the platform, not of this design, and it must be documented rather than hidden.

### 3.3 `ActorEntitlements` and ABAC

`actor.entitlements` is `map[string][]string` with keys that vary by subject kind — the OpenSearch
module handles this with a `dynamic_template` plus
`copy_to: actor.entitlement_values`, and filters on the copy.

Azure needs the same two-column shape, for two independent reasons:

1. A `dynamic` column is queryable but not efficiently filterable across arbitrary keys.
2. **ABAC conditions compare string columns only.** Any future row-level rule keyed on an
   entitlement is impossible against a `dynamic` column.

So the transform builds `ActorEntitlementValues` as a flattened, delimited string of all
entitlement values. `strcat_delim`, `split`, `array_concat` and `tostring` are all available in
transformations.

---

## 4. Deployment topology — the operator's choice, not the module's

**This section is documentation, not design.** How the operator isolates the audit table is their
security decision. The module's only obligation is to **not constrain it**, which costs exactly two
optional config fields (§4.3). There is no branching code path and no "Option A mode" in the adapter.

The module's actual storage decision was §2: audit gets its own table. Everything below is about
where that table lives and who may read it — and Azure answers that with resources and role
assignments the operator owns.

### Two topologies operators will pick between

**Separate audit workspace.** Read access is `Log Analytics Reader` scoped to that workspace.
Isolation is a resource boundary — nothing to reason about. Costs one more workspace and one more
`destinations.logAnalytics` entry per DCR; ingestion is charged per GB either way and audit volume
is small.

**Same workspace, separate table, ABAC condition.** A custom role with
`Microsoft.OperationalInsights/workspaces/query/read` (control) and
`…/workspaces/tables/data/read` (data), assigned **at the workspace** with a condition naming the
table:

```
Microsoft.OperationalInsights/workspaces/tables:name StringEquals 'OpenChoreoAuditLog_CL'
```

Assign at the workspace and put the table restriction in the condition — Microsoft's guidance, since
a condition on a resource-group-scoped assignment can silently misapply to workspaces lacking the
table. Fewer resources, but correctness now depends on the §4.1 trap being avoided and staying
avoided.

To the adapter these are indistinguishable: one workspace ID, one table name, one credential.
`QueryWorkspace` takes the workspace ID per call, so "same" and "different" is an argument value.

### 4.1 The additive-model trap — call it out in the README

Azure RBAC and ABAC are **additive with no deny**. From the granular RBAC docs:

> For granular RBAC conditions to take effect, you must remove any role assignments with higher
> access privileges. […] if you have two role assignments on the same scope, one set with a
> `*/read` action and the other with conditions that limit access to specific records, the
> resulting permission is the `*/read` action granting access to all logs in the scope.

**This does not apply to the adapter's own identity.** The adapter reads container logs *and* audit
records — that is what it is for — so there is no restriction anyone would want to place on it. It
holds one identity with read on both, and that is already least privilege for the component.

The trap applies to **every other principal**: the platform engineer with `Log Analytics Reader` at
the workspace, a CI service principal, another workload. If an operator wants those restricted to
`ContainerLogV2`, the condition goes on *their* role assignments, and any broader grant they hold
elsewhere defeats it. That is the operator's inventory problem, on principals the module never sees.

Also note: ABAC conditions are enforced **on queries only**. Workspace data export does not carry
them, and resource-context queries need the workspace set to *Require workspace permissions* or an
Azure read on the resource bypasses ABAC entirely. Both belong in the README's security section.

**Verification hook:** `LAQueryLogs.ConditionalDataAccess` records whether a query ran under an
applicable ABAC condition. That is the operator's proof the isolation is live, and it should be the
first item in the troubleshooting section.

### 4.2 What the module owes the operator

Nothing but flexibility, and honest documentation of the trap.

The README should state the §4.1 additive rule plainly, note that ABAC binds on **queries only**
(data export does not carry conditions, and resource-context queries need the workspace set to
*Require workspace permissions*), and point at `LAQueryLogs.ConditionalDataAccess` as the standing
proof that a condition is actually in force. Under the same-workspace topology that column is worth
monitoring rather than spot-checking, because the failure is silent: someone grants the identity a
broader role later and the condition stops binding with no error anywhere.

The chart's defaults should be the safe-and-boring ones — a separate `audit.workspace.id`, left
empty so nothing is assumed — but that is a default, not a constraint.

### 4.3 The module's entire obligation: one config field

`AUDIT_WORKSPACE_ID` — where to query. Set it to a dedicated workspace or to the logs workspace;
the module behaves identically. Empty means audit is not configured, and the endpoints answer 501
(§8.6).

The adapter keeps its **single existing managed identity**, which gains read on the audit table or
workspace alongside its existing `ContainerLogV2` grant. Nothing about credentials changes.

That is the whole story. One string, one extra role assignment for the operator.

---

## 5. Decision 4 — the ingestion path

Audit records are emitted by `openchoreo-api` and `observer` as container **stdout** lines
containing the marker `AUDIT-LOG` followed by JSON. Producers are identified by
namespace + container, matching the reference modules:

```yaml
producers:
  - producer: openchoreo-api
    namespace: openchoreo-control-plane
    container: api-server
  - producer: observer
    namespace: openchoreo-observability-plane
    container: observer
```

### 5.1 Path A (primary) — split in the Container Insights DCR

AMA already collects these lines into `ContainerLogV2`. Add a second `dataFlows` entry on the
`Microsoft-ContainerLogV2` stream whose `transformKql` selects audit lines, parses them, and routes
them to the audit table via `outputStream`; and add the **complementary** filter to the default flow
so audit lines leave `ContainerLogV2`.

```json
"dataFlows": [
  {
    "streams": ["Microsoft-KubeEvents", "Microsoft-KubePodInventory"],
    "destinations": ["ciworkspace"]
  },
  {
    "streams": ["Microsoft-ContainerLogV2"],
    "destinations": ["ciworkspace"],
    "transformKql": "source | where not(<AUDIT_PREDICATE>)"
  },
  {
    "streams": ["Microsoft-ContainerLogV2"],
    "destinations": ["auditworkspace"],
    "transformKql": "source | where <AUDIT_PREDICATE> | <PARSE>",
    "outputStream": "Custom-OpenChoreoAuditLog_CL"
  }
]
```

Why this path:

- **No new collector.** No fluent-bit DaemonSet, no otel collector. Consistent with the existing
  Azure module's stance that log shipping is out of chart scope.
- **No secret.** AMA's own identity does the ingesting; there is no client secret in the cluster.
- **Unforgeable provenance.** `PodNamespace` and `ContainerName` are stamped by AMA from the
  kubelet, not parsed from the log line — the same property the OpenSearch module gets from
  matching the kubelet-written log **filename**. A pod that prints a well-formed audit record
  cannot get it into the audit table.
- **Parse at ingest is possible.** `parse_json`, `substring`, `indexof`, `extract`, `tostring`,
  `todatetime`, `strcat_delim`, `split`, `iif`, `isnotempty` are all supported in transformations
  (verified against the supported-KQL reference). That buys typed columns, which buys cheap queries
  *and* row-level ABAC.

Constraints to respect: transformation max **15,360 characters**; `summarize`/`join` unsupported
(irrelevant here — transforms are per-record); the `parse` operator is capped at 10 columns per
statement, so use chained `extend` rather than one giant `parse`.

Note the DCR must use **individual streams**, not the `Microsoft-ContainerInsights-Group-Default`
group stream, or a transformation cannot be attached.

### 5.2 Path B (fallback) — fluent-bit → Logs Ingestion API

A module-owned fluent-bit DaemonSet with the `azure_logs_ingestion` output, writing to a
**module-owned** DCR (`"kind": "Direct"`), reusing the reference modules' `rewrite_tag` producer
gates verbatim.

When to use it: non-AKS / self-managed clusters with no AMA, or environments where the
Container Insights DCR cannot be edited or is reconciled back (§12).

Cost of this path — state it plainly in the README:

- The plugin authenticates with **`client_id` + `client_secret` only**; managed identity and
  workload identity are not supported (verified against the Fluent Bit output reference). That
  means an app-registration secret in a Kubernetes Secret — a real regression against the
  workload-identity posture the Azure modules otherwise hold.
- Requires a DCE or the DCR ingestion endpoint, plus `Monitoring Metrics Publisher` on the DCR.
- Respect the API limits: **1 MB max per call**, 64 KB max per field value (truncated beyond),
  2 GB/min and 12,000 requests/min per DCR. Enable `compress`.

A variant that keeps workload identity — fluent-bit `http` output → a module-owned forwarder using
`sdk/monitor/ingestion/azlogs` — is **out of scope for v1**. It trades a Secret for a bespoke write
path and a new authenticated ingress surface.

### 5.3 The two filters must be exact complements, generated from one source

OpenSearch does this atomically: `rewrite_tag`'s trailing `false` means "do not keep the record
under its original tag", so a record cannot land in both places. Path A splits that into two
independent `transformKql` strings. If they drift you get **silent duplication** (audit records in
`ContainerLogV2`, isolation broken) or **silent loss** (records in neither table).

Therefore: render both from a single Helm value / ARM parameter — one predicate, used once plainly
and once under `not(...)`. Never hand-author the two strings separately.

Two related guards:

**Marker semantics.** OpenSearch gates on `AUDIT-LOG(?!-)` — a negative lookahead excluding
`AUDIT-LOG-…` variants. KQL `has` is whole-term with `-` as a token separator, so
`LogMessage has 'AUDIT-LOG'` will very likely also match `AUDIT-LOG-FOO`. `matches regex` **is**
supported in transformations — use it to preserve the lookahead semantics rather than weakening the
gate.

**Null-row guard.** A non-audit line from `api-server` that happens to contain the marker would be
dropped from `ContainerLogV2` *and* land in the audit table as a row of nulls — strictly worse than
OpenSearch, where `ParseAuditRecord` errors and the record is skipped with a warning. So:

- in the transform, require structural validity (`isnotempty(tostring(_j.event_id))` and the other
  fields `ParseAuditRecord` treats as mandatory: `schema_version`, `event_id`, `actor.type`,
  `actor.id`, `result`, non-zero `event_time`);
- **and** filter `isnotempty(EventId)` at query time, because a transform deployed before this
  guard may already have written null rows.

Note the asymmetry this creates deliberately: a line that fails the structural guard is dropped
from the audit flow but must **still** be excluded from `ContainerLogV2` only if it matched the
predicate. Keep the predicate (provenance + marker) as the single shared discriminator and put the
structural validation *after* it, inside the audit flow only.

### 5.4 Multi-cluster: the DCR edit is per producer cluster

OpenChoreo's two audit producers routinely live in **different clusters** — `openchoreo-api` in the
control plane, `observer` in the observability plane. The OpenSearch README devotes a section to
this, and `openchoreo_cluster_instance` exists precisely because of it. Data-plane and
workflow-plane clusters have no producers and need nothing.

So Path A is not one DCR edit; it is **one edit per cluster that runs a producer**, all pointing at
the same audit workspace and table:

- Each cluster's Container Insights DCR gets the audit destination plus the two complementary data
  flows, with the predicate covering **only the producers that actually run in that cluster**.
  Rendering the control plane's predicate with `observer`'s namespace in it is harmless but
  misleading; keeping them accurate makes the allowlist reviewable per cluster, which is the point
  of it being a security boundary.
- The audit workspace is **cross-cluster shared state**. It lives in one subscription/region, and
  every producer cluster's DCR must be in the same region as its own workspace destination — check
  this early in a multi-region install, because a DCR and its Log Analytics destination must share a
  region.
- Each cluster's AMA identity needs no audit-specific grant: it already publishes through its own
  DCR. Only the DCR's destination changes.

**This also settles `ClusterInstance`.** Rather than deriving it from `_ResourceId` — a reserved
column name, with availability inside `source` unverified — stamp it as a **literal** in each
cluster's transform:

```kusto
| extend ClusterInstance = 'prod-control-plane'
```

That is the exact analogue of the fluent-bit `modify` filter that stamps
`openchoreo_cluster_instance` in the reference modules, needs no verification, and makes each
cluster's DCR edit self-documenting. The chart should surface it as
`audit.clusterInstance` and, like `fluentBitCustomizations.clusterInstance` in the OpenSearch
module, **fail the render if it is unset** — an unlabelled record in a multi-cluster audit trail
cannot be attributed after the fact.

### 5.5 Transformation billing — a genuine cost trap for Path A

From the transformations cost documentation, for Analytics tables:

> If a transformation reduces the ingested data by more than 50%, you're charged for the amount of
> filtered data above 50%. […] `[GB data dropped by transformation] - ([GB incoming data size] / 2)`

The audit data flow receives the **entire** `ContainerLogV2` stream and keeps a tiny fraction of
it. If that formula is applied per data flow, the charge approaches 50% of total container-log
volume — which would dwarf the cost of the audit data itself.

Mitigations, in preference order:

1. **Verify the billing granularity** (per data flow vs per DCR vs per destination) with Microsoft
   or against an actual bill before committing. This is the top item in §12.
2. If Microsoft Sentinel is enabled on the workspace, transformation filtering to Analytics tables
   is free regardless of volume — worth noting, not worth depending on.
3. If the charge is real and material, Path B becomes the primary path: fluent-bit filters at the
   edge, so the audit write path never carries non-audit volume.

Do not write the plan's cost section as if Path A is unconditionally cheaper. Its ingestion cost is
the open question, not a known quantity.

**Second, smaller cost item: `RawRecord` roughly doubles the ingested bytes per audit record.** From
the same doc: *"If a transformation increases the size of the incoming data, such as by adding a
calculated column, you're charged the standard ingestion rate for the extra data."* Keeping the raw
line alongside ~35 parsed columns is a deliberate trade — insurance against Azure's closed schema
silently dropping a field a future `schema_version` adds, plus the free-text search the reference
modules get from OpenSearch's `log` (`wildcard`) field. Audit volume is small in absolute terms, so
this is cheap; it is listed here so nobody meets it first on a bill. If it ever needs cutting, drop
`RawRecord` rather than any parsed column — the parsed columns are what queries and ABAC depend on.

---

## 6. Retention design

| Setting | Value | Meaning |
|---|---|---|
| `plan` | `Analytics` | **Pinned and validated.** Basic/Auxiliary force interactive retention to 30 days *and* break `azlogs` (`/query` vs `/search`) — the existing module already documents this constraint for `ContainerLogV2`. |
| `retentionInDays` | `365` (default; 4–730) | The window the adapter serves synchronously |
| `totalRetentionInDays` | operator's compliance period | One of 4–730, or 1095, 1460, 1826, 2191, 2556, 2922, 3288, 3653, 4018, **4383** (12 years) |

The important consequence, stated plainly: **at the 365-day default, interactive retention covers
the whole served window, so no search job is ever needed.** The two-tier complexity only appears if
an operator sets `totalRetentionInDays` beyond `retentionInDays`.

When they do, data between the two tiers is **operator-retrieved via a search job, out of adapter
scope** — and the reasons are hard, not stylistic:

- Search jobs need `workspaces/tables/write` + `workspaces/searchJobs/write` (Log Analytics
  Contributor). Granting that to a read-only audit adapter means granting control-plane write on
  tables. Non-starter.
- Search jobs do not support `summarize`, so the timeline and filter-values endpoints could not be
  served from one anyway.
- Granular RBAC is not yet enforced on search jobs, so a search job would bypass the very isolation
  this design rests on.

Document the operator procedure (`az monitor log-analytics workspace table search-job create`,
`_SRCH` result table, 100M record cap, 10 concurrent / 200 per day per workspace, delete the
`_SRCH` table when done) in the README as a forensic-retrieval runbook, not as a feature.

Retention mechanics worth encoding in the chart's comments:

- Shortening `totalRetentionInDays` has a **30-day grace** before deletion — a misconfiguration is
  recoverable.
- Increasing it applies to already-ingested, not-yet-expired data.
- Deleting a custom table on the Analytics plan does **not** delete the data, and reserves the name
  for 15 days. (On Auxiliary it *does* delete permanently — another reason to pin Analytics.)
- The `az`/PowerShell path caps `--total-retention-time` at 7 years; 12 years needs the REST API or
  portal. Provision retention via ARM/REST, not CLI, to avoid the cap.

---

## 7. Module boundary — extend the existing module, for now

### 7.1 The prerequisite that exists regardless

`observability-logs-azure-loganalytics`'s `Makefile` pins the **moving** upstream spec:

```make
SPEC := https://raw.githubusercontent.com/openchoreo/openchoreo/main/openapi/observability-logs-adapter-api.yaml
```

Its generated `gen/` implements 7 operations; the current spec declares 12. The next
`make openapi-codegen` will add `QueryPlatformLogs`, `QueryPlatformLogFilterValues`,
`QueryAuditLogs`, `QueryAuditLogFilterValues` and `QueryEvents` to `StrictServerInterface` and
**break that module's build**.

**Task 0, independent of everything else in this plan:** regenerate and add `501` stubs. The spec
sanctions this explicitly — *"Not Implemented. The adapter does not serve audit logs. Answer this
rather than…"* — and `…501JSONResponse` types are already generated, so the stubs are cheap. The
spec also notes the two audit operations are **separately declinable**: an adapter can serve
`/audit-logs/query` and answer 501 on filter-values.

### 7.2 The observer reaches audit through the logs adapter URL — verified

This constrains the design more than anything else in this plan, so it is stated before the
recommendation rather than after.

In the core repo, `observability.AuditLogsAdapter` is satisfied by **the same `*LogsAdapter`
struct** that serves container and platform logs
(`internal/observer/service/audit_logs_adapter.go`: `var _ observability.AuditLogsAdapter =
(*LogsAdapter)(nil)`), reaching whatever `LOGS_ADAPTER_URL` points at (default
`http://logs-adapter:9098`, 30s timeout).

**Audit deliberately has no adapter URL key of its own.** An earlier design revision *did* give it
one — the delivery plan records option D-C as *"the audit adapter URL is nonetheless its own key so
a later hardened backend is config, not code"* — and that was **reversed in the 2026-09-09
revision** in favour of a method set on the existing logs adapter. That is what shipped.

Consequence: **a separate audit Deployment is not reachable by the observer today.** The service the
observer resolves is the single generically-named `logs-adapter`, and two charts cannot both own it.

That is a constraint on packaging, not on security. The adapter is **one component with one job** —
serving the observer's log and audit queries — so it is correctly represented by **one managed
identity**, which gains read on the audit table alongside its existing `ContainerLogV2` grant.

Isolating audit from people who should not read it is achieved by *not granting them* the audit
workspace or table, on their own principals. That is operator role-assignment work on principals the
module never sees, and it is unaffected by how the adapter authenticates.

The audit endpoints must not inherit the logs module's `LoadConfig` requirements as
*additional* mandatory config: adding audit must not make `AUDIT_WORKSPACE_ID` required for
installs that only want logs. Gate the audit config on presence, and answer 501 when absent (§8.6).

---

## 8. Go implementation

Per §7.2, v1 lands **inside `observability-logs-azure-loganalytics`**. New files only — following
the OpenSearch module's two-file decomposition (handlers + query builder) rather than OpenObserve's
four, plus a column registry Azure needs and neither reference does:

```
observability-logs-azure-loganalytics/
├── internal/
│   ├── audit_handlers.go                  # NEW: contract clamps, filter map, gen conversions
│   ├── audit_handlers_test.go             # NEW
│   └── loganalytics/
│       ├── audit_queries.go               # NEW: KQL builders
│       ├── audit_queries_test.go          # NEW
│       ├── audit_types.go                 # NEW: internal AuditRecord + row→record mapping
│       ├── audit_types_test.go            # NEW
│       └── audit_columns.go               # NEW: the single column-name registry
```

`audit_columns.go` exists because Azure's schema is closed: the column names appear in the table
definition, the DCR transform, the KQL builders and the filter map, and a typo in any one of them
fails **silently** (a dropped value at ingest, or a filter that matches nothing). One Go constant
per column, referenced everywhere, with a test asserting the filter map's range is exactly the
registry — the analogue of the OpenSearch module routing its index prefix through one Helm helper so
collector and adapter "cannot drift".

Reuses as-is: `internal/loganalytics/client.go` (add a `QueryAudit` path and `buildColumnIndex`/
`rowString`/`rowTime`, which already tolerate short, nil and non-string cells), `internal/server.go`,
the access-log middleware, and the `OBS-V1-L-AZURE-*` error-code prefix.

### 8.1 Dependencies

**No new dependency is required.** The module already has everything the read path needs, on the
version line shared byte-for-byte across all three Azure modules:

```
github.com/Azure/azure-sdk-for-go/sdk/azcore                   v1.23.1
github.com/Azure/azure-sdk-for-go/sdk/azidentity                v1.14.1
github.com/Azure/azure-sdk-for-go/sdk/monitor/query/azlogs      v1.2.0   # query, not ingestion
github.com/Azure/azure-sdk-for-go/sdk/resourcemanager/monitor/armmonitor v0.13.0
```

Explicitly **not** added:

- `sdk/monitor/ingestion/azlogs` — the module reads; it never writes. Adding it would only be
  needed for §5.2's rejected workload-identity forwarder variant.
- `armoperationalinsights` — needed **only** if §8.5's boot validation reads table properties over
  ARM. Weigh it against the extra ARM read permission it requires; the `Ping` check alone needs none.

Note `azlogs` here is `sdk/monitor/query/azlogs`, the current client, not the older
`sdk/monitor/azquery`. They have similar names and a migration guide between them; the query body
type is `azlogs.QueryBody{Query, Timespan}`.

Auth: the existing `azidentity.NewDefaultAzureCredential(nil)`, unchanged and reused. One credential
already serves both the query and ARM audiences, and it serves the audit workspace too.

### 8.2 Config

Additive to the existing `Config`. **All optional** — audit must not become mandatory config for
installs that only want logs (§7.3):

| Env | Default | Meaning |
|---|---|---|
| `AUDIT_WORKSPACE_ID` | `""` | Audit workspace `customerId` GUID. Empty ⇒ audit not configured ⇒ **501** (§8.6). May equal the logs workspace ID — the module does not care (§4.3). |
| `AUDIT_TABLE_NAME` | `OpenChoreoAuditLog_CL` | |
| `AUDIT_QUERY_TIMEOUT` | inherits `QUERY_TIMEOUT` | |
| `AUDIT_EXPECTED_RETENTION_DAYS` | `365` | Boot validation only (§8.5) |

**One `azlogs` client, one credential.** `QueryWorkspace` takes the workspace ID per call, so the
same client serves the logs workspace and the audit workspace whether or not they are the same
workspace. No second credential, no second client.

`AUDIT_TABLE_NAME` must be validated at startup against `^[A-Za-z][A-Za-z0-9_]*_CL$` — the direct
analogue of the OpenSearch module's index-prefix validator, and for the same reason: a bad value
means the adapter queries a table nothing writes to and answers an empty 200.

### 8.3 KQL query building

Differences from the existing logs module's builders that must not be copy-pasted wrong:

- **`top N by` not `order by | take N`.** KQL `take` is non-deterministic; the existing
  `BuildComponentLogsKQL` uses `order by … | take N`, which is already subtly wrong there and would
  be wrong here under a `sortOrder` + `limit` contract.
- **Stable tiebreaker.** OpenSearch sorts `event_time` then `event_id`; OpenObserve sorts
  `_timestamp {dir}, event_id {dir}` with the note *"`event_id` breaks ties so paging by the last
  record's time is stable"*. Mirror it: `| top <limit> by TimeGenerated <dir>, EventId <dir>`.
  Paging is by **time window, not a continuation token** — the caller re-queries with the window
  closed up to the last record it received, so the adapter keeps no per-scroll state and needs
  nothing like a point-in-time from Azure.
- **Half-open window** `TimeGenerated >= start and TimeGenerated < end`, matching the reference
  modules' `gte`/`lt`.
- **Injection defence.** The existing `kqlString` quoter escapes `\`, `"`, `\n`, `\r`. It must be
  extended for `in (…)` list semantics across the ~20 audit filter fields, and the filter-name →
  column mapping must be a **closed map** (`map[gen.AuditLogFilterValuesRequestFilter]string`) with
  unknown filters rejected — never string-concatenated from request input.
- **Do not copy `resolveLogLevel`.** It exists in the logs module because AMA marks all stderr
  output `error`. Audit records carry their own `result` and `category`. Worth a note in the README
  so nobody reintroduces it — and as a general caution about trusting AMA-derived columns.

### 8.4 One round-trip for records + exact total + timeline

The contract requires a total that counts **fully**, past the page size (OpenSearch sets
`track_total_hits: true`). The logs module cannot do this — it returns `TotalCount: len(entries)`
and clamps at 1000 via `capTotal`. Copying that would silently violate the audit contract.

A single Query API call can return multiple tables. Use a `let`-bound base:

```kusto
let Base = materialize(
  <table>
  | where TimeGenerated >= datetime(...) and TimeGenerated < datetime(...)
  | where isnotempty(EventId)
  | <filters>
);
Base | top <limit> by TimeGenerated desc, EventId desc;
Base | summarize Total = count();
Base | summarize Count = count() by Bucket = bin(TimeGenerated, <interval>), Result;
```

Read `Results.Tables[0..2]`. Three tabular statements, one HTTP round-trip, one rate-limit slot
(the Query API allows only 5 concurrent queries per user and 200 requests / 30 s).

**`materialize()` is not optional here.** KQL re-evaluates a `let`-bound tabular expression once per
reference, so without it these three statements scan the filtered set three times — the exact cost
this single-round-trip shape exists to avoid. Caveat to respect: `materialize()` caches into a bound
memory budget, so the filtered set must stay bounded, which it is — the time window plus
`isnotempty(EventId)` plus the request's filters. If a caller's window is wide enough to blow the
cache the query fails loudly rather than silently degrading, which is the right failure.

Note `top … by` rather than `order by … | take` — per §8.3, and it takes the multi-key sort
directly.

Timeline interval resolution: port `ResolveTimelineInterval`'s **coarsening** behaviour
(`m`→`h`→`d`→`w`, then +1w) with the 500-bucket cap and ceiling arithmetic for the partial bucket —
it coarsens rather than rejecting, and reports the width actually used. `bin()` is available.
Zero-filling empty buckets (`min_doc_count: 0` + `extended_bounds` in OpenSearch) has **no KQL
equivalent inside `summarize`** — fill the gaps in Go from the requested window and interval.

Filter values: `Base | summarize Count = count() by <column> | top N by Count desc, <column> asc`,
dropping empty keys, plus `dcount` for the total-distinct value. Per the contract, drop the named
filter's own selections so the picker offers alternatives (`clearAuditFilter`'s behaviour).

### 8.5 Boot validation, mirroring `Ping`/`VerifyActionGroup`

The existing Azure modules fail fast at boot. Do the same, and make it *specifically* about the
properties this plan is built on:

1. `Ping` — `<table> | take 1` — proves the table exists and the identity can read it.
2. If ARM read permission is available, `GET .../tables/{name}` and **warn loudly** (do not exit) if
   `plan != "Analytics"` or `retentionInDays < EXPECTED_RETENTION_DAYS`. A silently-Basic table or a
   silently-30-day retention is exactly the misconfiguration that destroys audit records with no
   error, and it is the Azure analogue of the OpenSearch module's "index created before its
   template" trap.

### 8.6 Empty results vs 501 — decide deliberately

The OpenSearch module **never** returns 501: with audit collection off, the index is absent,
`IgnoreUnavailable: true` swallows it and the endpoint answers an **empty 200**. The spec's stated
preference is the opposite: *"answer 501, never an empty 200."*

For Azure, a missing table is a hard query error, not a silent empty — which is an opportunity to
get this right rather than inherit the wart:

- audit collection not configured (`AUDIT_TABLE_NAME` unset, or boot validation found no table)
  → **501** on both audit operations, with a message naming the missing table;
- table present, window genuinely empty → **200** with zero records.

Ship `/audit-logs/query` in v1 and answer **501** on `/audit-logs/filter-values` if the aggregation
work slips — the spec explicitly permits declining them separately.

---

## 9. Helm chart

An additive `audit:` block in the **existing** `observability-logs-azure-loganalytics` chart,
following its established conventions — validated values with `_helpers.tpl` failing the render on
bad input, and a `checksum/config` annotation so the adapter restarts itself on change.

Bump `helm/Chart.yaml` in the same PR: the repo's PR check fails a chart change without a version
bump.

```yaml
audit:
  # Only the READ path is gated here. Retention and the table exist whether or not this
  # is set, because — as the OpenSearch module learned — shaping the destination after
  # records have been written does not repair what was already written.
  enabled: false

  # The table the adapter reads. The DCR that writes it must name the same table:
  # they cannot be derived from each other, so both come from this value.
  tableName: OpenChoreoAuditLog_CL

  # Separate workspace (recommended) or the shared logs workspace (cost-optimised).
  workspace:
    id: ""                  # customerId GUID, required
    resourceId: ""          # ARM ID, for the provisioning docs and boot validation

  # Stamped as a literal into this cluster's DCR transform. Required: an unlabelled
  # record in a multi-cluster trail cannot be attributed after the fact.
  clusterInstance: ""

  retention:
    interactiveDays: 365    # 4-730; the window the adapter serves
    totalDays: 365          # 4-730 or 1095…4383

  # Security boundary, not ordinary config: each entry grants a workload the right to
  # write into the audit trail. Rendered into BOTH DCR data flow filters.
  producers:
    - producer: openchoreo-api
      namespace: openchoreo-control-plane
      container: api-server
    - producer: observer
      namespace: openchoreo-observability-plane
      container: observer

# existing keys unchanged; adapter.serviceAccount.annotations still carries the
# workload-identity client ID. Consider promoting it to a first-class
# adapter.serviceAccount.clientId as the metrics module already does — the logs
# module currently makes operators hand-write the annotation.
```

Templates touched: `adapter/configmap.yaml` (the five new env vars), `_helpers.tpl` +
`validate.yaml` (the new validators below). No new Deployment, Service or ServiceAccount — the audit
identity federates to the **existing** ServiceAccount (§7.3), so nothing changes in
`serviceaccount.yaml` either.

Two things the chart should carry beyond the adapter:

- **Rendered DCR fragment.** A ConfigMap (or a `helm template`-able output) containing the
  ready-to-apply `dataFlows` JSON with both complementary filters generated from `audit.producers`.
  The operator applies it; the chart does not mutate the AKS-managed DCR (§10). This is what keeps
  §5.3's two filters from drifting.
- **No collector subchart** on Path A. Path B would add fluent-bit, gated behind
  `audit.ingestion.mode: fluentbit`.

The `_helpers.tpl` validator should fail the render, when `audit.enabled`, on: empty
`workspace.id`; a `tableName` not matching `^[A-Za-z][A-Za-z0-9_]*_CL$`;
`retention.interactiveDays` outside 4–730; `retention.totalDays` not in the allowed set
(4–730, 1095, 1460, 1826, 2191, 2556, 2922, 3288, 3653, 4018, 4383);
`retention.totalDays < interactiveDays`; an empty `producers` list; an unset `clusterInstance`
(§5.4).

Follow the OpenSearch module's pattern of routing the table name through **one** named helper used
by every consumer, so the adapter's `AUDIT_TABLE_NAME` and the rendered DCR fragment cannot drift —
the identical concern its `auditIndexPrefix` helper exists to solve, with the identical failure if
it drifts (the adapter queries a table nothing writes to and answers an empty 200).

---

## 10. Provisioning — a documented operator procedure, not a Helm hook

The OpenSearch module provisions its index template and ISM policy from a post-install Job, because
it owns the OpenSearch cluster. **The Azure module owns none of its Azure resources**, and the
existing Azure module is explicit that log shipping is out of chart scope and has the operator
apply `container-azm-ms-agentconfig` by hand. Follow that precedent.

Deliver as ARM/Bicep templates plus `az` commands in the README:

1. **Create the audit workspace**, or reuse the logs workspace — see §4 for the tradeoff.
2. **Create the custom table** with the §3.1 schema, `plan: Analytics`. Via
   `PUT .../workspaces/{ws}/tables/{name}_CL?api-version=2025-07-01`.
3. **Set retention** —
   `PATCH .../tables/{name}?api-version=2025-07-01` with
   `{"properties":{"retentionInDays":365,"totalRetentionInDays":365}}`. Use REST/ARM, not CLI
   (7-year cap).
4. **Edit the Container Insights DCR — once per cluster that runs a producer** (§5.4). Switch
   `dataSources.extensions.streams` from `Microsoft-ContainerInsights-Group-Default` to individual
   streams, add the audit workspace as a destination, add the two complementary data flows from
   §5.1 with a predicate naming only that cluster's producers, and stamp that cluster's
   `ClusterInstance` literal. Control plane and observability plane are normally two separate
   clusters; data-plane and workflow-plane clusters need nothing.
5. **Role assignments:**
   - the **existing adapter identity** → `Log Analytics Reader` on the audit workspace, in addition
     to its current grant. If audit shares the logs workspace it already has read; nothing to do.
     No new identity, no new federated credential.
   - **everyone else** → this is where the isolation actually lives. Principals who should read
     container logs but not audit must not hold a grant that reaches the audit workspace or table
     (§4.1). Review existing workspace-scoped assignments before enabling audit collection.
   - **Do not** assign `Data Purger` to anything. Note in the README that it is the only role that
     can erase audit records, and that leaving it unassigned is part of the control.
   - remove any broader `*/read` assignment the audit identity holds (§4.1).
6. **Federated credential** on the managed identity for
   `system:serviceaccount:<ns>:audit-adapter-azure-loganalytics`.

Also document the AMA prerequisite the existing module's troubleshooting section already learned the
hard way: `container-azm-ms-agentconfig` must enable `metadata_collection`, and AMA must actually be
collecting stdout from `openchoreo-control-plane` and `openchoreo-observability-plane` — otherwise
the audit table stays empty with no error anywhere.

---

## 11. Phased delivery

| Phase | Scope | Exit criteria |
|---|---|---|
| **0** | Regenerate `gen/` in `observability-logs-azure-loganalytics`; add 501 stubs for all 5 new operations | That module builds against current `main` spec |
| **1** | Verify §12's open questions in a real subscription | `TimeGenerated` override, DCR reconciliation and transformation billing all answered |
| **2** | ARM/Bicep: workspace, table, retention, per-cluster DCR data flows, role assignments. No Go. | Audit records land in the table with `TimeGenerated == event_time`; none remain in `ContainerLogV2`; records from **two** producer clusters land in one table, distinguishable by `ClusterInstance`; a logs-only reader is **denied** the audit workspace |
| **3** | `/audit-logs/query` + column registry + boot validation; 501 on filter-values | Contract tests pass for records, exact total, paging, sort tiebreaker |
| **4** | Timeline aggregation + `/audit-logs/filter-values` | Interval coarsening, 500-bucket cap, zero-filled buckets, value ordering |
| **5** | `audit:` values block, README (security + retention + troubleshooting + search-job runbook) | Clean install on AKS; a logs-only reader is denied the audit workspace |
| **6** | Path B (fluent-bit) behind `audit.ingestion.mode`, **only if** phase 1 shows Path A is unviable | Non-AKS install works |
*(No upstream phase. The earlier draft had one; §7.3 removed the need for it.)*

Phase 2 before phase 3 is deliberate: the storage and access design is the risky part, and it is
verifiable with `az` and the portal alone. Do not write Go against an unproven table.

Phase 0 is not optional and is not really part of this feature — that module's build breaks on the
next `make openapi-codegen` regardless of whether audit is ever implemented.

---

## 12. Open questions — verify, do not assume

Ranked by how much a wrong answer costs.

1. **Transformation billing granularity (§5.5).** Is the >50%-dropped charge computed per data flow?
   If yes, Path A's ingestion cost is roughly 50% of *all* container-log volume and Path B becomes
   primary. Check with Microsoft or against a real bill.
2. **Does the AKS Container Insights addon reconcile away a hand-added data flow** on cluster or
   addon upgrade? If it does, Path A is not durable and Path B is primary. This is the main
   operational risk of the recommended path.
3. **Can a transformation override `TimeGenerated` on the `Microsoft-ContainerLogV2` stream?**
   (§3.2.) Documented for transformations generally; confirm on this stream. Determines whether every
   query needs double-bounding.
4. **Does AMA collect stdout from the two producer namespaces** under the documented
   `container-azm-ms-agentconfig`? No `exclude_namespaces` appears in the existing module, so
   probably yes — but an empty audit table is silent, so verify rather than infer.
5. **ABAC condition size** against ~30 columns and a table-name expression, versus Azure ABAC's
   per-condition expression count and KB limits. Only relevant to the same-workspace topology.
6. Exact KQL `matches regex` behaviour for the `AUDIT-LOG(?!-)` lookahead inside a transformation —
   noting that `parse kind=regex` in transformations requires a **full-string** match, unlike
   Log Analytics queries. Test it; do not port the regex blind.

### Answered during planning — recorded so they are not re-litigated

- **Does the observer support a distinct audit-adapter URL?** **No.** `AuditLogsAdapter` is a method
  set on the same `*LogsAdapter`, reached via `LOGS_ADAPTER_URL`; a dedicated key was designed and
  then deliberately reversed on 2026-09-09. **This turned out not to block anything** — see the next
  item.
- **Does the adapter need a second Azure identity to isolate audit?** **No** — and an earlier draft
  of this plan wrongly said it did. The adapter reads container logs *and* audit records by design,
  so no restriction on its identity is ever wanted. One identity with read on both is already least
  privilege for that component. Audit isolation is achieved by *not granting* the audit
  workspace/table to other principals, which is operator role-assignment work the module never
  touches (§4.1). Entra Workload ID does support many-to-one federation if a future split ever needs
  it, but nothing here does.
- **Can a DCR transformation set `TimeGenerated` from a parsed field?** Yes — documented pattern
  (`project TimeGenerated = todatetime(...)`). Open only for the `Microsoft-ContainerLogV2` stream
  specifically (question 3).
- **Is `parse_json` available in transformations?** Yes, along with `substring`, `indexof`,
  `extract`, `split`, `strcat_delim`, `todatetime`, `iif`, `isnotempty`, `matches regex`. `summarize`
  and `join` are not, which is irrelevant for per-record transforms.
- **Can the read path serve data past interactive retention?** No, and it must not try (§6):
  search jobs need `tables/write` + `searchJobs/write`, lack `summarize`, and are not covered by
  granular RBAC.
- **Does Azure clamp a back-dated `TimeGenerated`?** Yes — >2 days old or >1 day future is silently
  replaced with the received time (§3.2.1). Hence the separate `EventTime` column.
- **Do the reference modules offer any per-index/per-stream RBAC to copy?** No. Neither OpenSearch
  nor OpenObserve creates roles, users or service accounts; OpenSearch uses one shared `admin`
  credential for write, setup and read, and OpenObserve's only separation is a second *instance*.
  Azure is the first backend where storage-level access control is real — so there is no precedent
  to copy, and the design must be argued from the Azure primitives rather than ported.

---

## 13. Testing

- **Unit, Go:** KQL builders (every filter → column, half-open bounds, sort tiebreaker, `in (…)`
  quoting and injection attempts, unknown filter rejected, limit/maxValues clamps); interval
  coarsening and bucket cap; row→record mapping including short rows, nil and non-string cells;
  Go-side zero-filling; `isnotempty(EventId)` skipping null rows. Port the reference modules' 38
  audit test cases as the baseline — they encode contract requirements that are easy to miss.
- **Integration, needs Azure:** ingest a known audit record end to end; assert
  `TimeGenerated == event_time`; assert the same record is **absent** from `ContainerLogV2`; assert
  a workspace-wide reader identity is **denied** the audit table and that
  `LAQueryLogs.ConditionalDataAccess` reflects the condition; assert a spoofed audit line from a
  non-allowlisted pod lands in `ContainerLogV2` and **not** in the audit table.
- **Retention:** set `retentionInDays` below `totalRetentionInDays` and confirm the adapter's served
  window matches interactive retention and degrades honestly past it.

- **The self-audit loop.** Querying the audit trail emits a `category: access` record from the
  `observer` container — an allowlisted producer — so that record must itself land in the audit
  table. Assert it end to end: read the trail, then find the read. This is the property that
  distinguishes an audit trail from a log viewer, and it is what `CategoryAccess`'s comment in
  `types.go` says the category exists for: *"for the ones where knowing who looked is part of the
  point, so an investigator can filter disclosure apart from change."* It also exercises the
  producer allowlist for `observer` specifically, which the two-cluster topology makes easy to get
  wrong.
- **Clamp detection.** Ingest a record with an `event_time` more than two days old and assert
  `TimeGenerated != EventTime` while `EventTime` still holds the true value (§3.2.1).

Two tests carry most of the value. The **spoofing** test is the property the whole
producer-allowlist design exists to provide, and the only one that cannot be checked by reading
config. The **denied-reader** test is the property the user asked for — that logs access does not
confer audit access — and it is the one a misplaced role assignment silently revokes.
