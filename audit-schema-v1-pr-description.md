# feat(api): freeze the audit event schema as v1

## Purpose
> Briefly describe the problem or need driving this PR and how it resolves the issue. Include links to related issues if applicable.

Audit events carry no version stamp, so a stored record cannot say which schema produced it. That cannot be fixed retroactively. Everything written before a version field exists stays unidentifiable once the shape moves.

Two field names also described their contents wrongly. `resource.id` holds a Kubernetes object UID at every site that sets it, not a logical identifier, and an integrator who maps it as an id gets a value that changes whenever an object is recreated under the same name. `timestamp` is documented as "when the action occurred" but holds response-complete time, because the emitter runs in a deferred call after the handler returns. On a hijacked connection like `/exec/` that can be minutes late.

This is the last cheap moment to fix any of it. Nothing consumes the record yet, so renaming is a no-op today. Once log routing and indexing key on these names, the same change costs an index migration and breaks every saved query written against the old ones.

## Approach
> Summarize the solution and implementation details.

`schema_version` is `"1.0"`, major.minor, following CloudTrail's rule: major on a field removal or a changed value representation, minor on an addition. A consumer pinning `1.x` is guaranteed that later additions will not break it.

Three renames, all compiler-caught and with no behavioural change:

| Was | Now |
|---|---|
| `timestamp` | `event_time` |
| `service` | `producer` |
| `resource.id` | `resource.uid` |

`event_time` is more than a rename. It is now captured when the surface adapter receives the request rather than stamped at emit time, so it means what its name says. `buildEvent` reads no clock at all now, which leaves one clock read per event in one place.

Three additions:

- `actor.issuer` and `actor.session_id`, from the `iss` and `sid` claims. A bare `sub` is unique only within its issuer, so the same `sub` from two IdPs was previously indistinguishable. Both are resolved in `jwt/resolver.go` alongside `sub`, so the audit package stays ignorant of token formats.
- An `http` group carrying `method` and the decoded request path, on events that arrived over an HTTP boundary.

`resource.id` to `uid` touched 74 lines across the handlers. MCP needed one edit, because all 54 of its call sites, in 12 files, share `mcphandlers.setAuditResource`.

`Resource` and `Hierarchy` lost their json tags. Neither reaches the encoder: `resolvedResource` projects both into `resourceJSON`, which is the only definition of the published resource group. `Actor` and `HTTPInfo` keep theirs, since `eventJSON` embeds them by their own types.

## Behavioural changes
> Changes visible to callers or operators.

Published field names change as tabled above. `resource.uid` is absent at 7 of the 79 REST sites, all deletes and non-CRUD mutations where the service method returns only an error and there is no object to read a UID from. `resource.type` is unaffected.

The `http` group is present on any event produced at an HTTP boundary and absent for MCP `tools/call` operations, which have no request line of their own. Its presence is not derivable from `origin`: a token rejection at `/mcp` records `origin: mcp` with a real `POST /mcp` request line, while a tool call on the same surface records none. Two golden cases pin that pair.

`http.path` excludes the query string, so a parameter that changes an operation's meaning is not in the audit trail. The access log, joined by `request_id`, still has the full URI. The path is the decoded form so that it agrees with `r.PathValue`, which the resource group is seeded from.

`docs/audit/coverage-matrix.md` is regenerated. 90 rows now say `uid` where they said `id`.

## Deferred, with reasons recorded

Five fields from the original scope were cut after applying the additive-safe test consistently. Each is a minor bump whenever it lands, so none is blocked by this freeze:

- `audit_id` waits for the pre-action event. It would be 1:1 with `event_id` until an operation emits more than one record, so it ships with the two-event model that gives it something to group. Until then, an event is its own group. Worth recording for whoever implements it: `request_id` cannot substitute, because a client chooses the `X-Request-ID` header and can repeat it across unrelated requests, and on MCP one HTTP request already carries several `tools/call` operations.

- `received_time` waits for the collection pipeline. OCSF defines its counterpart `logged_time` as a collector's stamp, and there is no collector yet, so a producer-side version would approximate a field it does not own. Two timestamps microseconds apart also invite a wrong latency subtraction.
- `producer_instance` and `sequence` move together, since neither works alone. They detect head and interior gaps in a stream but never tail loss, which needs a heartbeat that does not exist. There is also no drop-prone hop to protect today: the only sink is an in-process writer to stdout.
- `actor.assurance` would be a constant. It is derivable from `actor.type`, because a subject context only exists where the JWT middleware already validated the signature.

The schema reference, PII register and IdP correlation guide are a follow-up PR. This one freezes the shape; that one publishes the contract, including the limitations noted above.

## Related Issues
> Include any related issues that are resolved by this PR.

Part of #3343. Builds on #4643 and #4664.

## Checklist
- [x] Tests added or updated (unit, integration, etc.)
- [ ] Samples updated (if applicable)
- [ ] Added `backport/<release-branch>` label if this should be backported (e.g., `backport/release-v1.0`)
- [x] This PR includes AI-generated code or content

## Remarks
> Add any additional context, known issues, or TODOs related to this PR.

Two commits on top of `main`, 62 files, +621/−328.

`go build ./...` is clean, `golangci-lint` reports 0 issues, and tests pass across `./internal/...`, `./pkg/...`, `./tools/...` and `./cmd/...`, all re-run after rebasing onto current `main`. Regenerating the audit definitions and coverage matrices shows no further drift. The controller and webhook suites fail locally for a missing envtest binary under `bin/k8s/`, unrelated to this change.

The rebase was checked rather than assumed: none of the six commits merged since this branch diverged touches a file this PR changes, and none added an `audit.SetResource` call site the rename would have missed. The count is 72 sites setting a UID at both revisions.

Eight golden cases in `record_shape_test.go` pin the published record byte for byte, including field order. Any change to the wire shape now shows up as a diff in review rather than silently.

One open item: `actor.session_id` is unverified against a real token. `sid` is optional in OIDC and nothing in the tree reads or configures it, so the field may be empty in every deployment until an IdP that issues it is confirmed. If it turns out Thunder does not, that becomes a documented omission class in the schema reference rather than a code change.

`producer` for the observer remains `observer`, matching its deployment and service names and the service keys in `tools/auditgen` and `tools/auditcoverage`. An earlier draft renamed it to `observer-api` to match the OpenAPI spec, which would have broken the rule that makes the value useful to an operator correlating events with a pod.
