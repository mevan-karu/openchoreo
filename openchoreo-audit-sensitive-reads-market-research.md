# How Other Products Audit Reads: Comparison with the Sensitive-Reads Plan

**Companion to:** `openchoreo-audit-sensitive-reads-plan.md` · **Date:** 2026-10-05
**Method:** Official docs fetched on 2026-10-05, plus source code where the docs were silent. Claims
marked **unverified** could not be confirmed from a fetched page. Claims marked **inferred** follow from
the docs but have no example to back them.

> **Superseded in places (2026-10-06).** This document records the research as it was done. Several
> decisions moved on afterwards, and the plan is the source of truth:
>
> - **Names.** The `disclosure` group, with `requested`/`returned` inside, became two top-level groups:
>   `request.parameters` and `returned`. Both are valid on any event; this phase only reads populate
>   them (plan §4).
> - **Defaults.** Secret reads and `QueryAuditLogs` publish by default. The four log reads ship
>   unpublished (plan D-7).
> - **Versions.** No version is recorded, either on returned items or on `resource`. A Kubernetes
>   `resourceVersion` can't be traced back to content. A keyed hash of secret values is a follow-up
>   (plan §3.1, D-6, §8).
> - **Storage filtering.** It's in `openchoreo-audit-returned-items-filtering.md`. The one-event-per-secret
>   fallback isn't needed for OpenSearch or OpenObserve.

---

## A. The audit event's data and format, checked against other products

Part A is the main answer. It covers whether the fields the plan adds are the right ones, and whether
the shape will still hold when reads we haven't planned for yet are audited. Part B (§1–6) is the
broader survey of coverage and defaults.

### A.1 The envelope we already have

There's no change here. The base record already lines up with the reference schemas:

| Ours | Closest equivalent |
|---|---|
| `actor` (type, id, issuer, session_id, entitlements) | CloudTrail `userIdentity`, GCP `authenticationInfo` |
| `action`, `operation_id` | CloudTrail `eventName`, GCP `methodName` |
| `category` | CloudTrail `eventCategory`, plus `readOnly` |
| `result` | GCP `status`, CloudTrail `errorCode` |
| `resource` (type, namespace, …, uid, name) | GCP `resourceName`, CloudTrail `resources[]` |
| `request_id`, `source_ip`, `user_agent` | CloudTrail `requestID`, `sourceIPAddress`, `userAgent` |
| `schema_version` | CloudTrail `eventVersion` |

`schema_version` uses **the same evolution rule as CloudTrail's `eventVersion`**: "CloudTrail
increments the major version if a change is made to the event structure that is not
backward-compatible … the minor version if a change adds new fields". So adding `disclosure` as a
minor bump (1.1) is the established pattern.

### A.2 The `disclosure` fields, one at a time

The "Outcome" column records what was decided in review on 2026-10-06. The plan's §4 holds the
final shape.

| Field in the first draft | Who records the same thing | Outcome |
|---|---|---|
| `requested`: a per-operation map of allowlisted parameters | **Free-form per operation everywhere:** CloudTrail `requestParameters` ("The parameters, if any, that were sent with the request", 100 KB cap); GCP `request` ("may not include all request parameters"); Kubernetes `requestURI`; BigQuery query text; OCSF `query_info.query_string` | **Kept, fully open, nothing reserved.** Parameters are recorded under the API's own names. |
| `requested` cap/truncation flag | CloudTrail truncates `userAgent` at 1 KB with no flag; BigQuery has `queryTruncated` | **Cap kept, flag dropped.** Truncation describes the audit record, not the request. The cap is documented instead, as CloudTrail does. |
| `returned.count` | GCP `numResponseItems` ("number of items returned from a List or Query API method"); BigQuery `outputRowCount`; Infisical `numberOfSecrets`; Vault `elide_list_responses` | **Kept, reserved.** |
| `returned.items[]` (identity of each object returned) | CloudTrail `resources[]` ("A list of resources accessed in the event": ARN, account, type); OCSF `resources[]`; Snowflake `base_objects_accessed[]` (`objectDomain`, `objectName`, `objectId`, `columns[]`); GCP `authorizationInfo[]`, one per resource and permission | **Kept, reserved, and each item gains a `type`.** Every reference schema uses one array of uniformly shaped, typed entries. |
| `item.version` + `item.data_version` | 1Password `used_version`, Infisical `secretVersion`, AWS `VersionId`, Azure versioned object id | **Collapsed to a single `version`**, the version of the content disclosed. No product records a declaration version alongside it. |
| `item.keys` (secret key names) | Snowflake `columns[]`, BigQuery `TableDataRead.fields`, Vault (key names kept readable), Bitwarden (per-field events) | **Dropped.** The precedents are products where the caller *chooses* which part to reveal or select. No planned OpenChoreo read does that: every secret read returns all keys. |
| `returned.containers` (log reads) | — | **Dropped.** It was specific to one operation. |
| `returned.first_time` / `last_time` | No precedent. Splunk records the *requested* window (`search_et`/`search_lt`). | **Dropped.** Only meaningful for log reads. |
| `lines`, `bytes` | `bytes`: AWS S3 `bytesTransferredOut`, BigQuery, OCSF `query_info.bytes` | **Dropped.** `lines` repeats `count`. `bytes` gives size, not identity. |
| `items_truncated` | — | **Dropped.** A cut list shows as `count` greater than the number of items. |
| `requested` vs `returned` split | GCP `request`/`response`; BigQuery `JobConfig`/`JobStats`. CloudTrail sets `responseElements` to `null` for read-only APIs and relies on `resources[]`. | **Supported.** Ours says more about what was returned than CloudTrail does, which is the point of the requirement. |
| Never values, bodies or log lines | Every product by default. Content capture is always an opt-in with a warning. | **Strongly supported.** |

### A.3 What the reference schemas have that ours doesn't

- **A `type` on each item.** CloudTrail, OCSF and Snowflake all carry one. A single read can return
  several kinds: the resource tree returns Deployments, ConfigMaps and Pods in one response. Without
  `type` those items are ambiguous.
  **Adopted.**
- **The authorization detail behind the decision.** GCP `authorizationInfo[]` records `{resource,
  permission, granted}` per resource. **Not adopted, because the record already covers it:**
  - the hierarchy the decision was made on is folded into the `resource` group
    (`internal/server/middleware/audit/types.go:220-241`)
  - the outcome is `result`, with `denied` distinct from `failure`
  - the permission checked follows from `operation_id`, since each operation always checks the same
    action

  The one thing that can't be worked out is which of the actor's entitlements matched. The PDP knows
  it (`internal/authz/casbin/pdp.go:464`), and `actor.entitlements` already carries the full list, so
  it's a nice-to-have.

### A.4 The agreed shape

`requested` and `returned` are open objects. Only attributes that mean the same thing across
operations, and that an investigator would filter on across them, are reserved:

```json
"disclosure": {
  "requested": { "<the operation's own request parameters, under the API's names>": "…" },
  "returned": {
    "count": 2,
    "items": [{"type": "secret", "namespace": "acme", "name": "db-creds", "uid": "…", "version": "90133"}]
  }
}
```

- **`requested`:** nothing reserved. An allowlist per operation, under the API's names.
- **`returned`:** `count` and `items[]` reserved.
- **`items[]`:** `type`, `namespace`, `name`, `uid` and `version` reserved.
- **Extras:** stored but not indexed. They can't reuse a reserved name, and they're promoted to
  reserved when a second operation needs the same meaning. That's a minor bump.
- **Single-object reads still use `items`**, with one entry. That makes "who read `db-creds`" one
  filter on `items.name`. It is the CloudTrail and OCSF choice.

How likely future reads fit, with no schema change:

| Read | `requested` | `returned` |
|---|---|---|
| Resource tree (if D-2 audits it) | — | `count`, `items[{type: deployment\|configmap\|…}]` |
| A workload with literal env values (if D-2 audits it) | — | `items[{type: workload}]` |
| Export or download (none exists yet) | its own format and filter parameters | `count` |
| Remote-connect, if it returns | its own parameters | `items[]`, plus `keys` as its own item key |

**Mapping to OCSF**, for a future webhook or SIEM transport (scope doc TR-1):

- `items[]` maps to API Activity `resources[]`.
- A query parameter maps to `query_info.query_string`.
- `category: access` maps to `activity_id: Read`.
- `count` has no OCSF field, so it would travel as an unmapped extension.

### A.5 Making "the policy decides what's audited" true for reads

With policies, operators can already choose what gets audited. **But only among operations that have
a definition:**

- The policy vocabulary is built from defined operations alone (`internal/auditconfig/audit.go:107-120`).
- A rule naming an exempt read such as `GetWorkload` fails validation (`:228-231`).
- The middleware passes any route without a definition straight through (`middleware.go`, the
  `patternMap` lookup).

Today, then, `RESTExemptions` decides which reads *can* be audited, and policy only switches them
off.

**Giving every read a definition** would make policy the single control. This is how GCP Data Access
logs work (defined for every service, shipped disabled) and how Kubernetes works (every request passes
through the audit policy).

**Not adopted,** for two reasons:

- **It complicates the default policy.** It needs a category of its own for routine reads, and a
  shipped `publish: false` rule matching it. The chart ships `policies: []` today.
- **Every read would still need its own reviewed `requested` allowlist.** Without one, an operator
  switching on an unenriched read gets a record that doesn't say what was read, which is the
  requirement this work exists for. Capturing parameters generically isn't safe without review either:
  free-text parameters are exactly where sensitive input appears. GCP and Kubernetes can define
  everything because their read records don't promise to say what was read. Ours does.

**What the plan does instead:** a read becomes auditable through a code change that adds its
definition and its disclosure capture together, in one reviewed PR. Policies decide which defined
reads are published. A test fails if an `access`-category operation has no disclosure capture, so no
read can be audited without recording what was read.

---

# B. Coverage and defaults survey

This is the survey as first written, and it assesses the first draft's fields, including some that
were later dropped (key names, the returned time window). The decisions are in Part A and §5.

## 1. Verdict

The plan follows mainstream practice. **No product contradicts its core shape**: audit a chosen set of
sensitive reads, record what was read without the content, keep reads apart from changes, and never
copy the data. Every product that records content makes it opt-in and attaches a warning.

There are two genuine disagreements, and neither breaks the plan.

1. **Whether secret reads are logged by default.**
   - AWS logs `GetSecretValue` by default, as the plan does.
   - GCP files `AccessSecretVersion` under Data Access logs, which are **off by default**.
   - Practice is split, so D-7 is a policy choice, not a matter of following a standard.
2. **Value hashes (D-6).** HashiCorp Vault HMACs values. Every other product surveyed records no
   hash. Vault's reason doesn't apply to us (§3.2).

The research also turned up four ideas the plan doesn't have yet (§4). Two of them could change the
design:

- **One event per secret on a bulk read**, which is what AWS does. It would remove the unverified
  OpenObserve array question from the plan entirely.
- **Per-principal exemptions** for log-follow volume. GCP offers them, but our policy validator
  deliberately forbids them (§3.1).

## 2. What each product does

### 2.1 Infrastructure and cloud

| Product | Are reads audited, and by default? | What is recorded about the read | Content |
|---|---|---|---|
| **Kubernetes API server** | Per policy rule. With no matching rule the level is `None`. | The upstream example policy logs secrets, configmaps and `pods/log` at `Metadata`: "requesting user, timestamp, resource, verb, etc. but not request or response body". `requestURI` keeps the full query string, such as `?container=…&sinceSeconds=…`. | `RequestResponse` would capture the body. The example policy keeps secrets at `Metadata` for that reason. |
| **AWS CloudTrail** | **Management events, reads included, are on by default.** Data events (S3 `GetObject`) are opt-in and billed. `readOnly` marks reads. | `GetSecretValue` takes `SecretId`, `VersionId` and `VersionStage` as request parameters. The returned version appears on the paired KMS `Decrypt` event (`SecretVersionId` in the encryption context). **`BatchGetSecretValue` is logged as `GetSecretValue` entries.** S3 data events record `bytesTransferredOut`. | `SecretString`/`SecretBinary`: "the service does not include it in AWS CloudTrail log entries. If you create your own log entries, you must also avoid logging the information in this field." |
| **Google Cloud Audit Logs** | Admin Activity is always on. Data Access (`ADMIN_READ`, `DATA_READ`, `DATA_WRITE`) "are disabled by default for all services but some BigQuery services." Secret Manager `AccessSecretVersion` is `DATA_READ`, so it is **off by default**. | `numResponseItems` is the "number of items returned from a List or Query API method". `request` and `response` are separate fields. **Per-principal exemptions** are supported. | `request`/`response` may omit "privacy-sensitive" parameters. |
| **Azure Key Vault** | "Resource logs aren't collected by default." | `SecretGet` records the REST request URI, and the object `id` "when an object is returned". Key Vault ids include the version. **Inferred:** the `id` on `SecretGet` carries it. | No value column in `AZKVAuditLogs`. |

### 2.2 Secret managers

| Product | Are reads audited, and by default? | What is recorded about the read | Content |
|---|---|---|---|
| **HashiCorp Vault** | Every request and response, once an audit device is enabled. **Fails closed:** if no device can write, Vault refuses the request. | Full request and response. "Only *values* are hashed: keys of objects are not", so a KV read logs the **key names**. `elide_list_responses` swaps a list for its count. | **HMAC-SHA256 of string values, with a per-device salt kept in Vault's own encrypted storage.** `/sys/audit-hash` "can be used to discover whether a given plaintext string … appears in the audit log in obfuscated form." |
| **1Password Events API** | Item usage in shared vaults. | `item_uuid`, `vault_uuid`, **`used_version`**, and an action: `reveal`, `secure-copy`, `fill`, `export` and others. | None. |
| **Bitwarden** | Yes, field by field. | Viewed item (1107), Viewed password (1108), Viewed hidden field (1109), Copied password (1111), Exported organization vault (1602). | None. "Event logs rely on user-reported and client-level data, which technically could be modified or suppressed." |
| **Infisical** | Yes. | Single read `get-secret`: `secretId`, `secretKey`, **`secretVersion`**. Folder read `get-secrets`: path plus **number of secrets only**. UI reveal is a separate event. | None. |
| **Doppler** | Only when "a payload containing the secret is returned". | **Aggregated**: per actor and secret, "the first time it was read, and the most recent time it was read." | None. |
| **GitHub** | No event for a workflow reading a secret at runtime. API request events are opt-in and stream-only. | The job-preparation event "includes the list of secrets that were provided to the job" (`secrets_passed`): names, recorded when the secrets are injected. | None. |

### 2.3 Data, observability and developer platforms

| Product | Are reads audited, and by default? | What is recorded about the read |
|---|---|---|
| **Snowflake** `ACCESS_HISTORY` | Yes (Enterprise Edition). | The objects and **columns** each query accessed, both named and underlying. `policies_referenced` records which masking or row policy applied. Query text (truncated at 100K) and row counts are in `QUERY_HISTORY`. |
| **BigQuery** | Data Access, on by default for some BigQuery services. | Query text "truncated if exceeds 50K", with a `queryTruncated` flag. Referenced tables, `outputRowCount` and bytes are kept as a separate stats object, distinct from the request config. Table reads record the accessed `fields`. |
| **Splunk** | Searches go to `_audit`. | The search string, plus the earliest and latest time searched (`search_et`/`search_lt`). |
| **Datadog Audit Trail** | Has a log-query event ("A user ran a Log Management List query…"). | Whether it carries the query string is **unverified**. |
| **Elasticsearch / OpenSearch** | Audit is off by default. OpenSearch has a separate `COMPLIANCE_DOC_READ` category. | Request bodies (the search query) are off by default in Elasticsearch, with the warning "sensitive data may be audited in plain text". OpenSearch's watched-field read auditing "can dramatically increase the size". |
| **Grafana Enterprise** | "The auditing feature is disabled by default." Data-source queries are audited as `action: query`. | Request and response bodies are off by default: they "may expose sensitive data such as query parameters, credentials, or personally identifiable information." |
| **Backstage Auditor** | Severity low/medium/high/critical. **Low maps to debug**, so catalog reads aren't logged by default. | — |
| **GitLab** | `variable_viewed_api`, `secrets_manager_read_project_secret`, `job_artifact_downloaded`: streamed only, not saved to the database. No event found for viewing a job log. | — |
| **OCSF 1.9** | Reads are an `activity_id` (Read, Query) inside the same classes as writes. | API Activity has a `resources[]` array. Datastore Activity has `query_info.query_string` (required) and `bytes`. **No result-count or time-range field.** |

## 3. The plan's choices against the evidence

| Plan choice | Supported by | Against it | Assessment |
|---|---|---|---|
| **Audit an allowlist of sensitive reads, not all reads** (§2, §6) | GCP (Data Access opt-in), AWS (data events opt-in), the Kubernetes example policy, Backstage (low severity off), Grafana/ES/OpenSearch (off by default), GitLab | Vault logs everything, but its whole API is sensitive. Warehouses (Snowflake, BigQuery) record every query. | **Supported.** Tiering by sensitivity is the norm for platforms like ours. |
| **Secret reads publish by default** (D-7) | AWS `GetSecretValue` (management event, on by default), Vault, Infisical, Doppler | GCP `AccessSecretVersion` (Data Access, off by default), Azure (diagnostics off by default) | **Split.** Both stances are mainstream. Ours matches AWS and the dedicated secret managers. |
| **Record identity and version, not content** (§3) | 1Password `used_version`, Infisical `secretVersion`, AWS (version on the KMS event; `VersionId` parameter), Azure versioned `id` (inferred) | Kubernetes records no version on a `get` (inferred from source) | **Supported.** Secret products record the version. General platforms mostly don't. |
| **Record key names, never values** (§3) | Vault keeps key names readable; GitHub `secrets_passed` lists names; Bitwarden logs per field | Infisical's bulk read records only a count | **Supported**, and more detailed than Infisical's bulk event. |
| **Record exact query parameters, including search terms** (§3, D-4) | Kubernetes full `requestURI`, BigQuery query text, Splunk `search`, OCSF `query_string` (required) | ES and Grafana keep request bodies off by default, with PII warnings | **Supported**, given the allowlist and length cap. BigQuery adds a `queryTruncated` flag, which the plan lacks (§4.5). |
| **Record a result count** (§4) | GCP `numResponseItems`, BigQuery `outputRowCount`, Infisical `numberOfSecrets`, Vault `elide_list_responses` | OCSF has no field for it | **Supported.** |
| **Record the first/last timestamp returned** (§4) | No precedent. Splunk records the *requested* window, not the returned one. | Nothing against it. | **Novel but cheap.** It is what lets an investigator replay a log read, so keep it. |
| **No `bytes`** (dropped last round) | Most products don't record it. | AWS S3 `bytesTransferredOut`, BigQuery bytes, OCSF `query_info.bytes` | **Mildly against.** Bytes shows up where data transfer is the concern. The decision still holds, because count plus time window identify the read. The evidence leans the other way, though, and adding it later is cheap. |
| **`requested` always, `returned` only on success** (§3) | GCP request/response split, BigQuery config vs stats, Splunk lifecycle `info`, Azure `id` "when an object is returned" | Doppler is stricter: it doesn't log a read that returned nothing | **Supported.** Logging denied reads too follows Kubernetes and Vault, and is the more useful choice. |
| **A first-class indexed group, not `metadata`** (D-1) | BigQuery typed `TableDataRead`/`JobStats`, Infisical typed metadata, 1Password fixed schema, Azure resource tables, OCSF `query_info` | AWS `requestParameters` and GCP `metadata` are free-form | **Supported.** Products with typed schemas are easier to query, and our reason (indexing) stands. |
| **No value hashes** (D-6) | AWS, GCP, Azure, Kubernetes, 1Password, Bitwarden, Infisical, Doppler, GitHub: none hash values | **Vault** | **Holds.** See §3.2. |
| **A separate `access` category** | GCP (Data Access vs Admin Activity; GKE sends get/list there), Kibana `event.type: access`, OpenSearch `COMPLIANCE_DOC_READ`, 1Password and Doppler (separate access logs) | AWS (`readOnly` is a flag) and OCSF (Read is an activity id) | **Supported.** Splitting by category is the common choice. A flag is the alternative, and `category` already does that job here. |
| **Never copy the data** | Every product by default. Content capture is always opt-in with a warning: K8s `RequestResponse`, ES `emit_request_body`, Grafana bodies, Vault `log_raw`. | — | **Strongly supported.** |
| **Capture server-side, in the authz wrapper** (§5) | Every server-side product | Bitwarden and 1Password rely on client-reported events, and Bitwarden documents that these can be suppressed | **Supported.** Client-reported events are the weaker design. |

### 3.1 Volume (D-3)

Every product with read auditing warns about volume, and four mechanisms recur:

- **Per-principal exemptions.** GCP `exemptedMembers`; AWS selectors on `userIdentity.arn`.
- **Category-level opt-out.** AWS "Exclude AWS KMS events", because KMS crypto calls are over 99% of
  KMS events.
- **Aggregation.** Doppler keeps only the first and last read per actor and secret.
- **Stream-only tier.** GitLab streams high-volume read events without persisting them.

OpenChoreo **deliberately does not allow per-principal exemption.** `validatePolicy` rejects
`actors` or `entitlements` on a `publish: false` rule: "actors/entitlements may not be used to
suppress publishing … permitted for escalation" (`internal/server/middleware/audit/policy.go:103-107`).
Suppression is possible by operation, category, surface, result or **`actor_types`** (for example,
all service accounts), but never for a named person or group. That is a stricter stance than GCP's
`exemptedMembers`, and a defensible one: it stops anyone from configuring themselves out of the trail.
D-3 therefore rests on operation-level or actor-type suppression, with Doppler-style aggregation as
the fallback. Relaxing the rule for reads is not recommended.

### 3.2 Why Vault's HMAC doesn't overturn D-6

Vault hashes values because it **logs full request and response bodies**. The HMAC is what makes that
safe. The plan never logs bodies, so it doesn't have the problem the HMAC solves.

Vault also keeps the brute-force risk contained in ways we have no equivalent for:

- the salt lives inside Vault's own encrypted storage
- comparison happens only through `/sys/audit-hash`, which Vault's access policy gates
- salts are per device and regenerated when a device is disabled

Without that infrastructure, a value hash in our records would be a liability, not a feature. D-6
stands.

## 4. Ideas the plan doesn't have yet

1. **One event per secret on a bulk read.** AWS logs `BatchGetSecretValue` as `GetSecretValue`
   entries rather than one batch record (CloudTrail entries page: "Generated by the GetSecretValue and
   BatchGetSecretValue operations"). For `ListSecrets` that means emitting one `read_secret` record per
   secret returned instead of an `items[]` array:
   - "Who read `db-creds`" becomes a plain filter on `resource.name`.
   - No array-of-objects mapping is needed in OpenSearch or OpenObserve, which removes the open
     verification item in plan §9.
   - The cost is up to 100 events per page in place of one, and a correlation field, since the
     shared `request_id` already ties them together.

   **Not adopted** (see §5): A.4's uniform `items[]` covers the same query in one event. This stays
   the fallback.
2. **Who granted the read.** Snowflake records `policies_referenced` and BigQuery `policyTags`. Our
   PDP already names the matching entitlement value on an allow (`internal/authz/casbin/pdp.go:464`).
   Putting that on an access record answers "why was this person allowed to read it" without a
   separate lookup against role bindings that may have changed since. It is a candidate minor
   addition, and probably not part of the first phase.
3. **Showing a value versus fetching it.** Bitwarden separates viewed from copied. Infisical logs a
   UI reveal apart from an API read. `user_agent` already tells the portal apart from `occ`. A
   distinct verb would only matter if the portal ever lists secrets without values and reveals them
   on demand. It doesn't today, since `ListSecrets` returns values, so **no change** is needed.
4. **Secrets injected into a build or deployment.** GitHub logs which secrets were provided to a job.
   The OpenChoreo equivalent is a workflow run or release binding that mounts secret references. Those
   create and update events are already in the management trail, and the references are in the
   object, so **no change** for now. Note it as a known boundary in the schema reference.
5. **A truncation flag for `requested`.** BigQuery pairs its 50K cut-off with `queryTruncated`. The
   plan caps `searchPhrase` at 256 characters (D-4). **Not adopted** (see §5): the cap is documented
   instead.
6. **Failing closed** (Vault). The scope doc's blocking delivery mode (RE-14) already covers this. It
   doesn't belong in this plan.

## 5. Amendments to the plan

Applied to the plan on 2026-10-06, together with the format changes in Part A.

| # | Change | Outcome |
|---|---|---|
| A-1 | `ListSecrets` emits one `read_secret` record per secret returned | **Not adopted.** One event per call with `items[]` (A.4). Kept as the fallback in plan §4 if a storage module can't filter on an array field. |
| A-2 | Name operation-level and `actor_types` suppression as the first answer to volume. Per-principal suppression is deliberately rejected by `validatePolicy`. Doppler-style aggregation is the fallback. | **Applied** to D-3. |
| A-3 | Add `requested_truncated` | **Not adopted.** Truncation describes the record, not the request. The cap is documented instead, as CloudTrail does (plan §4.1 rule 6). §4 item 5 above is superseded. |
| A-4 | Cite the AWS (on by default) vs GCP (off by default) split as the context for D-7 | **Applied** to D-7. |
| A-5 | Record the granting entitlement | **Not adopted.** Covered by the resource hierarchy, `result` and `actor.entitlements` (A.3). Listed in plan §8. |
| — | Define every read, with policy as the only control | **Not adopted** (A.5). Listed in plan §8, with a disclosure-capture test added to plan §6. |

## 6. Unverified or inferred

- **Exact CloudTrail `requestParameters` for `GetSecretValue`.** Only the API's parameters were
  confirmed; there is no documented example event.
- **Whether GCP `AccessSecretVersion` records the version** in `resourceName`.
- **Whether Azure's `SecretGet` `id` includes the version** (inferred from the id format).
- **That Kubernetes `objectRef.resourceVersion` is empty on a `get`** (inferred from apiserver source,
  not tested).
- **That OpenSearch can filter on `disclosure.returned.items.name`** when `items` is mapped as a
  plain object array. The usual behaviour is to flatten such an array into multi-valued fields, which
  supports a term filter but not matching two fields inside the same item. I couldn't fetch the
  OpenSearch page that documents this, so check it before Phase 0. OpenObserve remains unverified
  too.
- **Whether Datadog's log-query event carries the query string.**
- **Splunk `_audit` `result_count`.** Only community answers describe it.

## Sources

- Kubernetes auditing: https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/ · apiserver `request.go`: https://raw.githubusercontent.com/kubernetes/kubernetes/master/staging/src/k8s.io/apiserver/pkg/audit/request.go
- AWS:
  - CloudTrail management events: https://docs.aws.amazon.com/awscloudtrail/latest/userguide/logging-management-events-with-cloudtrail.html
  - CloudTrail data events: https://docs.aws.amazon.com/awscloudtrail/latest/userguide/logging-data-events-with-cloudtrail.html
  - `GetSecretValue` API: https://docs.aws.amazon.com/secretsmanager/latest/apireference/API_GetSecretValue.html
  - Secrets Manager CloudTrail entries: https://docs.aws.amazon.com/secretsmanager/latest/userguide/cloudtrail_log_entries.html
  - Secrets Manager encryption: https://docs.aws.amazon.com/secretsmanager/latest/userguide/security-encryption.html
  - KMS Decrypt event: https://docs.aws.amazon.com/kms/latest/developerguide/ct-decrypt.html
- GCP:
  - Audit logs overview: https://docs.cloud.google.com/logging/docs/audit
  - Configuring Data Access logs: https://docs.cloud.google.com/logging/docs/audit/configure-data-access
  - Secret Manager audit logging: https://docs.cloud.google.com/secret-manager/docs/audit-logging
  - `AuditLog` type: https://docs.cloud.google.com/logging/docs/reference/audit/auditlog/rest/Shared.Types/AuditLog
  - GKE audit policy: https://docs.cloud.google.com/kubernetes-engine/docs/concepts/audit-policy
- Azure: Key Vault logging: https://learn.microsoft.com/en-us/azure/key-vault/general/logging · resource logs: https://learn.microsoft.com/en-us/azure/azure-monitor/platform/resource-logs
- Vault:
  - Audit-hash API: https://developer.hashicorp.com/vault/api-docs/system/audit-hash
  - Audit API: https://developer.hashicorp.com/vault/api-docs/system/audit
  - Value hashing source: https://raw.githubusercontent.com/hashicorp/vault/main/audit/hashstructure.go
- 1Password: item usage reference: https://www.1password.dev/events-api/reference/items-usage.md · usage actions: https://www.1password.dev/events-api/item-usage-actions.md
- Bitwarden: event logs: https://bitwarden.com/help/event-logs/
- Infisical: audit logs: https://infisical.com/docs/documentation/platform/audit-logs
- Doppler: access logs: https://docs.doppler.com/docs/access-logs
- GitHub: workflow run audit events changelog: https://github.blog/changelog/2021-02-23-github-actions-workflow-run-events-are-now-included-in-the-audit-log/ · enterprise audit events: https://docs.github.com/en/enterprise-cloud@latest/admin/monitoring-activity-in-your-enterprise/reviewing-audit-logs-for-your-enterprise/audit-log-events-for-your-enterprise
- Snowflake: `ACCESS_HISTORY` view: https://docs.snowflake.com/en/sql-reference/account-usage/access_history · `QUERY_HISTORY` view: https://docs.snowflake.com/en/sql-reference/account-usage/query_history
- BigQuery: audit metadata: https://docs.cloud.google.com/bigquery/docs/reference/auditlogs/rest/Shared.Types/BigQueryAuditMetadata
- Splunk: CIM audit logs data model: https://help.splunk.com/en/data-management/common-information-model/6.2/data-models/splunk-audit-logs
- Datadog: Audit Trail events: https://docs.datadoghq.com/account_management/audit_trail/events/
- Elastic:
  - Auditing settings: https://www.elastic.co/docs/reference/elasticsearch/configuration-reference/auding-settings
  - Kibana audit events: https://www.elastic.co/docs/reference/kibana/kibana-audit-events
- OpenSearch: audit logs: https://docs.opensearch.org/latest/security/audit-logs/index/ · Amazon OpenSearch Service audit logs: https://docs.aws.amazon.com/opensearch-service/latest/developerguide/audit-logs.html
- Grafana: auditing: https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/audit-grafana/
- Backstage: Auditor service: https://backstage.io/docs/backend-system/core-services/auditor/
- GitLab: audit event types: https://docs.gitlab.com/user/compliance/audit_event_types/
- OCSF 1.9: Datastore Activity: https://schema.ocsf.io/api/classes/datastore_activity · API Activity: https://schema.ocsf.io/api/classes/api_activity · `query_info`: https://schema.ocsf.io/api/objects/query_info
