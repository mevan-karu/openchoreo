# Filtering Audit Records by a Returned Item's Name

**Companion to:** `openchoreo-audit-sensitive-reads-plan.md` (§4, §9) · **Date:** 2026-10-06
**Repos:** `openchoreo` (adapter contract), `community-modules` (the audit storage modules)
**Scope:** OpenSearch and OpenObserve are implemented with this plan. Azure Log Analytics is covered
here for reference only; it is handled with that module's own work in progress.

The event groups are `request.parameters` and `returned` (plan §4). Earlier drafts named them
`disclosure` (with `requested`/`returned` inside) and then `request`/`response`. The two OpenObserve
test runs in §4 used those earlier names. The flattening rule they demonstrate carries over to the
final names unchanged, as §4 explains.

---

## 1. The question

The plan puts the objects a call returned in an array of objects:

```json
"returned": {"count": 2, "items": [
  {"type": "secret", "namespace": "acme", "name": "db-creds", "uid": "u1"},
  {"type": "secret", "namespace": "acme", "name": "api-key",  "uid": "u2"}
]}
```

An investigator needs to ask "who read `db-creds`", which means filtering on `returned.items.name`.
The plan left open whether each audit storage module can do that. A string match is acceptable.

## 2. Answer

**All three backends can do it, so the one-event-per-object fallback isn't needed.** Each needs a
different technique.

| Backend | How `items` is stored | Exact filter on `items.name` | Evidence |
|---|---|---|---|
| **OpenSearch** | Mapped as a plain `object`, the array flattens into one multi-valued keyword per sub-field | `terms` on `returned.items.name`, the same as the entitlements filter | OpenSearch docs |
| **OpenObserve** | As a **JSON string** in an auto-created column, `returned_items` | `strpos(returned_items, '"name":"db-creds"') > 0` | **Tested** on the local cluster, OpenObserve v0.92.2 |
| **Azure Log Analytics** | Only in a column declared on the table (closed schema) | A `dynamic` column queried with `mv-apply`, or a `string` column matched with `contains_cs` on `"name":"db-creds"` | KQL and DCR docs; module code |

**String matching works, but only when anchored on the key.** A bare substring `db-creds` also matches
`db-creds-old`. That was reproduced on OpenObserve, and the same issue applies to Azure's
`contains` and `has`. The needle has to be `"name":"db-creds"`, quotes included. That's safe for two
reasons:

- **Names can't contain characters that need escaping.** Kubernetes object names are restricted to
  lowercase alphanumerics, `-` and `.`. The adapter must still validate the value against that pattern
  before building the needle, since it goes into a query string.
- **The key-value pair is a single unit,** so the needle doesn't depend on the order of keys inside an
  item.

**One change is needed in every module regardless: the record doesn't round-trip yet.** Each adapter
decodes into a typed `AuditRecord` and maps it field by field into `gen.AuditLogRecord`, which has no
`request` or `returned` field. New groups are stored but dropped from every API response. The
contract change goes in `openapi/observability-logs-adapter-api.yaml` (`AuditLogRecord`, line 1473).

## 3. OpenSearch

**How an array of objects is indexed.** The OpenSearch docs describe exactly this case. An array of
objects mapped as a regular object is "broken down into individual fields, and values for each field
across all objects are stored together":

```json
"patients.name": ["John Doe", "Mary Major"],
"patients.age": [56, 85]
```

The cost is that "the relation between age and smoking was lost". A query can't require two fields to
match **within the same item**. For "who read `db-creds`", that doesn't matter: a single-field `terms`
filter on `items.name` matches exactly, because the field is a keyword, not a substring.

`nested` would keep items separate, but it needs `nested` queries and aggregations. That would break
the module's generic `addTermsFilter` and its value picker. **Not recommended.**

**What the module does today:**

- The audit template is `"dynamic": "false"` (`init/setup-opensearch.sh:246`), so an undeclared field
  "is stored but unindexed, so it silently stops being filterable" (comment at `:229`). `request` and
  `returned` are therefore kept in `_source` already, but can't be searched until they're declared.
- **Entitlements are the precedent.** Claim values are copied into a keyword field
  (`copy_to: "actor.entitlement_values"`, `:255`). The filter is
  `addTermsFilter(filters, AuditEntitlementValuesField, params.ActorEntitlements)`
  (`internal/opensearch/audit_queries.go:119`). It's a `terms` query in `bool.filter`.

**Changes:**

1. Declare `returned.items.{type,namespace,name,uid}` as `keyword` and `returned.count` as a
   number in the template. Leave `items` as a plain object. Leave `request` undeclared, so it is
   stored but not indexed.
2. Add the filter with the existing `addTermsFilter`. The terms-aggregation value picker works on the
   same field unchanged.
3. Add `Request` and `Returned` to `AuditRecord` and to `toGenAuditLogRecord`.
4. **Rollout:** the template applies only to newly created daily indices. Existing `audit-logs-*`
   indices need `PUT <index>/_mapping`, then `_update_by_query` to index documents already written. If
   older records may stay unfilterable, letting them age out is simpler.

## 4. OpenObserve

**Tested twice on the local k3d cluster** (OpenObserve v0.92.2, default flatten level), each time in a
scratch stream that was deleted afterwards.

**Run 1** (stream `zz_disclosure_probe`, items at `disclosure.returned.items`). This established how
an array is stored and why the match has to be anchored. Records: `e1` lists `db-creds` and
`api-key`; `e2` reads `db-creds-old`; `e3` is a log read with no items.

| Query | Matched | Note |
|---|---|---|
| `SELECT *` | — | The array is stored as a JSON string. Scalars are flattened. **`podName` became `…_podname`: column names are lowercased.** |
| `str_match(<items column>, 'db-creds')` | e1, **e2** | False positive: `db-creds-old` |
| `<items column> LIKE '%"name":"db-creds"%'` | e1 | Exact |
| `str_match(<items column>, '"name":"db-creds"')` | e1 | Exact |
| `match_all('db-creds')` | error | "Search SQL not valid": no full-text index on this stream |

**Run 2** (stream `zz_reqresp_probe`, items at `response.items`, parameters at `request.parameters`).
Records:

- `e1` (`request_id: rid-1`): `request.parameters` = `{limit: 20, id: "param-id-nested"}`, and the
  items list `db-creds` and `api-key`.
- `e2` (`rid-2`): the items list `db-creds-old`.
- `e3` (`rid-3`): a **flat** `request.id: "param-id-flat"`, to test the collision `request.parameters`
  avoids.

| Query | Result |
|---|---|
| `SELECT *` | The items array is a JSON-string column named after its path, `response_items`. `response_count`, `request_parameters_limit` and `request_parameters_id` are flattened scalars. `e1`'s `request_id` is intact (`rid-1`). **`e3`'s `request_id` was stored as `param-id-flat`**: the flattened `request.id` overwrote the real correlation id. |
| `strpos(response_items, '"name":"db-creds"') > 0` | `e1` only. Exact, with no `db-creds-old` false positive. |
| `request_id = 'rid-3'` | No rows: `e3`'s real `request_id` is gone. |

**How this carries over to the final names.** OpenObserve names a flattened column by joining the path
with underscores, which both runs showed. With the final names, the items land in `returned_items` and
the count in `returned_count`. `request.parameters.<key>` is unchanged: `request_parameters_<key>`. The
final names were not re-tested separately.

Run 2 also shows that nesting the parameters under `request.parameters` is required, not cosmetic. A
flat `request` with a parameter named `id` silently corrupts `request_id` in OpenObserve.

The stored string kept the input's key order and compact form exactly:
`[{"type":"secret","namespace":"acme","name":"db-creds",…}]`.

**What the module does today:**

- The stream has no declared schema (`init/setup-openobserve.sh:194`), so columns are created as
  fields arrive. `request` and `returned` are therefore stored already, with no schema change.
- Records are rebuilt from the raw `log` column, so the nested `request` and `returned` come back
  intact once `AuditRecord` has the fields.
- **Entitlements are the precedent for arrays of strings:**
  `array_has(cast_to_arr("actor_entitlements_<claim>"), '<value>')`, OR'd across every
  `actor_entitlements_*` column (`internal/openobserve/audit_queries.go:113-126`). `array_has` matches
  whole elements, so **it doesn't work for an array of objects**. Hence the string match.
- Free-text search already uses `strpos(log, …) > 0` rather than `LIKE`, because OpenObserve rejects
  `LIKE` escape characters other than backslash (`audit_queries.go:132-135`). The new filter follows
  the same convention.

**Changes:**

1. Add the filter as `strpos(returned_items, '"name":"<validated name>"') > 0`. Keep the existing
   guard that skips the query when the column doesn't exist yet (`audit_queries.go:103-105`).
2. Add `Request` and `Returned` to `AuditRecord` and to `toGenAuditLogRecord`.
3. **Value picker: not supported in v1.** Listing distinct item names would need JSON extraction
   inside `unnest`. Whether this OpenObserve version supports that is unverified, and a free-text name
   input is enough.

## 5. Azure Log Analytics (reference only)

**The constraint is in the ingestion transform.** DCR transformations run once per record, and "Only
operators that take a single row as input and return no more than one row are supported". There is
no `mv-expand`, `mv-apply` or `summarize`. The only array functions are `array_concat`,
`array_length`, `pack`, `pack_array`, `parse_json` and `zip`. So **an array can't be flattened at
ingest** into a column of plain values. The module says the same:

> transformations run per record and have no mv-expand or bag_unpack
> (`internal/loganalytics/audit_columns.go:48`)

**This is the problem entitlements already hit, and they've been solved two ways:**

- **Committed (`HEAD`):** a delimited **string** column, `ActorEntitlementValues`, matched with
  `contains_cs` against a needle wrapped in the same delimiter. The README gives the reason a
  `dynamic` column was rejected: "it cannot be filtered across arbitrary claim keys, and Azure's
  row-level access conditions compare string columns only".
- **Working tree (uncommitted):** a `dynamic` column, `ActorEntitlements`, filtered at query time:
  ```kusto
  | where isnotnull(ActorEntitlements)
  | mv-apply _entKey = bag_keys(ActorEntitlements) to typeof(string) on (
      summarize _entMatch = max(iff(set_has_element(ActorEntitlements[_entKey], "admins"), 1, 0)))
  | where _entMatch == 1
  ```
  (`internal/loganalytics/audit_queries.go:324-344`). It ends in `summarize` so a record isn't
  repeated once per element.

**The table's schema is closed.** It is defined by `scripts/setup-audit.sh` (`--columns`, around
`:163-182`) and by the transform's `project` list (`scripts/audit-transform.kql:56-63`). A field
without a column is dropped. It survives only inside `RawRecord = tostring(LogMessage)`
(`audit-transform.kql:54`). `RawRecord` isn't a usable target for this filter: it holds the whole
record, so `"name":"db-creds"` would also match `resource.name` or a metadata key called `name`.

**Options:**

| | Column | Filter | Exact? | Row-level access (ABAC) can use it? |
|---|---|---|---|---|
| **A** | `Returned=dynamic`, from `LogMessage.returned` | `mv-apply _i = Returned.items on (summarize _m = max(iff(tostring(_i.name) == "db-creds", 1, 0))) \| where _m == 1` | Yes | No |
| **B** | `ReturnedItems=string`, from `tostring(LogMessage.returned.items)` | `ReturnedItems contains_cs '"name":"db-creds"'` | Yes, if anchored | Yes |
| **C** | No new column | `parse_json(RawRecord).returned.items`, then A's `mv-apply` | Yes | No |

**Recommendation: A.**

- It follows the working-tree entitlements pattern, so the module keeps one way of filtering arrays.
- It doesn't depend on how Kusto re-serializes JSON. Kusto doesn't keep property-bag key order, and
  the compact form of `tostring()` output is unverified, though the anchored needle doesn't depend on
  key order.
- Its cost is bounded, because `mv-apply` runs only on rows the time range and other filters leave.

ABAC on item names isn't a requirement. Row-level conditions scope tenancy columns such as namespace,
not which object was read. That removes the reason `HEAD` chose a string column.

**B is the string-filter option** if you prefer one technique across OpenObserve and Azure. C needs
no schema change but parses every row, so use it only as a stopgap.

**What option A needs:**

1. `Request=dynamic` and `Returned=dynamic` in `setup-audit.sh --columns`, each holding the whole
   group.
2. The assignments and `project` entries in `audit-transform.kql`.
3. The column constants in `audit_columns.go`, `auditProjection` and the row mapper.
   `unmarshalDynamic` accepts only JSON objects (`audit_mapping.go:114-126`), and whole groups are
   objects, so it works as is.
4. **Existing deployments:** add the columns to the live table and re-apply the DCR transform.
   `setup-audit.sh` uses `table create`; whether it handles an existing table is unverified.

**Your working tree is in progress.** `audit_columns.go`, `audit_queries.go`, the README and
`helm/values.yaml` have uncommitted changes in `community-modules`, and `scripts/` is untracked. This
research only read them. The `dcr.json`, `dcr.backup.json` and `dcr.norm.json` files at the module
root are an exported live DCR with real subscription and workspace IDs. Keep them out of any commit.

## 6. Changes common to the implemented modules

1. **Contract** (`openchoreo/openapi/observability-logs-adapter-api.yaml`):
   - add `request` and `returned` to `AuditLogRecord` (line 1473)
   - add a `returned.items.name` filter to `AuditLogsQueryRequest`
   - the same two changes in `observer-api.yaml`
   - then regenerate in each module (`make openapi-codegen`)
2. **Validate the name** against the Kubernetes name pattern in the adapter before building any
   query. The OpenObserve and Azure needles are built from it.
3. **Value picker for item names: not in v1** (OpenObserve is unverified). Filtering by a typed name is
   enough for the investigation case.
4. **An older module silently ignores the new filter.** Adapters decode the request body with a plain
   `json.Decoder`: no `DisallowUnknownFields`, no request validation, and no `additionalProperties:
   false` in the contract. Sent to an older module, a `returned.items.name` filter is dropped and the
   query returns every record. The plan tracks a guard for this as an open decision (plan D-8).

## 7. Unverified

- **The exact form of Azure's `tostring()` on a dynamic value** (whitespace). It only matters for
  option B.
- **Whether `setup-audit.sh` can add a column to an existing table**, or whether a separate
  `table update` step is needed.
- **OpenSearch was not tested live,** as no local OpenSearch is running. The behaviour comes from its
  docs and matches the existing entitlements field, which uses the same mechanism.
- **The final column names `returned_items` and `returned_count` in OpenObserve** follow from the
  flattening rule both runs showed, but weren't tested under those exact names.

## Sources

- OpenSearch: nested field type (object-array flattening): https://raw.githubusercontent.com/opensearch-project/documentation-website/main/_mappings/supported-field-types/nested.md · flat object: https://raw.githubusercontent.com/opensearch-project/documentation-website/main/_mappings/supported-field-types/flat-object.md
- OpenObserve: stream details (flatten level): https://openobserve.ai/docs/user-guide/data-processing/streams/stream-details/ · local tests, scratch streams `zz_disclosure_probe` and `zz_reqresp_probe`, 2026-10-06
- Azure:
  - Supported KQL in DCR transformations: https://learn.microsoft.com/en-us/azure/azure-monitor/data-collection/data-collection-transformations-kql
  - The dynamic data type (indexing, operators): https://learn.microsoft.com/en-us/kusto/query/scalar-data-types/dynamic?view=azure-monitor
  - String operators (term definition, three-character index): https://learn.microsoft.com/en-us/kusto/query/datatypes-string-operators?view=azure-monitor
  - `has` operator: https://learn.microsoft.com/en-us/kusto/query/has-operator?view=azure-monitor
- Module code: `community-modules` at `c15c9f0`, plus the Azure module's uncommitted working tree as
  noted in §5.
