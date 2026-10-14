# feat(observer): audit state-modifying operations, enforce it in CI

## Purpose
> Briefly describe the problem or need driving this PR and how it resolves the issue. Include links to related issues if applicable.

Observer had state-modifying operations that produced no audit record. #4571 built the audit framework and wired it into openchoreo-api alone, so nothing had ever tested whether it was reusable. This PR extends it to observer and closes #4624.

Only one operation actually needed auditing: `UpdateIncident`. The other 12 non-GET operations are exempted with a written reason. Eight are reads that happen to be POSTs, and four run on the unauthenticated internal port where there is no actor to record. The work here was the plumbing and the reuse, not the operations.

## Approach
> Summarize the solution and implementation details.

Both of observer's specs feed the generator, so a new mutating alert-rule write cannot pass CI unaudited. Each port builds its own audit middleware from its own spec, and both share one emitter, so a single configured policy covers the public port, the internal port and `/mcp`. Audit sits inside auth, which makes the actor the token's subject rather than `anonymous`. A second middleware sits outside auth to catch the 401s that auth short-circuits before anything downstream runs.

Two pieces moved out of openchoreo-api so observer could use them: `tools/internal/auditgen` for the spec-walking logic, and `internal/auditconfig` for config decode and validation. Both tools now take `-service` and pick from a registry, so adding a third API server is a registry entry rather than a new binary. openchoreo-api's generated definitions and its committed coverage matrix come out byte for byte identical, which is the proof these were refactors and not rewrites.

Observer's audit config loads from the same supplementary YAML as `auth.subject_types`, because `audit.policies` is a nested list that the flat env-var mapping cannot represent. `observer.audit` is now plumbed through the observability-plane chart.

## Behavioural changes
> Changes visible to callers or operators. The first one is the one to review carefully.

1. A scoped grant of `incidents:update` now actually permits the update. `UpdateIncident` authorized against an empty resource hierarchy, which becomes the `*` wildcard path, so only a cluster-wide binding ever matched. It now authorizes against the incident's own namespace, project and component, read from the store because `IncidentPutRequest` carries no scope of its own.
2. Because that read runs first, an unknown incident ID returns 404 before the authorization check. A caller without `incidents:update` can now tell an existing incident from a missing one. That is unavoidable if you want to authorize on the resource's own scope, and openchoreo-api already accepts the same tradeoff.
3. Observer's access log moves from `Debug("HTTP request")` to `Info("ACCESS-LOG")`, so it goes from logging nothing at the default level to one line per request. In exchange it now generates and validates `X-Request-ID`, which is what ties access logs to audit records.
4. `observer.audit` gains `enabled`, `defaults` and `policies` in the observability-plane chart. The defaults keep current behaviour: enabled, publish everything.

## Related Issues
> Include any related issues that are resolved by this PR.

Closes #4624. Builds on #4571 for the framework and #4623 for observer's generated OpenAPI server, both already in main.

## Checklist
- [x] Tests added or updated (unit, integration, etc.)
- [ ] Samples updated (if applicable)
- [ ] Added `backport/<release-branch>` label if this should be backported (e.g., `backport/release-v1.0`)
- [x] This PR includes AI-generated code or content

## Remarks
> Add any additional context, known issues, or TODOs related to this PR.

54 files, +3,467/-1,044. `go build ./...`, `go vet ./...` and `make test` are clean, `golangci-lint` reports 0 issues, and `make code.gen` regenerates with no drift.

`QuerySpanDetailsForTrace` is no longer exempted. #4640 reverted that operation from POST back to GET, so it is not state-modifying any more, and the coverage gate rejects an exemption that names it.

The four internal-port mutations stay exempted because port 8081 has no authentication. The audit middleware is wired there anyway, so lifting an exemption once that port gets auth needs no wiring change.
