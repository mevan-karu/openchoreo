# Issue #4459 — updated title & description

**Current title:** Restore audit event emission and prevent silent removal
**Current body:** The audit framework is dead code today; this restores emission and removes the conditions that let it disappear, via a single middleware-chain constructor that production and tests share.

Done when a real request emits a complete event, the actor is the token's sub claim, and removing the wiring fails a test.

---

The current title/description only covers REST-only restoration + an anti-regression test. What actually shipped (P0 of the audit-logging epic, #3343) also covers the MCP surface, a policy engine, and the hardening work found during review — none of that is captured. Suggested replacement:

## Title

`Restore audit logging with a policy-driven framework across REST and MCP`

## Description

Restores audit event emission — dead since #2588 — and rebuilds it as a shared, policy-driven framework rather than a point fix, so it can't silently disappear again and covers both surfaces that mutate state.

**Delivers:**
- One surface-agnostic emitter, config-driven policy engine (`categories`/`resources`/`operations`/`actions`/`origins`/`actor_types`/`results` selectors), and two thin adapters — REST and MCP — driven from a single `operationDefs()` table so a binding can't drift from its operation.
- Shared middleware constructors (`APIMiddlewares`, MCP's `NewHTTPServer`) that production and tests both build from, so removing audit from the chain fails a test instead of shipping silently.
- `resource.type`/`resource.name` populated consistently on both surfaces, for both success and denial (a PDP-denied request never reaches the handler that would otherwise set it).
- Actor identity from the token's `sub` claim only, with a fabricated-identity guard.
- Construction-time validation (not silent no-ops) for: an `operationId` with no matching route, an MCP tool binding with no registered tool, and a policy selector value that doesn't match any real operation/actor type.

**Done when:** a real request through either surface emits a complete event (actor, action, category, result, resource); removing audit from either middleware chain fails a test; `audit.enabled` is the only kill switch (the application log level can't silently suppress emission); and a PDP denial on either surface still records `resource.type`.
