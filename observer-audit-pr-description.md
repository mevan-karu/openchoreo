# feat(observer): audit state-modifying operations, enforce it in CI

## Purpose
> Briefly describe the problem or need driving this PR and how it resolves the issue. Include links to related issues if applicable.

Observer-api had real, unaudited mutations. #4571 built a policy-driven audit framework for openchoreo-api and wired it into exactly one binary, so nothing had ever proven the framework was reusable. This PR extends it to observer and closes #4624.

Only one operation actually needed auditing: `UpdateIncident`. The other 13 non-GET operations are exempted with reasons. Nine are reads that happen to be POSTs, and four run on the unauthenticated internal port where there is no real actor to record. The work here was the plumbing and the reuse, not the operations.

## Approach
> Summarize the solution and implementation details.

Both of observer's specs feed the generator, so a new mutating alert-rule write can't pass CI unaudited. Each port builds its own audit middleware from its own spec, and they share one emitter so a single configured policy covers both ports plus `/mcp`. Audit sits inside auth, so the actor is the token's subject rather than `anonymous`, and a second instance sits outside it so a 401 is recorded too. A startup check refuses to boot if the generated table and the served specs disagree.

Two pieces moved out of openchoreo-api so observer could use them: `tools/internal/auditgen` for the spec-walking logic, and `internal/auditconfig` for config decode and validation. Both tools now take `-service` and pick from a registry, so adding a third API server is a registry entry rather than a new binary. openchoreo-api's generated definitions and its committed coverage matrix come out byte for byte identical, which is the proof these were refactors and not rewrites.

Observer's audit config is loaded from the same supplementary YAML as `auth.subject_types`, since `audit.policies` is a nested list the flat env-var mapping can't represent, and `observer.audit` is now plumbed through the observability-plane chart.

## Behavioural changes
> Changes visible to callers or operators. The first one is the one to review carefully.

1. A scoped grant of `incidents:update` now actually permits the update. `UpdateIncident` authorized against an empty resource hierarchy, which becomes the `*` wildcard path, and only a policy whose own resource is `*` matches that. Namespace and project scoped grants were being denied, so in practice the operation required a cluster-wide binding. It now authorizes against the incident's own namespace/project/component, read from the store because `IncidentPutRequest` carries no scope of its own.
2. Because that read runs first, an unknown incident ID returns 404 before the authorization check. A caller without `incidents:update` can now tell an existing incident from a missing one. That is unavoidable if you want to authorize on the resource's own scope, and openchoreo-api already accepts the same tradeoff.
3. Observer's access log moves from `Debug("HTTP request")` to `Info("ACCESS-LOG")`, so it goes from logging nothing at the default level to logging every request. In exchange it now generates and validates `X-Request-ID`, which is what ties access logs to audit records.
4. `observer.audit` gains `enabled`, `defaults` and `policies` in the observability-plane chart. Defaults are unchanged behaviour: enabled, publish everything.

## Related Issues
> Include any related issues that are resolved by this PR.

Closes #4624. Depends on #4460 and builds directly on #4571.

## Checklist
- [x] Tests added or updated (unit, integration, etc.)
- [ ] Samples updated (if applicable)
- [ ] Added `backport/<release-branch>` label if this should be backported (e.g., `backport/release-v1.0`)
- [x] This PR includes AI-generated code or content

## Remarks
> Add any additional context, known issues, or TODOs related to this PR.

`go build ./...` is clean, `golangci-lint` reports 0 issues, and tests pass across `./tools/...`, `./internal/observer/...`, `./internal/auditconfig/...`, `./internal/openchoreo-api/...` and `./internal/server/...`. Both generators and both coverage matrices regenerate with zero drift, and `helm template` renders the new audit block correctly.

The four internal-port mutations stay exempted because port 8081 has no authentication. The audit middleware is wired there anyway, so lifting an exemption once that port gets auth needs no wiring change.
