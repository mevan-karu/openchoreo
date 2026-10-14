## Purpose
`resource.resource` is the Resource level of the audit hierarchy — a sibling of `component`, and on a ResourceRelease or ResourceReleaseBinding it differs from `resource.name`. The adapter contract gained it as a filter (openchoreo/openchoreo), so the module has to serve it. The index template already maps the field and the read path already returns it; only the filter path was missing.

## Approach
- Regenerated `internal/api/gen` from the updated adapter spec.
- Added `ResourceResources` to `AuditLogsQueryParams` and a `resource.resource` terms filter to the audit query builder.
- Wired the filter into `auditFilterFields` (filter-values), the request body mapping, and `clearAuditFilter`.
- Extended the "maps every filter onto its field" query test.

## Related Issues
<!-- fill in -->

## Checklist
- [x] Tests added or updated (unit, integration, etc.)
- [ ] Samples updated (if applicable)

## Remarks
Regeneration also picked up adapter-spec drift the module already owed to main (#4597): `EventsQueryResponse.Total` is now a required `int` and `searchScope` is optional. This PR carries only the minimal fixes to compile against that — it does not implement the `reasons` filter or unscoped sweeps. Note `QueryEvents` still dereferences `searchScope` without a nil check, so an unscoped request would panic instead of returning 501; worth a separate issue.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
