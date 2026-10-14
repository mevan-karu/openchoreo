## Purpose
`resource.resource` is the Resource level of the audit record's hierarchy — a sibling of `component`, not another spelling of `resource.name`. On a ResourceRelease or ResourceReleaseBinding it holds the owning Resource, while `name` holds the release or binding itself. The observer spec previously claimed the two always duplicated each other and offered no filter for it, so there was no way to ask for every event touching a given Resource.

## Approach
Adds `resource.resource` as a first-class audit log filter across the stack:
- `openapi/observer-api.yaml` and `openapi/observability-logs-adapter-api.yaml`: new `resource` array on the resource filter, added to the filter-values enum, and the "no filter for it" rationale replaced with what the field actually means.
- Regenerated `internal/observer/api/gen` and `logsadapterclientgen`.
- Wired through `types`/`pkg/observability` request structs, the validation tables, the adapter request body, and the `query_audit_logs` MCP tool.

## Related Issues
<!-- fill in -->

## Checklist
- [x] Tests added or updated (unit, integration, etc.)
- [ ] Samples updated (if applicable)
- [ ] Added `backport/<release-branch>` label if this should be backported
- [x] This PR includes AI-generated code or content

## Remarks
The logs adapter contract gained a field but stays schema-compatible: an adapter built against the older spec ignores `resource.resource` and returns unfiltered results rather than erroring. The OpenSearch adapter module needs the matching update before this filter is usable end to end.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
