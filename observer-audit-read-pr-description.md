# feat(api): add the audit logs read path to observer

## Purpose

The audit trail is published but cannot be read back. This adds the query surface on the observer, so the portal and `occ` have something to build against before any module implements the storage side.

- `POST /api/v1alpha1/audit-logs/query` — audit records, filtered and paged, with an optional result-stacked timeline
- `POST /api/v1alpha1/audit-logs/filter-values` — the distinct values one filter can take, for populating a filter picker

Both answer `501` until a module serves them. The routes exist and are permission-gated, so that is a working state rather than a stub.

## Approach

- **Filters mirror the record.** `actor.id` filters `actor.id`, so a client needs no translation table. Record-derived filters keep the record's snake_case; query controls stay camelCase.
- **The record is served verbatim**, so a response can be diffed against an exported log line.
- **Never an empty success.** No trail support is `501`; an expired cursor is a `400` telling the caller to restart. An empty `200` would be indistinguishable from "nothing happened".
- **Filter values are a separate operation, not facets on the query.** Answering every filter at once costs one aggregation per filter on every query change.
- **Gated on `auditlogs:view`, cluster-scoped.** Both reads carry the same check — the filter-values read enumerates actors and resource names, so gating only the records would leave a way around it. Tenancy filters narrow results and never widen authorization.
- **Reading the trail is itself audited**, as `read_audit_log` under a new `access` category. The filter-values read is exempt on volume: a picker fires on every query change and would bury the record reads.
- **Both responses are validated against the spec** in `TestResponsesConformToSpec`, since the handlers return hand-written structs through an untyped passthrough. The audit read, the adapter request body, and the namespace-scoped denial are each asserted directly.

## Related Issues

Closes #4698

## Checklist

- [x] Tests added or updated (unit, integration, etc.)
- [ ] Samples updated (if applicable)
- [ ] Added `backport/<release-branch>` label if this should be backported (e.g., `backport/release-v1.0`)
- [x] This PR includes AI-generated code or content

## Remarks

- Module query handlers (OpenSearch, OpenObserve) are out of scope — both endpoints answer `501` until those land.
- Queries are bounded to a 366-day window rather than the 30 days the log endpoints allow, since the trail keeps its own retention.
- A module answering `501` with a JSON content type but no body degrades to `500`. The contract requires a body, and platform logs behaves the same way.
- `security-admin` from AX-4 does not exist as a bootstrap role, so the grant goes to `admin` and `platform-engineer` only. `sre` and `rca-agent` are deliberately excluded.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
