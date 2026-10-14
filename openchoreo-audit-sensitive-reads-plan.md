# Audit Coverage: Sensitive Reads, and Recording What They Disclosed

**Status:** Approved for implementation on 2026-10-06. Nothing is implemented yet.
**Repo:** `openchoreo`, plus `community-modules` (OpenSearch and OpenObserve; Azure Log Analytics is handled with that module's own work) and `backstage-plugins` (display) · **Date:** 2026-10-05, revised 2026-10-06
**Builds on:** [#4703](https://github.com/openchoreo/openchoreo/pull/4703) (reads classified in `RESTExemptions`), `openchoreo-audit-read-classification-plan.md`
**Related:**
- `openchoreo-audit-sensitive-reads-market-research.md`: how CloudTrail, GCP, Kubernetes, Vault, Snowflake, OCSF and others record reads
- `openchoreo-audit-returned-items-filtering.md`: how each storage module filters on a returned item
- `audit-sensitive-reads-discussion-update.md`: the community update for discussion #4396

---

## 1. The problem

Every read is exempt from audit. Only two GET-shaped operations are audited: `Exec` and `Wirelogs`
(`internal/openchoreo-api/audit/exec_wirelogs.go:25-34`). One POST read is audited: `QueryAuditLogs`
(`tools/auditgen/config_observer.go:51-56`). Even that one ships unpublished: the
observability-plane chart's default audit policy sets `QueryAuditLogs` to `publish: false`
(`install/helm/openchoreo-observability-plane/values.yaml`, under `audit.policies`). Audit logging
as a whole is also off by default (`enabled: false`). That leaves two gaps.

**Gap 1: some reads disclose things worth recording, and none of them are audited.** The clearest
case is that `GetSecret` and `ListSecrets` both return plaintext secret values. `ListSecrets` returns
them for every item on the page (`services/secret/service.go:342`, schema at
`openapi/openchoreo-api.yaml:13188`). One `ListSecrets` call can hand out up to 100 secrets with no
audit record.

**Gap 2: an audited read records that something was read, not what was read.** `QueryAuditLogs` is
the only read audited for disclosure, and its record has no trace of the filter, the time range or
the result count. The handler sets nothing (`internal/observer/api/handlers/auditlogs.go:18-51`), and
the authz check sets an empty hierarchy. `http.path` deliberately drops the query string, and a POST
body is never captured. "Admin X read the audit trail at 10:02" does not tell an investigator whose
activity X looked at.

Simply moving reads out of `RESTExemptions` fixes gap 1 and does nothing for gap 2. This plan covers
both.

## 2. Inventory: which reads are sensitive

The full sweep covered all ~100 openchoreo-api reads and all observer reads. The tables below list
only High and Medium. Everything else is metadata, schema, health or platform templates and stays
exempt as `reasonRead`.

### 2.1 High: the response carries credentials or live data

| Operation | Surface | What the response carries | Evidence |
|---|---|---|---|
| `GetSecret` | REST only | Full `corev1.Secret` with `data` values, read live from the target plane | `services/secret/service.go:237-263` |
| `ListSecrets` | REST only | The same, for every item on the page (up to 100) | `services/secret/service.go:342`; `openchoreo-api.yaml:6174-6176` |
| `GetReleaseBindingK8sResourceLogs` | REST, MCP `get_resource_logs` | Live container logs through the cluster gateway; no tail or byte limit | `services/k8sresources/service.go:180-240` |
| `GetWorkflowRunLogs` | REST, MCP `get_workflow_run_logs` | Live build logs for every workflow container; build logs routinely echo tokens | `services/workflowrun/service.go:557-601` |
| `QueryLogs` (observer) | REST, MCP `query_component_logs` / `query_workflow_logs` | Stored application and workflow logs | `openapi/observer-api.yaml:1575-1609` |
| `GetPlatformLogs` (observer) | REST, MCP `query_platform_logs` | Raw pod logs across every collected namespace, control plane included | `openapi/observer-api.yaml:389-418` |
| `GetSpanDetailsForTrace` (observer) | REST, MCP `get_span_details` | A full span, always with `attributes` and `resourceAttributes`: whatever the instrumentation records, such as URLs with query strings, database statements and user ids | `openapi/observer-api.yaml:3034-3088` |
| `QuerySpansForTrace` (observer) | REST, MCP `query_trace_spans` | Spans for a trace. Attributes are included **only when `includeAttributes: true`** (default false); the handler strips them otherwise. | `internal/observer/api/handlers/traces.go:313-345`; `openapi/observer-api.yaml:2974` |

Trace reads were added on 2026-10-08. They carry the same class of data as log reads, but only in
span attributes, so only the two operations that can return attributes are audited.
`QueryTraces` (MCP `query_traces`) returns trace summaries (`TraceInfo`: names, span count, timings)
with no attribute fields, and stays exempt.

`remote-connect:resolve` also discloses secret values, indirectly, but it is deferred (§8).

### 2.2 Medium: config that can embed secrets, or authz posture

| Operation | What can leak | Evidence |
|---|---|---|
| `GetReleaseBindingK8sResourceTree` | Live objects are returned with only core Secrets sanitized, so **ConfigMap `data` is returned**. The full `RenderedRelease` is attached unsanitized. | `services/k8sresources/service.go:883-908`; `handlers/releasebinding_k8sresources.go:49-56` |
| `Get/ListObservabilityAlertsNotificationChannel(s)` | Webhook URLs (a Slack URL is a credential by itself), inline header `value`, email recipients | `api/v1alpha1/observabilityalertsnotificationchannel_types.go:119,133,138` |
| `Get/List*RoleBinding(s)` | Entitlement values (user `sub`/email), the shape of who holds what | `authzrolebinding_types.go:51-61` |
| `Evaluates` | Accepts another subject's `subject_context`, so it can reveal another principal's permissions (see §2.3) | `handlers/authz.go:85-97` |

**None of the Medium tier is in this phase.** This phase delivers the eight High-tier operations plus
`QueryAuditLogs` (§6). D-2 records what to do with the Medium tier later.

**Workload, release binding, component release and resource release binding reads are not
sensitive, and were taken off this list on 2026-10-06.** They return secret *references*, never
values:

- `valueFrom.secretKeyRef` comes back as `{name, key}`, and the value is resolved in the data plane at
  runtime.
- Resource outputs carry a `SecretKeyRef` "for sensitive credentials; the underlying value never
  leaves the data plane" (`api/v1alpha1/resourcereleasebinding_types.go:107-111`).

The literal `value` fields (`workload_types.go:23,52`; `resourcereleasebinding_types.go:101-105`) are
defined as non-sensitive configuration. A credential typed into one is a misuse of the data model, not
a disclosure by these APIs, and read auditing doesn't try to cover it.

### 2.3 Findings that are not audit work

> **Keep this section private.** Remove it, or raise it through `SECURITY.md` first, before any part
> of this plan is posted to a public discussion or issue.

The sweep turned up three things an audit record would only witness. Before anything is fixed, each
needs a maintainer to confirm whether it is intended:

1. **The log permissions differ between REST and MCP.** REST `GetReleaseBindingK8sResourceLogs`
   checks `releasebinding:view` (`services/k8sresources/service_authz.go:57-61`). MCP
   `get_resource_logs`, which returns the same logs, requires `logs:view` (`pkg/mcp/tools/pe.go:477`).
   Confirm which one is meant. If it is `logs:view`, a subject without it can still read logs over
   REST.
2. **The resource tree returns ConfigMap data and the raw `RenderedRelease`** (above). The
   sanitization covers only core Secrets.
3. **`Evaluates` takes any `subject_context` the caller supplies, and nothing gates it.**
   - **What the code does:**
     - The handler uses the request's subject over the caller's (`handlers/authz.go:85-97`).
     - The service wrapper is ungated "because requests contain their own subject context"
       (`services/authz/service_authz.go:274-277`).
     - MCP `evaluate_authz` declares `authzrolebinding:view` (`pkg/mcp/tools/scoped_authz.go:271`), but
       that check is in the tool filter, which a client can switch off per request with
       `?filterByAuthz=false` (`pkg/mcp/server.go:39`). The filter's design assumes the service layer
       enforces authz anyway, and here it doesn't.
   - **What it exposes:**
     - A caller can submit arbitrary entitlement values and read back allow or deny.
     - On an allow, the reason names the entitlement value that matched (`internal/authz/casbin/pdp.go:464`).
     - Probing with candidate groups or user ids therefore maps who holds which permission where,
       without `authzrolebinding:view`.
     - It grants nothing.
   - **Why it may be by design:** `subject_context` was required until #3885 (`543540a40`) made it
     optional.
   - **Why it may not be needed:** the portal backend, the main caller, always sends the user's own
     subject, read from `/authz/profile` (`backstage-plugins/.../AuthzService.ts:69-89`). Nothing seen
     in this sweep needs to evaluate a subject other than the caller.

Fixing 1–3 reduces what the Medium tier discloses. For high-volume portal reads, this plan prefers
reducing disclosure to auditing it (§7, D-2).

## 3. What "what was read" means

No one field set fits every read. The record should hold **enough to reconstruct the disclosure, never
the content**. What that takes depends on whether the source can be replayed:

| Read shape | Examples | The source replays? | So record |
|---|---|---|---|
| **Single mutable object** | `GetSecret` | No, values change on rotation | Identity. The event time places the read against the audited updates around it. |
| **List of mutable objects** | `ListSecrets` | No | Count, plus each item's identity |
| **Query over an append-only store** | `QueryAuditLogs`, `QueryLogs`, `GetPlatformLogs` | Yes, within retention | The **query parameters as sent** (scope, time window, filters, search), plus count |
| **Live fetch from a cluster** | Pod logs, workflow logs | Partly: pod logs vanish with the pod | The fetch parameters (pod, container, task, since), plus count |

For live pod logs, only the *requested* window is kept, not the window actually returned. A pod that
has since gone can't be replayed exactly. That's an accepted trade-off: a returned-window field would
only apply to log reads (§4).

Two rules apply to every shape.

**Request versus returned.** On a denial or failure nothing was disclosed, but the request still
matters ("who tried to read `prod-db-creds`"). The record splits the two: `request` is always
present, and `returned` is present only when `result: success`. The emitter enforces this. It drops
`returned` on any non-success result, so a handler bug cannot claim a disclosure that never happened.

**What is never recorded:** secret values, log lines, span attributes, ConfigMap data, or any response
body. The audit trail must not become a second copy of what it guards.

### 3.1 Secret reads record no version (decided 2026-10-08)

An earlier draft recorded the plane Secret's `resourceVersion` with each secret read. It was dropped
after review on discussion #4396.

- **It can't be traced back to the content.** Kubernetes keeps no history of an object, so once the
  Secret changes, nothing says what an older version held. It's a label for a state, not a
  fingerprint of the values.
- **It isn't Vault's version either, and can't be mapped to it.** `GetSecret` and `ListSecrets` read
  the plane Kubernetes Secret, never the store. ESO pushes each key to KV v2 as its own property
  write and records no store version anywhere: `PushSecret.status.syncedResourceVersion` is the
  PushSecret's own version.
- **Timestamps cover what the version could do.** Secret creates and updates are audited, so a read is
  placed against the updates around it by `event_time`.

Identifying the value itself needs a keyed hash, which is tracked as a follow-up (D-6).

## 4. Schema: where the details live

This is the main design decision.

**Option A (rejected): put it in `metadata`.** `Event.Metadata` is plumbed end to end and has no writer
(`internal/server/middleware/audit/types.go:245-248`), so this costs no schema change. But the
OpenSearch module stores `metadata` **unindexed** (opensearch-module plan, D-2). "Who read secret X"
could be shown on a record but not searched for. Moving the data out later would break every saved
query, so this option cannot be adopted as a first step and fixed afterwards.

**Option B (adopted): first-class `request` and `returned` groups.** It is a minor version bump,
`1.0` to `1.1`, under the CloudTrail rule already adopted (an addition is minor).

**How the names were settled (2026-10-06).** An earlier draft used one `disclosure` group with
`requested`/`returned` inside.

- **Every reference schema splits the record into a request side and a response side:** CloudTrail
  `requestParameters`/`responseElements`, GCP `request`/`response`, Kubernetes
  `requestObject`/`responseObject`, OCSF `api.request`/`api.response`. None uses a word like
  "disclosure", so the single group was split in two.
- **The request side became `request`.** It is valid for any operation, mutations included, as
  CloudTrail's `requestParameters` is.
- **The other side is `returned`, not `response`.** `response` suggests the whole HTTP response, body
  and status included. The group holds neither: it holds `count` and `items`, and the outcome is
  already `result`. `returned` names what the group actually holds, which is what the call handed back.
- **`returned` is not read-only.** For a read, `items` is what you received. For a mutation, it is
  what the call handed back. That's usually the object `resource`
  already names, but not always: `GenerateRelease` acts on a `Component` and returns a new
  `ComponentRelease`.

### 4.1 Shape

`request.parameters` and `returned` are **open objects**. The schema reserves only the attributes that
mean the same thing across operations and that an investigator would filter on across them. Everything
else an operation records is its own key, with no schema change.

```json
"request": {
  "parameters": { "<the operation's own request parameters, under the API's names>": "…" }
},
"returned": {
  "count": 2,
  "items": [
    {"type": "secret", "namespace": "acme", "name": "db-creds", "uid": "…"}
  ]
}
```

| Group | Valid on | Reserved | Open |
|---|---|---|---|
| `request.parameters` | Any event | nothing | The operation's own request parameters under their API names, allowlisted per operation, e.g. `podName`, `container`, `sinceSeconds` |
| `returned` | Any event, on success | `count`: entries returned (a secret, a log line, an audit record) | Operation-specific keys, if an operation ever needs one |
| `returned.items[]` | — | `type`, `namespace`, `name`, `uid`: identity of each object returned. `uid` tells apart two objects created under the same name. | Operation-specific keys per item |

**Why `request.parameters` and not parameters directly under `request`:**

- OpenObserve flattens nested keys with underscores. A parameter named `id` would turn `request.id`
  into `request_id`, which is an existing top-level field the module filters on
  (`community-modules/.../openobserve/audit_queries.go:95`). Under `request.parameters` the flattened
  name is always `request_parameters_*`.
- It leaves room for request-side fields that aren't parameters, such as which fields a mutation
  changed.

**Existing fields these sit next to:**

- `http` holds the request line, with the query string deliberately excluded. `request.parameters`
  holds query and body parameters, never path parameters, which are already in `http.path` and
  `resource`.
- `resource` is the object the operation addressed. `request` doesn't repeat it. `returned.items` does
  repeat it on a single-object read, on purpose (rule 4).
- `result` is the outcome. `returned` describes what came back, never how the call ended.
- `metadata` and `resource.metadata` stay as they are: open maps with no writer, unindexed by the
  modules.

**Mutations this phase:** no create, update or delete operation populates either group yet; only the
planned reads do. Both groups are defined generally, so a mutation can adopt them later without
another schema change. For a mutation, the reviewed list matters more than for a read: a
`CreateSecret` body carries the secret values, so its list would include things like the secret type
and target plane, never `data`.

**Rules:**

1. **Reserved attributes are declared in the schema and indexed.** Operation-specific keys are stored
   but not indexed, the same treatment `metadata` gets.
2. **An operation-specific key is added per operation in review.** `request.parameters` is an allowlist, never
   a dump of the query string or body, so a parameter can't slip in unreviewed. An operation-specific
   key may not reuse a reserved name.
3. **A key is promoted to reserved when a second operation needs the same meaning.** That's a minor
   version bump.
4. **`items` is used even for a single-object read**, with one entry. "Who read `db-creds`" is then
   one filter on `returned.items.name`, whether the secret came from a get or a list. This
   is the CloudTrail `resources[]` and OCSF `resources[]` shape.
5. **Each item carries a `type`.** CloudTrail, OCSF and Snowflake all type each entry, and a future
   read may return several kinds of object at once (the resource tree does).
6. **Free-text values are capped, with no truncation flag.** The cap is documented in the schema
   reference. CloudTrail does the same for `userAgent` ("content exceeding that limit is truncated").
7. **A cut `items` list shows as `count` greater than the number of items**, so it needs no flag of
   its own. Our list reads can't exceed 100 items per page (`api/handlers/pagination.go:13`).

**What each planned read records:**

| Read | `request.parameters` | `returned` |
|---|---|---|
| `GetSecret` | — | `count: 1`, `items[secret]` |
| `ListSecrets` | — | `count`, `items[secret…]` |
| Pod / workflow logs | `podName`, `container`, `task`, `sinceSeconds` | `count` |
| `QueryAuditLogs`, `QueryLogs`, `GetPlatformLogs` | Their filter, time-window and search parameters, under the API names | `count` |
| `QuerySpansForTrace` | `traceId`, time window, `includeAttributes` | `count` |
| `GetSpanDetailsForTrace` | `traceId`, `spanId` | `count: 1` |

`QuerySpansForTrace` is audited on every call, because a definition applies per operation and a
policy can't match on a parameter. The recorded `includeAttributes` shows whether attributes were
actually returned.

### 4.2 Left out, and why

| Left out | Why |
|---|---|
| `lines`, `bytes` | `lines` repeats `count`, since both log endpoints return one entry per line. `bytes` gives size, not identity. |
| A returned time window (`first_time`/`last_time`, `time_range`) | Only meaningful for log reads, so it doesn't belong in the shared schema |
| `keys` / `fields` (which parts of an object) | No planned read lets the caller choose a subset; every secret read returns all keys. Remote-connect grants are per key, so if it comes back it can add `keys` as its own item key. |
| `limit`, `cursor` | `count` covers the first. A cursor is an expiring continue token, so it can't replay a page. |
| `version` on items, and on `resource` | A Kubernetes `resourceVersion` can't be traced back to content, and status updates change it constantly (§3.1, §8). Where a version means something in OpenChoreo, it's already in the name: a `ComponentRelease` is immutable. An operation can still add a version as its own key if one ever means something. |
| An authorization block (GCP `authorizationInfo`) | Already covered. The hierarchy the decision was made on is folded into `resource`, the outcome is `result`, and the permission checked follows from `operation_id`. |

**Indexing.** Declare `returned.count` as a number, and `returned.items.type`, `.namespace`, `.name`
and `.uid` as keyword fields. Leave `request` and every operation-specific key stored but unindexed.
The resource hierarchy already carries the scope (namespace, component, environment), which covers
the common filters.

**What follows from B:**

- Golden cases in `record_shape_test.go`.
- Mapping and query changes in `community-modules`, for OpenSearch and OpenObserve. Both can filter on
  `returned.items.name` (see `openchoreo-audit-returned-items-filtering.md`). Azure Log Analytics
  is handled separately, with the module work in progress there.
- `request` and `returned` added to `AuditLogRecord` in the observer and adapter APIs, plus a
  `returned.items.name` filter, following AT-9.
- Display in `occ audit-logs` and the Backstage plugin.
- An entry in the schema reference and PII register (delivery plan §3.1).

## 5. Capture: where the details are recorded

**Store it in a sibling field of `AuditData`, not inside `Resource`.** `SetResource` replaces the whole
`*Resource` (`context.go:39`), so anything stored there would be wiped by the next call. The
`Hierarchy` field was kept as a sibling for exactly that reason. It takes two setters:

- `audit.SetRequest(ctx, operationID, params)`: called before the authz check, so a denial still
  records what was asked for.
- `audit.SetReturned(ctx, operationID, Returned)`: called after a successful inner call.

Each call names the operation it describes. The reason is explained below.

**Call them from the authz wrappers (`services/*/service_authz.go`, `internal/observer/service/*_authz.go`),
not from the handlers.** REST and MCP share these services, so one call site covers both surfaces. The
wrapper already sees the post-authz objects, which is exactly what was disclosed. `AuthzChecker.Check`
already writes `SetHierarchy` from this layer (`services/authz.go:52`), so this layer already writes
audit data.

Every operation captures in its own wrapper. Secrets are no exception, because without a version
(§3.1) the wrapper already has everything it records: each returned secret's name and uid.

**A record keeps only the `request` and `returned` that describe its own operation.** The setters
fire on any audited request that reaches a wrapped read. A mutation that resolves a secret or lists
items through a `*ServiceWithAuthz` on its way would otherwise carry that read's parameters and
results on its own record. This is the same last-write exposure `SetHierarchy` has.

Each setter names the operation it describes, and `buildEvent` keeps a group only when that name equals
the event's `operation_id`. A read's capture therefore can't attach to another operation's record.
Restricting by category would not work, because both groups are valid on mutations too.

**Fix this pitfall before auditing any list.** `FilteredList` calls `Check` once per item
(`services/pagination.go:288`), and each call overwrites the hierarchy. An audited list would record
the last item checked as its hierarchy, possibly a denied one. The list wrapper must call
`SetHierarchy` with the list scope after `FilteredList` returns. A test should pin this.

**Rejected: capturing the response in middleware.** Wrapping the `ResponseWriter` and parsing JSON
would:

- route secret values through audit code
- not see MCP results, which have a different shape
- break on streams and hijacked connections
- couple audit to response schemas

### 5.1 Effect on create, update and delete records

**Their shape does not change.** `request` and `returned` are `omitempty`, and no mutation populates
either group this phase. Records under `management` and `authorization` therefore carry neither,
even when a mutation's code path passes through a wrapped read (§5). The existing management cases in
`record_shape_test.go` should pass without edits, apart from the version line below. That is the
review contract for this change, the same way #4703's was a byte-identical `definitions.gen.go`.

**Their `schema_version` does change, from `"1.0"` to `"1.1"`.** `SchemaVersion` is a single constant
stamped on every published event (`types.go:103`), and all six golden cases pin `"1.0"`. A consumer
pinning `1.x` is unaffected. One that matches the exact string `"1.0"` would stop matching new
management records, even though their shape hasn't changed. That goes in the release note.
Versioning per category was considered and rejected. The version describes the record schema, and a
reader should not need to know an event's category before knowing how to parse it.

**Secret create and update do not count as disclosures.** `CreateSecret` and `UpdateSecret` return
the secret's data, but only what the caller sent in that request (`buildResponseSecret(...,
stringMapToBytes(req.Data))`, `services/secret/service.go:167,231`). Nothing comes back that the
caller didn't already have. `UpdateSecret` prunes keys absent from the request and does not echo
them. `DeleteSecret` returns no body.

**Some mutations return data the caller didn't send.** `GenerateRelease` returns a ComponentRelease
that embeds the workload. Creating or updating a release binding returns its overrides. Neither
carries secret values, since secrets appear only as references (§2.2), so neither is a disclosure
this plan needs to record. They don't populate `returned` this phase. `GenerateRelease` is the
natural first candidate later: it acts on a `Component` and returns a different object, so `returned`
would name something `resource` doesn't.

**A secret read is linked to the write that produced its values by time.** A read falls between two
audited secret updates. That is reliable for writes made through the API. It can't see a change
made outside the API, such as a `kubectl edit` on the plane Secret, and recording versions wouldn't
help either (§3.1, §8).

## 6. Mechanism and phasing

Promoting a read follows the precedent #4703 set: remove it from `RESTExemptions` and add an
`Overrides` entry under `CategoryAccess`, named like `read_audit_log`. Examples are `read_secret`,
`list_secrets` and `read_component_logs`. No new table. Each phase also binds the matching MCP tools.

**The definition and the `request`/`returned` capture land together, in one reviewed change.** A read
becomes auditable only through code. Policies then decide which defined reads are published. That
keeps the default policy to the one rule D-7 needs, and keeps each operation's `request.parameters`
allowlist a reviewed decision (§8, "defining every read").

**A read can't be audited without recording what was read.** A test fails if an operation under
`CategoryAccess` has no `request`/`returned` capture declared. The mechanism is a declared set of
operation-to-capture pairs, checked against the generated definitions, in the same style as the
coverage gates.

**MCP coverage needs a gate.** openchoreo-api's `MCPToolExemptions` covers only tools that declare a
mutating action. A promoted REST read whose MCP tool is left unbound fails nothing. **Recommendation:**
a test that every MCP tool calling a service method behind a `CategoryAccess` REST op is bound. In
practice that means a declared list of tool-to-op pairs, checked against `mcpEnrichment`. Observer's
table is already exhaustive (`TestMCPToolCoverage`), so its pinned counts just move.

**All of the following ship in this phase.** The steps below are the order of implementation work,
not a prioritization of what gets delivered.

| Step | Scope | Why this order |
|---|---|---|
| **0. Plumbing + pilot** | `Request` and `Returned` types (open `request.parameters`; `returned` with reserved `count` and `items`), the two setters, emitter keeps each group only for its own operation and drops `returned` on non-success, the capture test, schema `1.1`, golden cases. Pilot on `QueryAuditLogs`: record its filter, time-window and search parameters under their API names, plus `count`. Contract, observer API and module changes follow. | `QueryAuditLogs` is already audited and only missing these details, so it proves the whole path first: producer, index, query API and display. |
| **1. Secrets** | `GetSecret` and `ListSecrets` (REST only; no MCP tool returns values). | The first operations whose `returned.items` lists objects, so they exercise the item path and the FilteredList hierarchy fix. |
| **2. Live logs** | `GetReleaseBindingK8sResourceLogs` + `get_resource_logs`, and `GetWorkflowRunLogs` + `get_workflow_run_logs`. Parameters: `podName`, `container`, `task`, `sinceSeconds`. Returned: `count`. | — |
| **3. Observer logs and traces** | `QueryLogs` + `query_component_logs`/`query_workflow_logs`, `GetPlatformLogs` + `query_platform_logs`, `QuerySpansForTrace` + `query_trace_spans`, and `GetSpanDetailsForTrace` + `get_span_details`. Five tools leave observer's `MCPToolExemptions`, where they're exempt today as `reasonTelemetryRead`: `query_component_logs`, `query_workflow_logs`, `query_platform_logs`, `query_trace_spans` and `get_span_details`. `query_traces` stays exempt, and its reason is reworded to say summaries carry no attributes. | — |
| **4. Default policy** | Control-plane chart: a rule with `publish: false` for `GetReleaseBindingK8sResourceLogs` and `GetWorkflowRunLogs`. Observability-plane chart: replace the existing `QueryAuditLogs` `publish: false` rule with one for `QueryLogs`, `GetPlatformLogs`, `QuerySpansForTrace` and `GetSpanDetailsForTrace` (D-7). | Each service validates rules against its own operations, so the rule is split by chart. |

The §2.3 log-permission question doesn't block step 2. The audit records whatever the authz check
decides, and the question is raised separately.

Each step leaves `definitions.gen.go` diffs limited to the promoted ops. The pinned counts in both
`audit_coverage_test.go` files move by the number promoted.

**Testing for each promoted op:**

- A record-shape case for success, denial and failure.
- A **no-leak test**: the marshaled event must not contain any value from the response fixture, such
  as the base64 secret data or a log line.
- The FilteredList hierarchy test for lists.

## 7. Decisions

| # | Decision | Recommendation |
|---|---|---|
| **D-1** | Where the details live: `metadata` (A) or first-class groups (B) | **B, as `request` and `returned`** (§4). A cannot be searched, and moving off it later breaks saved queries. |
| **D-2** | The Medium tier: audit it, or reduce what it discloses | **Not in this phase.** For later: reduce what it discloses rather than audit it. Sanitize the resource tree (strip ConfigMap data, drop or sanitize `RenderedRelease`), and gate `Evaluates` for other subjects (§2.3). Notification channels are the one candidate for auditing, being low volume and credential-bearing. Workload, binding and release reads are not in this tier: they return secret references only (§2.2). |
| **D-3** | Log-follow volume: one event per poll, or coalescing | **Settled by D-7: one event per call.** Each `occ … logs --follow` poll returns new lines, so each is a real read. Log reads ship unpublished, and the volume is the operator's choice when they turn one on. Publishing can be switched off by operation, category, surface, result or `actor_types`. It can't be switched off for a named actor or entitlement: `validatePolicy` rejects that for a `publish: false` rule (`internal/server/middleware/audit/policy.go:103-107`), so nobody can configure themselves out of the trail. If volume ever needs more, the fallback is Doppler-style aggregation (first and last read per actor and object). |
| **D-4** | Whether free-text log search terms (`searchPhrase`) go into `request.parameters` | **Record them** as an operation-specific key under the API name, capped at 256 characters, with no truncation flag (§4.1 rule 6). "Searched prod logs for `password`" is exactly the signal wanted, and Splunk, BigQuery and Kubernetes all record the query. **Reconfirmed 2026-10-08 after review on #4396**, where the PII and retention risk was raised. The record has to say what was read, and some PII in an audit trail is expected: every record already carries the actor's identity and source IP. Reading the trail needs `auditlogs:view` (AX-4). The search text is documented as personal data in the audit schema reference. Audit retention (365 days by default) is longer than log retention by design. Practice elsewhere is split: Splunk and BigQuery record query text, while GCP's audit `request` "should never include user-generated data" and OWASP advises against logging sensitive PII directly. |
| **D-5** | Whether `Wirelogs` moves from `CategoryManagement` to `CategoryAccess` | **Move it**, in its own change with a release note. It is a read of live traffic. Changing a published category value breaks saved queries, so it should not ride along with this work. `Exec` stays under management, since it is interactive control. |
| **D-6** | Value fingerprints (HMAC of each secret value) to prove whether a leaked value matches what was read | **Not in this phase; tracked as a follow-up.** Raised on #4396: a keyed hash is the only way to identify the value a read returned, which the `resourceVersion` couldn't (§3.1). It's deferred for three reasons. Secret writes are already audited, so a read can be placed in time against them. It needs a key to manage, and rotating the key breaks comparison with older records. It needs a gated way to hash a candidate value, as Vault does with `/sys/audit-hash`. Add it if matching leaked values becomes a requirement. |
| **D-7** | Whether `access`-category events publish by default | **Decided 2026-10-06: split by operation**, once audit logging is enabled. Secret reads and `QueryAuditLogs` publish by default. The four log reads and the two trace reads ship unpublished: `GetReleaseBindingK8sResourceLogs`, `GetWorkflowRunLogs`, `QueryLogs`, `GetPlatformLogs`, `QuerySpansForTrace` and `GetSpanDetailsForTrace`. Each chart carries its own rule, because each service validates rules against its own operations (§6 step 4). The control-plane chart's `policies: []` gains a rule for the two openchoreo-api log reads. The observability-plane chart's existing `QueryAuditLogs` `publish: false` rule is replaced by one for `QueryLogs`, `GetPlatformLogs`, `QuerySpansForTrace` and `GetSpanDetailsForTrace`. **Two behaviour changes go in the release note.** Secret reads start producing events. Audit-log queries, unpublished until now, start producing events too. The note also says that an operator running in allowlist mode (`defaults.publish: false`) gets none of these until they add a `categories: [access]` rule. Practice elsewhere is split: AWS logs `GetSecretValue` by default, while GCP's `AccessSecretVersion` is a Data Access log, off by default. Ours matches AWS and the dedicated secret managers. |
| **D-8** | An older storage module silently ignores the new `returned.items.name` filter | **Open.** Adapters decode the query body with a plain `json.Decoder`, with no `DisallowUnknownFields`, no request validation and no `additionalProperties: false` in the contract. Sent to an older module, the filter is dropped, and the query returns every record. **Recommendation:** new adapters echo the filters they applied in the query response (for example, `appliedFilters`). The observer checks it, and returns an error asking for a module upgrade when the filter isn't there. That covers modules already deployed, because they never send the field. The observer can't filter results itself instead, since pagination and totals would be wrong. |

## 8. Not in scope

- Auditing every read. Most reads disclose platform metadata, and the scope doc's "volume is high,
  value low" still holds for them.
- Recording response bodies, even redacted ones.
- Reads made outside OpenChoreo's APIs (`kubectl get secret`). The Kubernetes audit log covers those.
- Fixing the §2.3 authz and sanitization problems. They are prerequisites to file, not part of this
  plan.
- **Considered after the market research, and not adopted:**
  - **Defining every read, with policy as the only control** (the GCP Data Access and Kubernetes
    model). Two reasons:
    - It complicates the default policy. It needs a category of its own for routine reads, plus a
      shipped `publish: false` rule covering roughly a hundred of them.
    - Every read would need its own reviewed `request.parameters` allowlist anyway. Without one, an operator
      switching on an unenriched read gets a record that doesn't say what was read, which is the
      requirement this plan exists for.
  - **Recording which entitlement granted the read** (Snowflake `policies_referenced`). The actor's
    entitlements are already on the record, so this is a nice-to-have.
  - **Separate verbs for revealing and fetching a value** (Bitwarden, Infisical). No current read
    lets the portal reveal a value apart from fetching it, and `user_agent` already separates the
    portal from `occ`.
  - **One event per secret on a bulk read** (AWS). Kept only as a fallback for a storage module that
    can't filter on an array field. Neither OpenSearch nor OpenObserve needs it.
  - **A read-only name for the returned side**, such as `retrieved`. `returned` covers mutations too.
  - **`resource.version` on every event** (decided 2026-10-08). It was meant to join a write to a later
    secret read, which no longer records a version (§3.1). Nothing else justifies it. It can't be
    traced back to content. It can't order events, since Kubernetes compares it for equality only. It
    can't detect changes made outside the API either: `resourceVersion` changes on every modification,
    status updates included, and OpenChoreo's controllers write status constantly, so it would almost
    never match. `metadata.generation` moves only on spec changes and could support drift detection,
    but that's a separate feature.
- **`POST /api/v1/remote-connect:resolve`, deferred.** It is off by default (`remoteConnect.enabled:
  false` in the control-plane chart). It sits outside the OpenAPI spec, so it is in neither the audit
  table nor the exemption table, and no coverage gate sees it.
  - **Who can call it:** any authenticated subject. It is behind the JWT middleware only
    (`cmd/openchoreo-api/main.go:273`), with no permission check on the call itself.
  - **What is checked:** each declared dependency, through `BatchCheck`
    (`handlers/remoteconnect.go:168-355`):
    - `component:connect` for each endpoint dependency
    - `resource:connect` for each resource dependency
    - `resource:read-secrets` for the values behind a connectable resource's secret-backed outputs,
      and only when `secretsEnabled` is on
  - **What is deliberately not checked:** the caller's own project and component. They come from a
    local workload file, and by design need not exist yet.
  - **Who holds the actions by default:** `developer` and `platform-engineer` hold all three
    (`values.yaml:1418-1429`, `1539-1551`), and `admin` has them through `"*"` (`values.yaml:1355`).
    `sre` holds none of them.
  - **What a caller with no grants gets:** a signed capability with no targets or grants. No agent is
    provisioned, because provisioning runs only when something was authorized (`remoteconnect.go:526`).
  - **What is recorded today:** each authorized secret read, as an Info-level application log line
    with the subject, resource, source and key (`remoteconnect.go:480-483`). That line goes to the
    service log, not to the audit trail.
  - **Coming back to it:** it gets the grant-issuance shape (grant coordinates, never values), and
    it also needs the out-of-spec coverage check.

## 9. Unverified

- ~~Array filtering in OpenSearch and OpenObserve~~ **Resolved 2026-10-06.** OpenSearch flattens an
  object array into a multi-valued keyword per sub-field (confirmed from its docs). OpenObserve stores
  the array as a JSON string, and an anchored `strpos` match is exact (tested locally). Details are in
  `openchoreo-audit-returned-items-filtering.md`. The final OpenObserve column names
  (`returned_items`) follow the flattening rule the tests showed, but weren't tested under that exact
  name.
- ~~Whether `QuerySpansForTrace` honours `includeAttributes`~~ **Resolved 2026-10-08:** it does. The
  handler adds `attributes` and `resourceAttributes` only when the flag is set
  (`internal/observer/api/handlers/traces.go:337`).
- **Whether any real ComponentType, Trait or ResourceType renders a `kind: Secret` with data.** If one
  does, the `RenderedRelease` in the resource tree carries it, and the tree moves to High. None was
  found in samples or the Helm charts.
- **Whether the Backstage portal polls the resource-tree endpoint.** It bears on D-2.
