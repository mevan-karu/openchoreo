refactor(api): rename audit origin to surface and record the user agent

## Purpose
`origin` read as provenance, but the value answers "which way in". And `api` vs `mcp` implied MCP is not an API when it wraps the same one.

Separately, nothing in the record told a portal session apart from `occ`, CI or an agent.

## Approach
The field is now `surface` over `rest` and `mcp`, with the audit config key following from `origins:` to `surfaces:`.

`user_agent` is read from the request header in `EmitFromContext`, which was already being handed it, so no signature changed and both surfaces get it. Client-supplied, so treat it as a claim rather than proof.

Schema version stays at `1.0`. Both changes are breaking, but v1 has not shipped.

## Related Issues
Part of #3343.

## Checklist
- [x] Tests added or updated (unit, integration, etc.)
- [ ] Samples updated (if applicable)
- [ ] Added `backport/<release-branch>` label if this should be backported (e.g., `backport/release-v1.0`)
- [x] This PR includes AI-generated code or content

## Remarks
The observer read contract on `feat/audit-observer-contract` encodes `origin` in its query filter, record type and OpenAPI spec. It needs the same rename before it lands.

The four failing webhook and controller suites fail the same way without this change and none of them import the audit packages.
