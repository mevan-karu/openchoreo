## Purpose
The OpenObserve logs module can't store or serve OpenChoreo's audit trail: the adapter answers neither audit-logs endpoint, and audit records mix into the container logs under the same retention. This PR adds audit log collection, storage and retrieval, bringing the module in line with the OpenSearch module.

## Approach
- **Collection:** Fluent Bit sends audit records from allowlisted producers (`auditLogs.producers`, matched on the kubelet log filename) to a dedicated `audit_logs` stream. `event_time` becomes the record's OpenObserve timestamp, so time windows and sorting are exact to the microsecond.
- **Storage:** the setup job creates the stream before any records arrive, with retention from `openObserveSetup.auditLogsRetentionDays` (default 365).
- **Read path:** implements `POST /audit-logs/query` (exact total, timeline with zero-filled buckets) and `POST /audit-logs/filter-values`. Records are rebuilt from the original log line so nested `metadata` stays intact, and entitlement filters cover every claim column.
- **Multi-cluster:** the gateway HTTPRoute now also exposes the audit stream's ingest path.

## Related Issues
- Resolves openchoreo/openchoreo#4716

## Checklist
- [x] Tests added or updated (unit, integration, etc.)
- [ ] Samples updated (if applicable)

## Remarks
- The default logs output now matches `kube.*` instead of `*`. The chart's only input tags `kube.*`, so container logs are unaffected.
- `ZO_INGEST_ALLOWED_UPTO=8760` is set on both bundled OpenObserve charts, so audit records Fluent Bit replays are not dropped by OpenObserve's 5-hour ingest window. It isn't gated on `auditLogs.enabled`, because that flag is set on the control-plane cluster in multi-cluster installs.
- Regenerating the API code made `searchScope` optional, so an events query without a scope now returns 501 instead of crashing the handler.
- Verified against OpenObserve v0.92.2, and with the rendered Fluent Bit 4.2.3 config running in Docker.
