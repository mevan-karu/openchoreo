# refactor(api): classify read operations in the audit exemption tables

## Purpose
> Briefly describe the problem or need driving this PR and how it resolves the issue. Include links to related issues if applicable.

The audit generator skipped every GET, and so did both coverage gates. That made the HTTP method a stand-in for "modifies state", which it isn't in either direction. `Evaluates` and the eight `Query*` endpoints are POSTs that only read. `GetSecret` is a GET that arguably should be recorded.

The consequence is one-sided. A POST that only reads can be exempted, but a GET that ought to be audited had nowhere to go. Worse, adding one is silent: no error, no warning, nobody prompted. Add `GET /secrets/{name}/value` tomorrow and it is unaudited forever.

## Approach
> Summarize the solution and implementation details.

Reads are now listed in each service's `RESTExemptions` with a reason, same as any other exemption. The generator walks every operation and fails on a GET that no table classifies, so a new endpoint cannot slip through unclassified. A GET that should be audited gets there through an `Overrides` entry, which was impossible before because the skip ran ahead of the override lookup.

The generator configs now read the exemption table from `internal/*/audit` instead of keeping their own copy. That copy had already drifted: it still excluded `QuerySpanDetailsForTrace`, an operation reverted out of the spec a while back.

Also removes `tools/auditcoverage` and the two generated matrices under `docs/audit`. Nothing referenced them, they never made it to the docs site, and the exemption reasons they rendered now live next to the entries themselves.

## Behavioural changes
> Changes visible to callers or operators.

None at runtime. `definitions.gen.go` is byte-identical for both services, so what gets audited today is unchanged. The `audit-coverage-matrix` make target is gone.

## Related Issues
> Include any related issues that are resolved by this PR.

Part of #3343.

## Checklist
- [x] Tests added or updated (unit, integration, etc.)
- [ ] Samples updated (if applicable)
- [ ] Added `backport/<release-branch>` label if this should be backported (e.g., `backport/release-v1.0`)
- [x] This PR includes AI-generated code or content

## Remarks
> Add any additional context, known issues, or TODOs related to this PR.

`make test` and `make lint` both pass, and `make audit-gen` produces no diff.

Three new generator tests cover the parts that did not exist before: an unclassified GET fails, an exempted one is skipped, and an overridden one produces a definition.

One thing deliberately dropped: the "Known non-events" prose that lived in `tools/auditcoverage`. It documented real gaps, things like `source_ip` being unobtainable for a direct MCP client and `?filterByAuthz=false` recording a denial as `failure`. Worth keeping somewhere. I would rather it went into the audit schema reference when that is written than sit in a tool nobody reads. Happy to file an issue if that suits.
