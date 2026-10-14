# feat(api): record resource.environment in audit events

## Purpose
> Briefly describe the problem or need driving this PR and how it resolves the issue. Include links to related issues if applicable.

Audit records showed the namespace, project and component an operation touched, but not the environment. You could see that someone updated a release binding, but not which environment it targeted — and on a refusal, not which environment they were refused for.

Environment is one of the four tenancy fields audit is meant to record, and the only one nothing populated. `authz.ResourceHierarchy` has no environment level, and the audit hook copies that struct, so there was nowhere for the value to come from.

This goes in before the event schema gets versioned, so v1 doesn't ship with a tenancy field missing.

## Approach
> Summarize the solution and implementation details.

`AuthzChecker.Check` already records the resource hierarchy for audit. It now also reads the environment from `req.Context.Resource.Environment`, which the check already carries. That covers REST and MCP at once, and since it runs before the PDP evaluates, denials carry the environment too.

The value is stored as-is, in the `{namespace}/{name}` form authorization uses, so the record matches what the policy matched. `Hierarchy` gains an `Environment` field, and it publishes as `resource.environment`.

Observer's `CheckAuthorization` is wired the same way. It records nothing yet, because all its environment-scoped operations are reads, but it won't need changing when an audited one becomes environment-scoped.

Adding a field to the event meant editing two render paths, so they were merged first: `Logger.LogEvent` now renders `Event.MarshalJSON`'s output instead of building slog attrs by hand.

## Behavioural changes
> Changes visible to callers or operators.

- `resource.environment` appears on the release-binding families, `/exec/` and wirelogs, as `default/production` rather than `production`.
- Wirelogs used to record the environment in `resource.name`. It now uses `resource.environment`, and `resource.name` is absent for that route.
- `json.Marshal` of an `Event` changes field order and no longer emits `"resource":null`. Nothing consumes that yet — `Logger` is the only sink.
- Multi-key `actor.entitlements` and `metadata` maps now have a stable key order.

The log line is otherwise byte-identical, including sub-second timestamps. The golden test was captured against the old implementation before the render changed, so that's verified rather than assumed.

## Related Issues
> Include any related issues that are resolved by this PR.

Part of #3343. Builds on #4571 and #4643.

## Checklist
- [x] Tests added or updated (unit, integration, etc.)
- [ ] Samples updated (if applicable)
- [ ] Added `backport/<release-branch>` label if this should be backported (e.g., `backport/release-v1.0`)
- [x] This PR includes AI-generated code or content

## Remarks
> Add any additional context, known issues, or TODOs related to this PR.

`go build ./...` is clean, `golangci-lint` reports 0 issues, `make code.gen` shows no drift, and tests pass across `./internal/server/...`, `./internal/openchoreo-api/...`, `./internal/observer/...` and `./pkg/mcp/...`.

Coverage follows the authorization check, which leaves three gaps on purpose. Observability alert notification channels have a required `spec.environment` but don't pass it to authz, so they record none — giving them one changes what policies can express, so it belongs in its own change. The Environment CRUD operations don't set the field, since `resource.name` is already the environment name there. And `/exec/` needed no change, because it already passes its resolved environment to the check.

Wirelogs sets the environment in the handler as well as in its check request, because its ownership-mismatch 403 returns before the check runs. Both read it from the same check request.
