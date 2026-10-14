# Auditing Sensitive Reads, and Recording What Was Read

The audit framework today records state changes and leaves reads out, since most reads are high volume and low value. But that is not true for sensitive data reads. For example, `GetSecret` and `ListSecrets` return secret values, and pod logs, build logs, log queries and span attributes return whatever your workloads wrote. Today none of these reads are audited.

Today only querying the audit trail is the one read we allow to audit and that is also disabled by default. Also its record says who looked and when, but not what they filtered for or how much came back.

So the plan is to audit the reads that disclose sensitive data, and record what each one disclosed without recording the data itself.

## What other systems do

I looked at how [AWS CloudTrail](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-event-reference-record-contents.html), [GCP Cloud Audit Logs](https://docs.cloud.google.com/logging/docs/reference/audit/auditlog/rest/Shared.Types/AuditLog) and [Kubernetes auditing](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/) record reads.

| | AWS CloudTrail | GCP Cloud Audit Logs | Kubernetes |
|---|---|---|---|
| Reads audited | Management reads on by default (incl. `GetSecretValue`), data reads opt-in | [Data Access log](https://docs.cloud.google.com/logging/docs/audit/configure-data-access), off by default | Per policy rule |
| What was asked for | `requestParameters` | `request` | `requestURI` |
| Which objects came back | `resources` | `resourceName` | `objectRef` |
| How many | — | `numResponseItems` | — |
| Records the content read | No, the [secret value is excluded](https://docs.aws.amazon.com/secretsmanager/latest/apireference/API_GetSecretValue.html) | No | No, at the `Metadata` level used for secrets |

None of them records the content that was read. They record what was asked for and which objects came back. Only GCP also records how many items came back. That matters for us, because log and audit queries return entries rather than objects. None of them records which version of a secret you received, and after a leak that is the first thing you need to know.

## Proposed Solution

The audit event gets two new groups, `request` and `returned`, the same split between what was asked for and what came back similar what above three systems are doing. Audited reads go under the existing `access` category:

```json
"request": {
  "parameters": { "podName": "checkout-7f9c", "container": "api", "sinceSeconds": 3600 }
},
"returned": {
  "count": 2,
  "items": [{ "type": "secret", "namespace": "acme", "name": "db-creds", "uid": "…", "version": "90133" }]
}
```

`request.parameters` records what was asked for, using the API's own parameter names. Unlike Kubernetes, it doesn't copy the raw request. Each operation records only the parameters reviewed for it, since free-text parameters are where sensitive input turns up. It is recorded on denied reads too, so failed attempts show up.

`returned` records what came back, and it only appears when the call succeeds. `items` lists each object returned, with the version you received. `count` is the number of entries returned. A single-object read also uses `items`, so you find out who read `db-creds` with one filter, whether it came from a get or a list.

Neither group is specific to reads. A create or update can use them later to record its own parameters and what it handed back, without another schema change. In this phase only the reads below populate them.

Values, log lines and response bodies are never recorded.

### Operations to be covered

| Operation | Surfaces | `request.parameters` | `returned` | Audited by Default |
|---|---|---|---|---|
| `GetSecret`, `ListSecrets` | REST | — | `count`, secrets with versions | yes |
| `QueryAuditLogs` | REST, MCP | filters, time window | `count` | yes |
| `GetReleaseBindingK8sResourceLogs` | REST, MCP | pod, container, `sinceSeconds` | `count` | no |
| `GetWorkflowRunLogs` | REST, MCP | task, `sinceSeconds` | `count` | no |
| `QueryLogs` | REST, MCP | scope, time window, filters, search | `count` | no |
| `GetPlatformLogs` | REST, MCP | cluster, namespace, pod, container, time window, search | `count` | no |
| `QuerySpansForTrace` | REST, MCP | trace, time window, `includeAttributes` | `count` | no |
| `GetSpanDetailsForTrace` | REST, MCP | trace, span | `count` | no |

Secret reads list each secret returned, because a secret changes over time and you need to know which one you read and at which version. Log reads record only a count, because the requested parameters already identify which logs were read.

Trace reads are audited only where they return span attributes, since attributes carry whatever the instrumentation records, such as URLs with query strings or database statements. `GetSpanDetailsForTrace` always returns them. `QuerySpansForTrace` returns them only when `includeAttributes` is set, so the record includes that flag. `QueryTraces` returns trace summaries without attributes, and stays unaudited.

The Default column applies once audit logging is enabled. The defaults follow CloudTrail and GCP, which record some reads by default and leave high-volume data reads opt-in. Secret reads and audit-log queries are published by default. Both are behaviour changes. Every secret read will now produce an audit event, and audit-log queries, which the observability plane ships unpublished today, will start producing them too. Log and trace reads are switched off in the default audit policy. You can turn each one on, and the extra audit volume is part of what it costs to collect that telemetry.

### Impact on existing users

`request` and `response` are being added to the audit event schema, so `schema_version` moves from `1.0` to `1.1`. Consumers that accept any `1.x` version are unaffected. Every event carries the new version, including create, update and delete events whose shape doesn't change. A consumer that checks for exactly `"1.0"` will need to be updated.

The parameters sit under `request.parameters` rather than directly under `request` so they can't collide with the existing `request_id` field in stores that flatten nested keys, such as OpenObserve.

Publishing needs no module change. The records go out as JSON lines, and OpenSearch and OpenObserve both keep fields they haven't seen before. Returning `request` and `returned` in query results, and filtering on them, needs the upgraded module. Until then, an older module returns records marked `1.1` with those groups left out, because `schema_version` describes the event as it was produced, not what the query returned. Azure Log Analytics only keeps the new groups once its table gains the columns, which is part of that module's work.

Appreciate any feedback on the direction.

