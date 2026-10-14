## Purpose

Observer serves the audit trail over REST but not over MCP, so an agent asked "who deleted this component, and was anyone refused?" has no way to look. This adds a `query_audit_logs` tool with the same filter surface as `POST /api/v1alpha1/audit-logs/query`.

## Approach

The tool takes one argument per record field the REST query filters on, flattened from the nested groups onto their full path. `actor.id` becomes `actor_id`, `resource.namespace` becomes `resource_namespace`. Arguments map straight onto `types.AuditLogsQueryRequest` and run through `handlers.ValidateAuditLogsQueryRequest`, the same validator the REST handler uses, so a query that would earn a 400 fails the same way here instead of reaching the adapter. `include_timeline` and `timeline_interval` come along with it.

No tool for filter values. That endpoint populates a picker and fires on every keystroke that changes a query, which is why the REST contract keeps it separate, and an agent is not driving a picker.

The rest of the change is audit. Reading the trail is itself recorded in the trail, and REST's `QueryAuditLogs` already emits `read_audit_log` under `access`. Observer's MCP chain audited only 401s, so as written the tool would have been a way to read the trail that the trail does not show. `NewHTTPServer` now installs `mcpaudit`'s middleware from a binding table in `internal/observer/audit`, the same shape openchoreo-api uses.

One shared change fell out of that. `mcpaudit` now prefers a handler's `audit.SetResult` over its own classification, matching the precedence the REST middleware already applies. Observer enforces authz in the service decorators, inside the tool handler, and the MCP SDK folds a handler error into `CallToolResult.IsError` before the middleware can read its identity. Every denied read was landing as `failure`, indistinguishable from a backend outage. Since `result=denied` is the query an audit review actually runs, that was worth fixing rather than documenting.

## Related Issues

Closes #4730.

## Checklist

- [x] Tests added or updated (unit, integration, etc.)
- [ ] Samples updated (if applicable)
- [ ] Added `backport/<release-branch>` label if this should be backported (e.g., `backport/release-v1.0`)
- [x] This PR includes AI-generated code or content

## Remarks

`pkg/mcp/mcpaudit` is shared with openchoreo-api, so the change there deserves a look even though this PR is titled for observer. It is additive: a handler that never calls `SetResult` gets exactly the classification it got before, and openchoreo-api's only `SetResult` callers are the REST hijack paths in `exec.go`, which no MCP tool reaches.

`limit` declares its 1000 cap in the tool schema, unlike the other observer tools. A compliance sweep is the query most likely to reach for a big page, and the cap is an error rather than a clamp, so it should be visible before the call. The SDK rejects an oversized `limit` before the handler runs, and that rejected call is still audited.
