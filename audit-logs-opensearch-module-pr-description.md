# feat: collect, store and serve audit logs in the opensearch module

<!-- Repo: openchoreo/community-modules -->

## Purpose

OpenChoreo's audit trail is written to the container logs of `openchoreo-api` and `observer`, where it expires with operational logs and cannot be queried as its own signal. This routes those records to a dedicated index with its own retention and implements the adapter contract the Observer queries them through.

Relates to openchoreo/openchoreo#4699.

## Approach

**Collection** — Fluent Bit moves audit records onto an `audit.*` tag and ships them to `audit-logs-*`, out of `container-logs-*`. Each producer is matched on the container log filename, which the kubelet writes, so a pod cannot route itself into the audit trail.

**Storage** — an index template for the v1 audit record and an ISM policy defaulting to 365 days. Both apply on every install so the destination is shaped before the first record arrives.

**Retrieval** — implements `POST /api/v1alpha1/audit-logs/query` and `/audit-logs/filter-values`, with a result timeline and per-filter value aggregation. `POST /api/v1alpha1/platform-logs/filter-values` answers `501`, which the contract sanctions for an adapter that serves platform logs but does not aggregate them.

Configuration:

```yaml
auditLogs:
  enabled: false           # routing; the template and policy apply regardless
  indexPrefix: "audit-logs-"
  producers:               # (namespace, container) allowlist
    - {producer: openchoreo-api, namespace: openchoreo-control-plane, container: api-server}
    - {producer: observer,       namespace: openchoreo-observability-plane, container: observer}

openSearchSetup:
  dataRetentionTime:
    auditLogs: "365d"
```

Three decisions worth noting:

- **Wildcard index pattern, not `GenerateIndices`' day-walk.** At 365 days that walk is ~8KB of request line against OpenSearch's 4KB default, which fails as a malformed request rather than a length error.
- **`track_total_hits: true`.** The contract specifies `total` as exact and offers no field to mark a count truncated, so counting cannot stop at the 10000 default.
- **`actor.entitlements` is copied into `actor.entitlement_values`.** The claim key varies by subject kind, so filter and value picker both read the copy.

## Related Issues

- openchoreo/openchoreo#4699

## Checklist

- [x] Tests added or updated (unit, integration, etc.)
- [ ] Samples updated (if applicable)

## Remarks

**Depends on** the audit contract in `openapi/observability-logs-adapter-api.yaml` reaching `openchoreo` `main`; the module's `SPEC` still points there. The committed stubs are byte-identical to what that spec currently generates.

**Two pre-existing bugs fixed in passing:**

- The `rewrite_tag` rule's regex could not contain double quotes — the tokenizer split `"msg":"AUDIT-LOG"` so the new tag became `:"AUDIT-LOG"`, which matched nothing and, with `Keep false`, silently discarded every audit record. Now `AUDIT-LOG(?!-)`.
- `Parsers_File parsers.conf` resolves against the collector's `--workdir`, not the config directory, so the ConfigMap's parsers were never loaded. The audit parser is in its own file loaded by absolute path.

**Unrelated churn:** regeneration renamed `gen.Success` to `gen.AlertWebhookResponseStatusSuccess` (the audit `result` enum's `success` collided with the alert webhook's), touching four pre-existing call sites.

**Known limitation:** the contract has no pagination — `limit` caps at 1000 with no cursor or offset, so a window holding more than 1000 records cannot be read past the first 1000. `total` reports the real figure, so a caller can tell and narrow the window.

**Verified on k3d against OpenSearch 3.3.0:** records land in `audit-logs-YYYY-MM-DD` with collector provenance and cluster stamp; a pod forging `producer: openchoreo-api` stays in `container-logs-*`; `AUDIT-LOG-RENDER-FAILED` stays in `container-logs-*`; real records do not leak into `container-logs-*`; `total` reported 988 against a returned page of 2; timeline keeps zero-count buckets; ISM attached at 365d.

Retrieval **through the Observer** is untested — blocked by two environment problems in the test cluster that affect existing endpoints equally (`/api/v1/logs/query` fails identically): `AUTHZ_SERVICE_URL` is unresolvable in-cluster, and the Observer calls `/api/v1/authz/evaluates` anonymously. Confirmed on that path: routes exist, JWT auth succeeds, the `auditlogs:view` decorator fires, and the read is itself audited.
