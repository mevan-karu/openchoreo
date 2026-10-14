**Title:** feat(audit): make the actor.id claim configurable

## Purpose
Audit records always used the token's `sub` claim as `actor.id`. Deployments whose `sub` is an opaque ID had no way to record a readable identity such as `email`.

## Approach
- Added `audit.actor.id_claim` (default `sub`) to the audit config for openchoreo-api and observer, exposed in both Helm charts.
- Added `audit.MiddlewareConfig`, built once via `cfg.Audit.MiddlewareConfig()` and passed to the REST, MCP, exec/wirelogs and unauthenticated audit middlewares.
- Middlewares read the claim from the validated JWT claims in the request context. A missing claim records `unknown`, never a fallback to `sub`.

## Related Issues
Part of #3343

## Checklist
- [x] Tests added or updated (unit, integration, etc.)
- [x] This PR includes AI-generated code or content

## Remarks
`audit.policies[].match.actors` compares against the recorded `actor.id`, so changing the claim changes what those selectors match.
