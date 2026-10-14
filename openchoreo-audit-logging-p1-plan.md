# Audit Logs Module — P1 Implementation Plan

**Companion to:** [`openchoreo-audit-logging-scope.md`](./openchoreo-audit-logging-scope.md) — the requirements register · [`openchoreo-audit-logging-phased-plan.md`](./openchoreo-audit-logging-phased-plan.md) — P0–P10 sequencing · [`openchoreo-audit-logging-p0-plan.md`](./openchoreo-audit-logging-p0-plan.md) — the preceding phase

**Proposal:** [Discussion #4396](https://github.com/openchoreo/openchoreo/discussions/4396) · **Epic:** [#3343](https://github.com/openchoreo/openchoreo/issues/3343) · **Phase:** P1 · **Preceded by:** P0 — restoration shipped as `6bd147a1`, core framework per that plan's §5b

**Requirements delivered:** PR-1, PR-2, PR-3, PR-7 · EM-3 (existing values), EM-4a, EM-4c, EM-6 · DO-2 · EM-16 (partial — hardening only)

**Status:** ✅ **Shipped**, with two verification tasks open (1.13, 1.14) and three implementation tasks added after the fact (1.15 — §0.7; 1.16, 1.17 — §0.4) · **Date:** 2026-08-13 · **Revised:** 2026-08-19 · **Reconciled against the shipped code:** 2026-08-24 (§0) · **Extended:** 2026-08-25 (§0.7), 2026-08-30 (§0.8 — PR #4571) · **Verified against:** PRs [#4462](https://github.com/openchoreo/openchoreo/pull/4462), [#4460](https://github.com/openchoreo/openchoreo/pull/4460) and [#4571](https://github.com/openchoreo/openchoreo/pull/4571) on `feat/audit-logging-p1`

> ### ⚠️ Read §0 first — this document is now a record, not a plan
>
> **P1 is implemented.** §0 is the as-built account: what shipped, which exit criteria are met,
> and what shipped that no plan asked for. **§§1–8 below are preserved as the plan as
> written** — they are the reasoning trail, and they are no longer edited in place except where
> a factual number was wrong. Where §0 and the body disagree, **§0 wins**.
>
> Appendices A (deviations from the phased plan) and A.2 (as-shipped in P0) remain accurate and
> useful. **Appendix E is new**: what shipped beyond plan, with the downstream phase that owns
> each item.

---

## 0. As built — 2026-08-24

### 0.1 Commit-citation policy

**This section has now gone stale twice, which is the failure it was written to warn about.**
The three SHAs originally cited (`6bd147a1`, `96fe08ca`, `8c8b33b7`) were superseded by a squash
into `0ad4c9bd`/`8d257ca9`, which this section was rewritten to cite instead — and that squash
was itself superseded: `0ad4c9bd` and `8d257ca9` now also fail
`git merge-base --is-ancestor <sha> HEAD`. They linger as loose objects in some working copies
— so `git cat-file -t` still answers `commit` — but would not resolve in a fresh clone. Two
rewrites of "cite the current SHA instead" is a pattern, not a one-off; per the policy line
below, this document does not attempt a third.

**Cite PRs, not SHAs — no exceptions, including in a correction.** P1 (this document's
deliverables) shipped as **PR [#4460](https://github.com/openchoreo/openchoreo/pull/4460)**, on
top of P0's PR [#4462](https://github.com/openchoreo/openchoreo/pull/4462). Review feedback on
**PR [#4571](https://github.com/openchoreo/openchoreo/pull/4571)** then produced a third commit —
see §0.8. Diff a PR on GitHub, or locally against its merge base, rather than against a commit
SHA recorded here.

> **The policy vindicated itself a third time (2026-08-30).** This section previously cited
> `6b54b8c4` as "P1, squashed". That SHA is gone too; the branch now carries `6ee4f787`
> (P1's deliverables) and `92a7e1ba` (the #4571 follow-up) on top of `a7f1d9c1` (P0). Those
> two SHAs are recorded here **only** to scope a diff today and will rot the same way — the
> durable citations are #4462, #4460 and #4571.

`main` is behind this branch's base (`git merge-base main HEAD`), so `git diff main...HEAD`
pulls in unrelated upstream commits — scope any audit diff to the PR's own commits instead.

> **On line-number citations (added 2026-08-30).** The table below used to cite `types.go:53`
> and `middleware.go:177`. Both drifted — 1.15 inserted `Hierarchy` above them — and were
> replaced with symbol names. **Cite symbols, not line numbers**, for the same reason this
> section cites PRs and not SHAs: a grep for a symbol survives an edit above it, a line number
> does not. (`adapter.go:88` happened to still be correct when checked, and was de-numbered
> anyway.)

### 0.2 Task status

| # | Task | Status | Evidence |
|---|---|---|---|
| 1.0b | `ResultUnauthenticated` + split both mappers | ✅ | `audit.ResultUnauthenticated` in `types.go`; `determineResult` in `middleware.go` maps 401→unauthenticated / 403→denied; `mcpaudit.classifyResult` |
| 1.1 | Confirm the authentication topology per surface | ⚠️ **no artefact** | Nothing recorded. Either settled verbally or skipped; carry the conclusion forward |
| 1.2 | Spec→definitions generator | ✅ | `tools/auditgen/`; output `internal/openchoreo-api/audit/definitions.gen.go`; in `code.gen` via `make/lint.mk:160` |
| 1.3 | Full MCP binding table | ✅ | `internal/openchoreo-api/audit/mcp_bindings.go` — 69 tools, **90 binding keys** |
| 1.4 | Exemptions with mandatory reasons | ✅ | `internal/openchoreo-api/audit/exemptions.go` — 2 REST, **8** MCP |
| 1.5 | The validation gate | ✅ *code* | `mcphandlers/audit_coverage_test.go` — 7 assertions (the 6 planned, plus a tool-count pin). Deliberate-breakage demonstrations outstanding → **1.13** |
| 1.6 | PR-7 second outer instance | ✅ **extended after the fact** | `unauthenticated_middleware.go`; both instances composed in `APIMiddlewares`; exactly-once proven including under panic. **Amended by #4571** — `NewUnauthenticatedMiddleware` now takes an `origin Origin` parameter and is wired onto the exec, wirelogs and `/mcp` chains too. See §0.8 |
| 1.7 | `SetResource` on body-carried creates | ✅ **overshot, then reordered** | **78** call sites across 35 handler files — creates *and* updates/deletes, adding `resource.id`. See Appendix E.2. **Amended by #4571** — 73 of those calls moved *above* the response-conversion step; see §0.8 for the rule and its unguarded status |
| 1.8 | Audit `/exec/` + wirelogs | ✅ *code* | `audit/exec_wirelogs.go`, `handlers/exec_wirelogs_audit.go`. Live verification outstanding → **1.14** |
| 1.9 | Harden `X-Request-ID` | ✅ | `audit.RequestIDFromHeader` in `adapter.go` — UUID-or-generate, with a `RequestIDRejections()` counter |
| 1.10 | **Decide** how `agent` is determined | ✅ **decided by construction** | `SecurityConfig.KnownActorTypes()` — `agent` is a *configured subject type*, not a code constant. Residual caveat in §0.4 |
| 1.11 | Coverage matrix, both surfaces | ✅ **exceeded** | `tools/auditcoverage/` → `docs/audit/coverage-matrix.md`, with its own freshness gate. Appendix E.12 |
| 1.12 | Confirm PR-2 and PR-6 on both surfaces | ◐ | PR-2 ✅ — 12 operations carry `category: authorization`. PR-6 has a documented hole on MCP — Appendix E.10 |
| 1.15 | Resource hierarchy — `project` / `component` / `resource` in the audit record | ✅ **shipped 2026-08-25, corrected same day** | Design and rationale in **§0.7**; implemented per that design, then a real "first Check wins" bug was caught by testing against a live k3d cluster (not by any unit test) and fixed to last-write-wins — see §0.7's precedence note |

### 0.3 Exit criteria — six of eight met, two qualified

| # | Criterion | Verdict |
|---|---|---|
| 1 | Every state-modifying REST operation audited or exempted | ✅ 114 audited + 2 exempted, gate-enforced |
| 2 | Every registered MCP tool bound or exempted | ✅ 69 bound + 8 exempted; 80 read-only by the verb rule |
| 3 | The gate fails on each of six conditions, **"each demonstrated by a deliberate breaking change, not merely asserted in code"** | ◐ **code complete, demonstration unevidenced** → task **1.13**. Note also that assertions 3 and 6 are structurally unable to fail — §0.5 |
| 4 | Rejections `unauthenticated`; PDP refusal `denied` with a real category; no double-audit | ✅ on REST and on MCP's filter path — proven by test. **Improved by #4571**: an HTTP 401 on `/mcp`, exec and wirelogs now emits too, stamped with the right `origin` (§0.8). ◐ still **not** on MCP's service-layer path — and #4571 established that hole is *structurally unfixable* at the middleware, not merely unfixed (Appendix E.10) |
| 5 | `resource.name` present on **all 114** REST operations | ◐ **not as written.** Path-derived on 72 and available even on denial; on the 42 body-carried creates it is handler-supplied **on success only**. The matrix says so itself: *"unavailable on a denied/failed request."* Closing this needs P3's body extraction — it is a dependency, not a task |
| 6 | The only `Category` values anywhere are `management` and `authorization` | ✅ gate assertion 5 |
| 7 | Coverage matrix generated, both surfaces, every exemption and reason | ✅ plus a *Known non-events* section the plan did not ask for |
| 8 | `/exec/` audited or its exclusion recorded | ✅ audited — but held to criterion 3's standard, the live verification is outstanding → task **1.14** |

### 0.4 Still open

- **1.13** — demonstrate each of the six gate failures by deliberate breakage, and record it.
- **1.14** — live-verify a `k3d` exec session and a wirelogs tail through the audit wrapper.
  `TestFindFlusher_SeesThroughAuditWrapper` covers `Flusher`; nothing covers a real hijacked
  connection, which is what B7 explicitly said not to take on trust.
- **`resource.name` on denied/failed creates** — structural, waits on P3.
- **PR-6 on MCP's service-layer path** — Appendix E.10. **Reclassified 2026-08-30** from "unfixed"
  to "unfixable here": #4571 traced it to the go-sdk's typed-tool adapter, which calls
  `CallToolResult.SetError(err)` and returns a **nil** error, so the receiving middleware sees
  only `IsError` and the original error's identity is already gone. Closing it needs either a
  per-tool-handler translation of `services.ErrForbidden` or a structured `*jsonrpc.Error`
  (client-visible protocol change) — a P7 design decision, not a middleware bug.
- **The `SetResource`-before-conversion ordering rule is unguarded** *(new, 2026-08-30)*.
  #4571 moved 73 call sites above the response-conversion step (§0.8). The AST gate
  (`TestHandlers_BodyCarriedCreatesCallSetResource` and its update sibling) asserts the call is
  **present**, not **where** — so a future handler that reintroduces the old ordering passes
  every test and silently loses `resource` on the convert-fails path. Either extend the AST gate
  to compare statement positions, or accept it and say so. Filed as **task 1.16**.
- **No test walks `GetOperations()` asserting hierarchy coverage** *(new, 2026-08-30)*. §0.7's
  closing paragraph asks for exactly this — *"a test that walks `GetOperations()` and asserts
  each mutating operation's hierarchy is populated or explicitly accounted for, rather than
  trusting the survey"* — and it does not exist. The 36-file `service_authz.go` survey is still
  the only evidence, which is the trust-the-survey position §0.7 explicitly rejected. Filed as
  **task 1.17**.
- **`source_ip`'s trusted-proxy assumption** — documented in `adapter.go` and in the matrix,
  unfixed. Its sibling `X-Request-ID` was hardened; this was not.
- **Task 1.1** — no artefact.
- **`agent` (1.10)** — decided as a configured subject type, but nothing *distinguishes* an
  agent-driven call. `origin: mcp` is the only signal, and EM-11 says that is not a proxy for
  it. Decided-with-a-caveat rather than closed.
- **Audit output format** — level-gating is fixed; *format* is not. `NewLogger` reuses the
  application handler, so under a text handler audit records ship unparseable. P5a's TR-7.
- ~~**1.15 — the record carries no `project` or `component`**~~ **Closed 2026-08-25.** Implemented
  per §0.7's design: `Hierarchy` lives as a sibling of `Resource` on `Event`/`AuditData` (not
  merged into `Resource`, so the 78 `SetResource` call sites needed no edits), captured once in
  `AuthzChecker.Check` via `audit.SetHierarchy` and pre-call-seeded on MCP via
  `audit.SeedHierarchy` (`namespace_name`/`project_name`/`component_name`/`resource_name`, the same
  convention `callToolScope` uses), and folded into the `resource` JSON/log group at render time in
  both `Event.MarshalJSON` and `Logger.LogEvent`. All five test groups §0.7 called for are in place
  (`context_test.go`, `emitter_test.go`, `logger_test.go`, a service-level `authz_test.go` case, and
  an `mcpaudit` seed test) — **plus a sixth added after the fact**, reproducing the precondition-check
  ordering bug below. **The shipped `SetHierarchy` precedence rule ("first Check wins") was wrong**
  and was caught only by testing MCP's `create_component` against a live k3d cluster — every unit
  test passed throughout, because none exercised a real downstream `Check` call. Fixed to
  last-write-wins same day; see §0.7's precedence note for the full account, including why a
  resource-type-keyed fix (the first alternative considered) was rejected as unsafe. The gap
  surfaced from a `workflowrun` authz asymmetry — recorded there
  as the motivating case, but **fixing that asymmetry was not audit's work**: it is a separate issue
  against `services/workflowrun/`, and 1.15 makes such an event *legible*, not impossible.

### 0.5 Two gate assertions cannot fail — recognised, and mitigated

`audit_coverage_test.go` says so in its own comments, so this is a design choice rather than an
oversight — but it is exactly the **D8 trap** Appendix A warns about, one level down:

- **Assertion 3** (every binding points at a real operation): bindings carry a fully resolved
  `*Operation` and construction already errors, so *"there is no way for a binding to reference
  a nonexistent operation and reach this line."* Kept as defence against a future refactor that
  loosens the guarantee.
- **Assertion 6** (no duplicate bindings): a Go map cannot hold a duplicate key. The real guard
  is upstream in `MCPBindings`/`MergeMCPAliases`. It compensates by **pinning
  `len(boundToolNames) == 69`**, which catches the case a bare error check would miss — a
  refactor that swallows the collision error.

The mitigation is count-pinning, not elimination. Worth knowing before someone re-derives the
objection and "fixes" it by deleting the assertions.

### 0.6 Corrected counts

The plan's census figures were wrong in five places. The generated table and the matrix are
derived from the live spec and registry, so they are authoritative.

| This document said | Actually | Source |
|---|---|---|
| *(the 112/114 split is stated nowhere)* | `generatedOperationDefs()` = **112** spec-derived (100 management / 12 authorization); `exec_wirelogs.go` adds **2** non-spec (`Exec`, `Wirelogs`); `GetOperations()` = **114** (102 / 12) | Both pinned separately: `tools/auditgen/generator_test.go` asserts 112 with the verb census `create:38 update:34 delete:38 trigger:1 generate:1`; `audit/definitions_test.go` asserts 114 with `notInSpec == 2`. **Change one without the other and you get a confusing double failure** |
| 69 state-modifying MCP tools | **77** state-modifying — 69 bound + 8 exempted | `docs/audit/coverage-matrix.md` |
| 88 read-only MCP tools | **80** read-only | same |
| explicit MCP exemptions reduce to **5** | **8** — the 5 `get_*_creation_schema` tools plus 3 deprecated `get_cluster_*_creation_schema` aliases | `exemptions.go` |
| a "69-**entry**" binding table | 69 **tools**, **90 binding keys** = 38 plain + 21 scope-collapsed × 2 + 10 aliases | `definitions_test.go` |

`157` total registered tools was correct, and is now pinned by gate assertion 0.

### 0.7 Task 1.15 — resource hierarchy in the audit record (added 2026-08-25)

**Status:** ✅ shipped, but not where planned. **Lands** originally said: a follow-up commit on
`feat/audit-logging-p1`, after PR [#4460](https://github.com/openchoreo/openchoreo/pull/4460) is
reviewed — deliberately **not** folded into it, because a wire-schema change is a different
review than a coverage change and shouldn't reopen a 73-file PR already in review. **That did not
hold**: this task's `Hierarchy` type, both setters, the `services/authz.go` capture and both
render paths are present in the same squashed commit (`6b54b8c4`) as the rest of PR #4460 — the
review pass that found this (`pr-4460-review.md`) confirmed it by inspecting the shipped diff, not
by trusting this line. The reasoning that produced the original decision still applies to a
*future* wire-schema change; it just didn't get followed here. **Requirement:** extends EM-3/PR-1's
resource identification; overlaps P2's tenancy dimension (see the E.1 and Appendix C amendments)

#### The gap

`resource` is `{type, namespace, id, name, metadata}`. Namespace shipped early (E.1); nothing
below it did. So a `create_workload`, `update_releasebinding` or `trigger_workflow` event names
the object but not the **project or component it belongs to** — the dimension of tenancy that
matters most below the namespace, and the one every authorization decision in this codebase is
actually keyed on.

**The case that surfaced it.** `UpdateWorkflowRun` authorizes against `wfRun.Labels` taken from
the **request body** (`services/workflowrun/service_authz.go:95`), then overwrites the path-named
object's labels unconditionally (`services/workflowrun/service.go:143`) — with neither the
`spec.owner.projectName is immutable` guard that `component` and `resource` carry, nor the
fetch-`existing` pattern that `releasebinding` / `workload` / `projectreleasebinding` use.
`ActionUpdateWorkflowRun` has `LowestScope: ScopeComponent` (`internal/authz/core/actions.go:382`),
so a subject genuinely can hold it for one project only.

> **That asymmetry is a pre-existing authz issue, filed separately against
> `services/workflowrun/`. It is not an audit bug and must not be fixed here.** What belongs to
> audit is that such an event is, today, indistinguishable in the log from a legitimate one.

#### Three decisions

| | | Why |
|---|---|---|
| **Scope** | New P1 task, follow-up commit | The coverage story is P1's; the schema change should not reopen a 73-file PR already in review |
| **Wire shape** | Flat siblings under `resource`: `project`, `component`, `resource` join the shipped `namespace` | Non-breaking, and mirrors `authz.ResourceHierarchy`'s json tags so a record is directly comparable to a policy scope. Settles E.1's open question |
| **Source** | Capture at `AuthzChecker.Check`, plus the MCP pre-call seed | One site per surface, no handler edits, correct on the denial path |

#### Why the authz check, and not the path

**REST paths are flat.** `/api/v1/namespaces/{namespaceName}/components/{componentName}`, not
`/projects/{projectName}/components/…`; exactly one path in the entire spec nests under
`projects/{`. So unlike `namespace`, project and component **cannot** come from `r.PathValue`.
They exist in three places only: the request body, the object the service returns, and
`authz.ResourceHierarchy` — which `AuthzChecker.Check` (`services/authz.go:36`) already receives
fully resolved, for every audited mutating operation, on both surfaces.

Capturing there buys, from one call site: correct values **on a denial** (the check records before
it returns `ErrForbidden`), zero edits to the 78 `SetResource` sites, and identical behaviour on
REST and MCP — the surface parity this plan keeps insisting on.

#### Design

1. **`audit/types.go`** — a local `Hierarchy` struct (four strings, json tags matching
   `authz.ResourceHierarchy`), hung off `Event` as a `Hierarchy` field tagged `json:"-"`, plus
   `AuditData.Hierarchy`.

   > **Mirror `Event.ResourceType`; do not extend `Resource`.** `ResourceType` is already a
   > top-level `json:"-"` field that both render paths fold *into* the `resource` group. This is
   > not cosmetic: `SetResource` **replaces** the whole `*Resource` rather than merging
   > (`context.go:29-35`), so a hierarchy stored inside `AuditData.Resource` would be silently
   > wiped by every one of the 78 handler calls that run *after* the authz check. Storing it
   > beside `Resource` and folding at emit time is immune to that ordering.

   Declare the struct locally rather than importing `internal/authz/core`. `audit` imports only
   `internal/config` and `middleware/auth` today; keeping it a leaf is what lets `services` depend
   on it in step 3 without a cycle.

2. **`audit/context.go`** — two setters, because two sources disagree:

   ```go
   func SeedHierarchy(ctx context.Context, h Hierarchy)  // pre-call adapter seed — lowest precedence
   func SetHierarchy(ctx context.Context, h Hierarchy)   // authz capture — last write wins
   ```

   Both no-op with no audit data in context, which is what keeps unaudited GETs and
   `FilteredList`'s per-item checks out of the record — those go through `BatchCheck`, deliberately
   **not** hooked.

   > **Corrected 2026-08-25, against a real cluster.** The line originally here read *"A second
   > `SetHierarchy` is ignored — the operation's own decision is the first `Check` in the chain;
   > anything later is nested or secondary."* **That premise is false, and shipping it produced a
   > live bug**, caught only by testing against a running k3d cluster rather than by any unit test
   > (every hierarchy test called `SetHierarchy`/`SeedHierarchy` directly; every mcpaudit test
   > stubbed `next` so no real `Check` ever fired downstream). MCP's `create_component` calls
   > `resolveComponentTypeKind`, which resolves the referenced `ComponentType` — itself an
   > authz-wrapped, `Check`-triggering lookup — *before* the create's own `Check` runs. Traced live:
   > seed (`namespace`+`project`) → `Check` for `componenttype:view` (`namespace` only) → `Check`
   > for `clustercomponenttype:view` (empty) → `Check` for `component:create` (the real, full
   > hierarchy) — and "first wins" locked in the `componenttype:view` result, silently dropping
   > `project`/`component` from every MCP `create_component` audit event.
   >
   > **Fixed to plain last-write-wins**, with no guard at all. This is safe not by assumption but by
   > a structural fact confirmed by tracing both surfaces: every `*ServiceWithAuthz` wrapper calls
   > its own `Check` as the first statement of its method, before delegating to the internal
   > (unwrapped) implementation — so any precondition lookup's `Check` necessarily completes before
   > the enclosing operation's own. REST's `create_component` (whose handler needs no such
   > precondition lookup) confirmed exactly one `Check` call in the trace; MCP confirmed the
   > precondition's `Check` first, the real one last. A resource-type-keyed lookup was considered
   > and rejected: `audit.Operation.ResourceType` (generated by `tools/auditgen` from OpenAPI path
   > segments) and `services.CheckRequest.ResourceType` (hand-written per `service_authz.go`) are
   > independently maintained and already disagree in casing and in kind (e.g. `secretReference` vs.
   > `gitsecret`) — joining on them would have silently emitted *no* hierarchy at all for every
   > operation where the two don't match, which is worse than the bug it would fix.
   > **`DeleteClusterRole`'s "role-in-use scan"** — the scenario "first wins" was originally
   > justified by — **does not exist in the code**; `DeleteClusterRole` (`services/authz/service_authz.go:83`)
   > makes exactly one `Check` and returns. If a future wrapped service ever calls another wrapped
   > service's method *after* its own `Check` (against the "use the unwrapped Service directly"
   > convention `component/service_authz.go` documents), that later call's hierarchy would win
   > instead — a latent risk of the same shape, inverted, and worth revisiting only if it's ever
   > actually observed.
   >
   > The rest of this section's precedence reasoning (seed vs. resolved, and the namespace-rule
   > inversion below) is unaffected and still holds — it concerns *which* value should win, not the
   > mechanism, and last-write-wins reduces to exactly that rule in practice.

   - **`SetHierarchy` overrides `SeedHierarchy`.** On MCP both fire: the seed reads `project_name`
     from raw call arguments — a *claim* — while `Check` may have resolved the project from a
     fetched object. Resolved beats claimed.
   - **This inverts the namespace rule below, for the same reason.** Namespace's seed *is*
     authoritative (a path parameter the write is pinned to); project's seed is not. The two rules
     are written together here precisely so they don't read as an inconsistency.

3. **`services/authz.go`** — one call at the top of `Check`, before `pdp.Evaluate`, so a PDP
   *error* path records too. **Namespace keeps its existing owner:** the hierarchy fills
   `Resource.Namespace` only when that field is still empty. The empty case is real, not
   theoretical — `CreateNamespace` has no `{namespaceName}` in its path but its check knows
   `ns.Name` (`namespace/service_authz.go:42`).

4. **`pkg/mcp/mcpaudit/audit_middleware.go`** — the seed at `:56` already reads `namespace_name`
   from parsed arguments. Add `project_name`, `component_name`, `resource_name`: **the identical
   field set `callToolScope` uses** (`pkg/mcp/tools/filter.go:338-364`), so the audit seed and the
   MCP authz scope read one convention and cannot drift apart. This is what covers MCP
   *filter*-layer denials, which never reach `AuthzChecker.Check` at all — the filter calls
   `pdp.GetSubjectProfile` directly. These three go through `SeedHierarchy`, **not** into
   `Resource`; `namespace_name` continues to seed `Resource.Namespace` exactly as today.

5. **`envelope.go` + `adapter.go` + `emitter.go`** — `EmitFromContext` (`adapter.go:141`) builds an
   `Envelope`, and `Envelope` has no hierarchy field, so all three change or the value never
   reaches `buildEvent`. In `buildEvent`, **shallow-copy `env.Resource` before filling namespace**:
   it is the same pointer `AuditData` holds and the same one every sink receives, and `Sink`'s
   contract already forbids mutating a shared event.

6. **Both render paths** — `logger.go`'s `LogEvent` and `types.go`'s `MarshalJSON`/`resourceJSON`.
   Emit `project`, `component`, `resource` inside the `resource` group after `namespace`, each
   omitted when empty. **E.5 exists for exactly this**: two independent paths, one guard test.

#### Resulting coverage

| Path | Source | On denial? |
|---|---|---|
| REST mutating operation | `AuthzChecker.Check` | ✅ the check runs before the write |
| REST `/exec/` | same — and its project is resolved authoritatively by `resolveComponentProject`, never caller-supplied (`handlers/exec.go:131-139`) | ✅ |
| MCP, filter-layer denial | pre-call argument seed | ✅ |
| MCP, service-layer path | `AuthzChecker.Check` | ✅ |
| REST 401 / unauthenticated | none — no handler, no check | ➖ namespace only, as today |

**What the ✅s rest on, precisely.** Every namespaced service wrapper surveyed builds its hierarchy
from `namespaceName` plus either the body's `spec.owner` or a fetched `existing` — including
`TriggerReleaseBindingCronJob`, which resolves project *and* component off the fetched release
binding (`k8sresources/service_authz.go:86-95`). Two shapes get less, both legitimately:
**cluster-scoped operations** pass `ResourceHierarchy{}` by construction — there is nothing below
namespace to record — and **any audited operation that reaches its handler without a `Check`**
would record nothing at all. The survey covered the 36 `service_authz.go` files, not all 114
operation IDs one by one, so 1.15 should close the gap the way this plan closes every other one:
a test that walks `GetOperations()` and asserts each mutating operation's hierarchy is populated
or explicitly accounted for, rather than trusting the survey.

#### Two hazards

- **What is recorded is the hierarchy the decision was made *on*, not necessarily the one the
  object persisted with.** For the `workflowrun` case that is the *claimed* project — which is
  exactly the forensic value, since the claim becomes visible and comparable — but it is not proof
  of stored state. Recording the effective hierarchy off the returned object needs the 78
  `SetResource` sites and is success-only: **handed to P2 task 2.12**, beside the
  `resource.name`-on-denied-create gap already parked there.
- **Redundancy is expected, not a defect.** For an operation *on* a component,
  `hierarchy.component` equals `resource.name`; likewise for a Resource CRD. That is how
  `authz.ResourceHierarchy` is built, and preserving it is what keeps a record comparable to a
  policy scope. Recorded here so nobody "fixes" it.

#### Tests

- `context_test.go` — `SetHierarchy` overrides `SeedHierarchy`; a later `SetResource` wipes
  **neither** (the regression the storage choice exists to prevent); both no-op without audit data;
  **`TestSetHierarchy_LastCheckWins`**, added after the live-cluster bug below, reproduces the exact
  observed sequence (seed → `componenttype:view` `Check` → `clustercomponenttype:view` `Check` →
  `component:create` `Check`) and asserts the last one wins.
- `emitter_test.go` — `buildEvent` fills an empty `Resource.Namespace` from the hierarchy, leaves a
  non-empty one alone, and does not mutate the caller's `*Resource`.
- `logger_test.go` and the existing two-render-path test — the three keys appear in both shapes and
  are absent when empty.
- A service-level test: a **denied** mutating operation still emits the hierarchy; `FilteredList`'s
  `BatchCheck` emits nothing.
- `mcpaudit` — the seed picks up `project_name` / `component_name` / `resource_name`.

**None of the above caught the live bug.** Every hierarchy test calls `SetHierarchy`/`SeedHierarchy`
directly, and every `mcpaudit` test stubs `next` so no real downstream `Check` ever fires — which is
exactly how "first Check wins" shipped green and broke in production. Only testing against a running
k3d cluster (REST and MCP `create_project`/`create_component`, success/denied on both surfaces)
surfaced it. `TestSetHierarchy_LastCheckWins` closes that specific gap; it does not generalize to
"always test against a live cluster before calling a task done," which is the larger lesson.

---

## 0.8 PR #4571 — auth rejections on exec, wirelogs and MCP (added 2026-08-30)

**Status:** ✅ shipped as `92a7e1ba`, *"fix(api): audit auth rejections on exec, wirelogs and
MCP"*, in response to review feedback on
**PR [#4571](https://github.com/openchoreo/openchoreo/pull/4571)**. **No plan asked for any of
it** — it is recorded here because §0 is the as-built account and this commit changes three
things §0 previously stated as settled.

### What changed

**1. `NewUnauthenticatedMiddleware` gained an `origin Origin` parameter, and three more chains.**
Task 1.6 composed the outer instance into `APIMiddlewares` only, which covers the generated REST
routes. Exec, wirelogs and `/mcp` are registered outside that chain, so a 401 on any of them
emitted nothing — the coverage matrix's *Known non-events* section said so in as many words for
`/mcp`. All three are now wrapped. `origin` is a parameter rather than the hardcoded `OriginAPI`
because otherwise every MCP token rejection would be recorded as if it had arrived over REST.

> **This closes the second bullet of Appendix E.10** and is the reason exit criterion 4's verdict
> above moved from a flat ◐ to "improved". It does **not** close the first bullet — see the
> reclassification in §0.4.

The event still carries a nil `Operation`, so `Action`, `Category`, `OperationID` and
`ResourceType` are empty and every operation-derived policy selector short-circuits. **A
rejection is selectable only by `origins`, `results`, `actor_types` and `actors`** — on either
surface. That constraint is now stated in the middleware's doc comment and rendered into the
matrix's *Known non-events* section, replacing the `/mcp`-emits-nothing entry it supersedes.

**2. `audit.SetResource` moved above the response conversion at 73 sites.** The rule task 1.7
inherited from P0's deliverable 0.6 was *"`SetResource` after the error check"*. That is
necessary but not sufficient: at 73 of the 78 sites the call sat *after* the `convert[...]` step
as well, so a mutation that **succeeded** and then failed to marshal into its generated response
type returned 500 with an empty `resource` — an audit record saying something was refused when
the object had in fact been created or deleted.

> **The rule is now: `SetResource` after the error check and *before* the response conversion.**
> It applies to every call site added from here on, including observer's.
>
> **It is not guarded.** The AST gate checks presence, not position. Filed as task 1.16 in §0.4.

**3. `Hierarchy.Namespace` was missing from one render path, and dropped on the other.** §0.7
step 6 claims both render paths shipped correctly. They did not:
- `Logger.LogEvent` never rendered `Hierarchy.Namespace` at all — only `Resource.Namespace`.
- `Event.MarshalJSON` overwrote the namespace with `e.Resource.Namespace` unconditionally, so an
  Event carrying a hierarchy but a nil (or namespace-less) `Resource` published
  `resource.project` while silently dropping `resource.namespace`.

Both now apply one precedence rule — hierarchy's namespace unless `Resource` carries its own —
matching `buildEvent`'s `withHierarchyNamespaceFallback`.

> **This is the second time a two-render-path invariant shipped broken**, and E.5 exists
> precisely to prevent it. `TestEvent_MarshalJSONMatchesLogEventShape` did not catch it because
> the two paths were only ever compared on events where `Resource` was populated — the shape
> agreed, the *precedence* did not. **The lesson for E.5's guard: comparing shapes is not
> comparing behavior.** A two-path test needs a case per branch of any precedence rule the paths
> share, not one representative event.

### Verified state after #4571 (2026-08-30)

Run against `92a7e1ba` with `GOROOT` unset (see the toolchain note below):

| Check | Result |
|---|---|
| `go build ./...`, `go vet ./...` | ✅ clean |
| `go test ./tools/... ./internal/openchoreo-api/... ./internal/server/... ./pkg/mcp/... -count=1` | ✅ 0 failures |
| `make lint` | ✅ 0 issues, headers and newlines clean |
| `make code.gen-check` | ⚠️ **fails, but not on generated code** — it greps `git status --porcelain`, which lists this plan set's untracked `*.md` files. No tracked generated file is stale. Worth knowing before someone "fixes" a generator |

> **Toolchain footgun.** `GOROOT` is set to a gvm path (`~/.gvm/gos/go1.26`) while
> `GOTOOLCHAIN=auto` resolves go1.26.3, producing
> `compile: version "go1.26.0" does not match go tool version "go1.26.3"` on every package.
> `unset GOROOT` fixes it. The repo's own `.claude/hooks/pre-commit-checks.sh` inherits the
> broken env, so it reports `make lint failed` when lint is in fact clean.

---

## §§1–8 — the plan as written

> **Historical from here down.** Preserved for the reasoning; superseded by §0 on any question
> of fact. Factual corrections are marked inline; the argument is left intact.

---

## 1. Goal, and why P1 splits

Four requirements:

1. **Audit API operations** — all 114 state-modifying REST routes, not the 12 P0 covers.
2. **Audit MCP operations** — the 69 state-modifying tools, which emit nothing today.
3. **One common config for the same operation**, not a separate config per surface.
4. **A validation mechanism for both surfaces** — every API and MCP operation is either audited or explicitly exempted, enforced at build time.

### What P0 already provides

**The core framework is P0's, not P1's.** Per that plan's §5b, P0 delivers the surface-agnostic `Operation` / `Envelope` / `Emitter` / `PolicySet`, the configuration model (ordered `policies`, first match wins), both the REST and MCP adapters, and resource resolution — proven end to end on the 12 operations its `definitions.go` covers.

That cut is deliberate: all the architectural risk sits in the core, so it is retired on a small surface where a mistake is cheap. **P1 is therefore coverage and enforcement only** — bulk mechanical work on a settled foundation, held honest by a build-time gate. P1 adds no new audit machinery.

§4 below restates the design for reference, since an implementer needs it to hand; §5 lists only what P1 builds.

**Attributes are out of scope for both.** The four requirements concern operations, config and validation. Configurable actor claims and object attributes — the replacement for the scope doc's AT group — land in P3, which also carries the body-extraction and strict-middleware questions.

---

## 2. Verified current state

Assessed against `feat/audit_logging` at `6bd147a1`. Each claim carries its reproducing command or file reference.

### 2.1 What P0 delivered

`6bd147a1` re-wired the audit middleware through `openapihandlers.APIMiddlewares` (`internal/openchoreo-api/api/handlers/handler.go`), so production and the test harness share one chain constructor. It trimmed `definitions.go` to 12 verified-live definitions, fixed actor identity to the token's `sub` claim without fabricating `"<nil>"`, populated `resource.id` from the K8s UID at 12 handler call sites, and added an `AuditConfig` toggle read inside the middleware.

**The shared-constructor invariant is load-bearing and extends to P1.** P0's regression guard is real only because `main.go` and `newTestHTTPHandler` both call `APIMiddlewares`. Every new composition point — the MCP middleware, the second HTTP instance — needs the same treatment: one exported constructor, driven by both production and a wiring test.

### 2.2 The REST surface

```sh
# 114 state-modifying routes
grep -oE 'm\.HandleFunc\("(POST|PUT|PATCH|DELETE) "\+options\.BaseURL\+"[^"]*"' \
  internal/openchoreo-api/api/gen/server.gen.go | wc -l          # → 114

# verb census — note zero PATCH
grep -oE 'm\.HandleFunc\("(POST|PUT|PATCH|DELETE) ' \
  internal/openchoreo-api/api/gen/server.gen.go | sort | uniq -c  # → 38 DELETE, 42 POST, 34 PUT
```

| Fact | Detail |
|---|---|
| **214 operationIds = 214 `func (h *Handler)` methods** | So operationId is a sound canonical key: 1:1 with a route |
| **111 of 114 routes are uniform CRUD** | The 3 exceptions: `POST /api/v1/authz/evaluates`, `POST …/components/{componentName}/generate-release`, `POST /api/v1alpha1/…/releasebindings/{name}/trigger` |
| **38 distinct resource kinds** | 4 authz kinds → `permission`, 34 → `resource`. See §4.3 for the derivation command and its trap |
| **2 routes are reads expressed as POST** | `evaluates` evaluates authz decisions; `generateRelease` renders without persisting. Both must be exemptible. `trigger` is a genuine action |
| **`handleAutoBuild`** (`POST /api/v1alpha1/autobuild`) | HMAC webhook, not a user action → exempt |
| Current coverage | **12 of 114**, in `internal/openchoreo-api/audit/definitions.go` |
| **72 of 114 carry the resource name in the path** | 38 DELETE + 34 PUT. The 42 POSTs carry it in the body |

### 2.3 The MCP surface — audited by nothing today

`/mcp` is registered on `baseMux` (`cmd/openchoreo-api/main.go:217`) with its own chain: `mcpLoggerMw → mcpAuth401Mw → jwtMiddleware` (`:210-216`). `APIMiddlewares` feeds only `gen.HandlerWithOptions` (`:224`), so **MCP mutations emit no audit events**. MCP is enabled by default (`MCPDefaults()` → `Enabled: true`), so this unaudited surface ships on.

| Fact | Evidence |
|---|---|
| **157 registered tool names** = 130 canonical + 27 deprecated aliases | `pkg/mcp/tools/register.go:24-52`. A `grep 'const name ='` misses ~60 — every `scoped_*.go` tool passes its name as a string literal to a helper |
| ~~**69 state-modifying** (60 canonical + 9 deprecated)~~ | 23 create, 14 update, 22 delete, 1 trigger. ~~The other 88 are read-only~~ · **Corrected 2026-08-24: 77 state-modifying (69 bound + 8 exempted) and 80 read-only** — see §0.6. The 69 figure counted the tools that end up *bound*, not the tools that declare a mutating verb |
| **Zero orphans** | Every state-modifying tool reaches a service method some REST handler also reaches. The join exists |
| **81 of 114** REST write operations are MCP-reachable; **33 are not** | All plane writes, namespace update/delete, secrets, gitsecrets, alert channels, workflowrun update/delete |
| **5 tools declare `create` actions but make no service call** | The `get_*_creation_schema` tools (`scoped_component_types.go:98` and four siblings) return a static schema. Any "create action ⇒ state-modifying" heuristic misclassifies them |
| 5 tool names are registered twice; the `pe` variant wins | `AddTool` replaces (`$SDK/mcp/server.go:277-278`), `perms[name]` overwrites (`register.go:348`). Not a live divergence — all toolset fields hold one handler instance (`main.go:320-346`) |

### 2.4 The MCP hook is a protocol middleware — better than the phased plan assumed

The phased plan's §2.4 concluded MCP needed **service-layer emission** because *"route-pattern middleware cannot capture MCP calls at all"*. Verified: a **protocol-level receiving middleware** is the right hook, and it is strictly better.

| Capability | Evidence |
|---|---|
| **Ordering: each `AddReceivingMiddleware` call wraps *outside* prior ones** | `addMiddleware` wraps the current handler (`$SDK/mcp/shared.go:86-90`). So a second call placed after `pkg/mcp/server.go:63` is **outermost** and observes the authz filter's denials |
| Tool name and **full raw argument JSON** | `req.GetParams().(*mcp.CallToolParamsRaw)` → `.Name`, `.Arguments`. A *superset* of the tool closure's view; for the 42 scope-collapsed tools there is no typed args struct at all |
| **Per-request** HTTP headers | `req.GetExtra().Header` is the live header map of the POST carrying that call (`$SDK/mcp/streamable.go:1143-1149`). `mcpLoggerMw` is outermost in the MCP chain, so `X-Request-ID` is present |
| Result and error | `err == nil && !IsError` → success; `err == nil && IsError` → tool-execution failure; `err != nil` → protocol error. `CallToolResult.GetError()` recovers the original Go error (`$SDK/mcp/protocol.go:132`) |
| Subject | `auth.GetSubjectContextFromContext(ctx)` is **ctx-based** (`auth/interface.go:43`), so it works here. `jwt.GetClaims` is request-based and does **not** — see §4.3's selector parity rule |

**Consequence:** no service-layer plumbing, and the double-counting problem never arises — HTTP audits by route, MCP audits by tool, and the two never both fire. `/mcp` is outside the applied middleware slice, and even inside it `r.Pattern` would be `/mcp`, which has no operationId.

### 2.5 The tool↔operation mapping cannot be derived

This is why a declared binding table is unavoidable.

| Obstruction | Detail |
|---|---|
| **1 tool → 2 operations** (21 cases) | Scope-collapsed writes route on an MCP-only `scope` argument (`scoped_helpers.go:248-251`). `create_component_type` is `createComponentType` **or** `createClusterComponentType` |
| **2 tools → 1 operation** (10 cases) | `trigger_workflow_run` and `create_workflow_run` both reach `WorkflowRunService.CreateWorkflowRun` with different argument shapes and different mcphandlers. Plus 9 deprecated aliases colliding with their canonical tool's cluster branch |
| **Wrong-sounding target** (1 case) | `create_component_release` calls `ComponentService.GenerateRelease` (`mcphandlers/components.go:144-147`) — REST `generateRelease`, **not** `createComponentRelease`. A name join gives the wrong answer |
| **Naming breaks** | `patch_component`→`updateComponent`; `create_authz_role`→`createNamespaceRole`; `list_dataplanes`→`listDataPlanes`. MCP is inconsistent with itself: `workflowplanes` without an underscore, `observability_planes` with one |
| **Argument names are ad hoc** | Only 12 of 37 identifier-carrying tools align mechanically with their REST path parameter. 25 hardcode `name` while REST uses `{ctName}`, `{envName}`, `{traitName}`… So the resource-name argument must be **declared per binding** |

### 2.6 Traps and constraints

| Item | Evidence / consequence |
|---|---|
| ~~**`denied` is indistinguishable from `failure` on MCP**~~ | **RESOLVED in P0 (`96fe08ca`).** `pkg/mcp/tools/filter.go:32-37` now exports `ErrNoSubject`, `ErrForbidden` and `ErrPDPFailure`, and `mcpaudit.classifyResult` maps the first two to `denied` while deliberately mapping `ErrPDPFailure` to `failure` — so a PDP outage is never recorded as a policy denial. Better than this plan proposed; D7 is done |
| **`SubjectContext` is frozen at session creation** | `Server.Connect` receives the initialize request's ctx (`$SDK/mcp/streamable.go:496-499`); jsonrpc2 wraps it in `notDone{ctx}`, preserving values for the session's life (`conn.go:218-219`). **Being fixed separately — see Appendix D** |
| **`source_ip` unobtainable for direct MCP clients** | `RequestExtra` has exactly three fields; no `RemoteAddr`. Only `X-Forwarded-For`/`X-Real-IP` from `Extra.Header`. Parity gap vs `middleware.go:165-180` |
| Concurrent `tools/call` on one session run in **parallel goroutines** | `$SDK/mcp/server.go:1429-1431`. `Emitter` and the policy set must be immutable after construction |
| Legacy clients batch N calls per POST | Protocol `2025-03-26` (`$SDK/mcp/streamable.go:1132-1176`), all sharing one header set. N events legitimately share one `request_id` |
| `r.Pattern` and `r.PathValue` work in the **middleware** position | `ServeMux.ServeHTTP` sets `r.Pattern`, `r.pat`, `r.matches` (`$GOROOT/src/net/http/server.go:2826`); `WithContext` shallow-copies the struct (`request.go:372-375`), preserving them |
| ~~The resolver silently drops uncompilable patterns~~ | **RESOLVED in P0 (`96fe08ca`).** `resolver.go` is deleted; `pattern_map.go`'s `BuildPatternMap` returns an error on an operationId with no route and on two operations colliding on one pattern. D6 is done, and construction time is now the loud failure point — see A.2 item 13 for what that does to the gate |
| JWT middleware short-circuits on 401/403 | `writeErrorResponse(...)` then `return` at every rejection path (`jwt/middleware.go:139-164`). One instance cannot be both inside auth (for `SubjectContext`) and outside it (to see rejections) |
| `X-Request-ID` is trusted from the client | `logger/middleware.go:38-50` reads the inbound header unvalidated |
| **Name-collision trap** | 6 of P0's 12 action names are string-identical to MCP tool names. But `create_secret` (REST `/api/v1alpha1/…/secrets`) is **not** MCP's `create_secret_reference` — different resource, API version, route and service. Nothing may join on action name |
| **Resource-kind extraction trap** | Kind comes from the *resource* segment, not the last path segment. `.../releasebindings/{name}/trigger` is `releasebindings`; `.../components/{name}/generate-release` is `components`. Naïve last-segment extraction invents `trigger` and `generate-release` as kinds |
| Parameter binding precedes all middleware | The generated wrapper binds path params and returns 400 before `HandlerMiddlewares` run (`server.gen.go:5003-5028`). A malformed-path-param 400 never reaches audit |
| `/exec/` and wirelogs sit outside the OpenAPI chain | `main.go:232-269`. The stated reason — ResponseWriter wrappers break `http.Hijacker` — **no longer applies**: `8c8b33b7` added `responseWriter.Unwrap()`, so `http.ResponseController` reaches `Hijacker`/`Flusher` through the wrapper. Whether the routes are *moved* into the chain is still a separate call; see B7 |
| `X-Request-ID` is trusted, and so is `source_ip` | `8c8b33b7` documented the trusted-proxy assumption on `SourceIPFromHeader`: both `X-Forwarded-For` and `X-Real-IP` are taken at face value with no trusted-peer check, so a client not behind a stripping proxy chooses its own `source_ip`. B8 hardens the request ID; **`source_ip` needs the same treatment or an explicit documented exemption** |
| A second MCP server exists | `cmd/observer/main.go:311-316`; its middleware setup is unverified. Relevant when PR-5 brings audit to Observer. **Confirmed 2026-08-24:** observer has *no* audit code at all — `grep -ri audit internal/observer/ cmd/observer/` returns nothing — and all 13 of its MCP tools are read-only. See `openchoreo-audit-logging-observer-plan.md` |
| **`?filterByAuthz=false` breaks the denied/failure distinction on MCP** *(found in implementation, 2026-08-24)* | In a session opened with that query parameter the authz filter is skipped, so a refusal surfaces as a **service-layer** `services.ErrForbidden` returned from the handler. The go-sdk converts a handler-returned error into `CallToolResult.IsError` *before* the middleware sees it, discarding the error identity — so `classifyResult` records `failure`, not `denied` (`pkg/mcp/mcpaudit/audit_middleware.go:86-98`, which documents this rather than fixing it). **Qualifies exit criterion 4** and means a `results: [denied]` query on the read path silently misses these. See Appendix E.10 |
| **`pkg/mcp.NewSTDIO` registers no middleware** | Neither audit nor the authz filter (`pkg/mcp/server.go:110`). Sound today — there is no HTTP request and so no identity — and it has **no production caller** in this repo. But it is exported, so whatever first wires MCP-over-stdio gets an unaudited mutating surface. Belongs in the coverage matrix's *Known non-events* |

---

## 3. Non-goals

- **Configurable actor claims and object attributes** — the AT-group replacement, P3. The config model reserves the block; P1 does not populate it.
- **Body-field extraction and the oapi-codegen strict middleware** — P3.
- **First-class tenancy fields, `producer`, `producer_instance`, `sequence`, `audit_id`, `schema_version`** — P2. *(`origin` was on this list; it shipped in P0 as a real `Event` field alongside `operation_id` — see A.2 item 6.)*
- ~~**`resource.uid`** (EM-14, a *Should*) — P2. P1 targets `resource.type` + `resource.name`.~~
  **Not held, 2026-08-24.** P1 populated `resource.id` from the K8s UID at **78 REST call
  sites** — creates, updates and deletes — so EM-14 is largely delivered on REST. It is *not*
  delivered on MCP: only 6 sites in 2 files. **P2's task 2.12 re-scopes from "add it" to
  "finish MCP and close the surface-parity gap."** Appendix E.2.
- **`Resource.Metadata`.** `8c8b33b7` added a `map[string]any` to `Resource`. It has **none** of the AT-group guarantees — no declaration, no type checking, no allowlist — so **P1 must not populate it from any of the `SetResource` call sites it touches.** P3 owns the declaration layer, and it must land before anything writes here; otherwise P3 inherits exactly the free-form-metadata problem the AT group exists to remove, and the scope doc's SE-1 secret-leakage argument applies directly. Leaving it empty in P1 keeps that option open.
  **✅ Held — verified 2026-08-24.** No production writer exists for either `Resource.Metadata`
  or `AuditData.Metadata` (the only assignment in the repo is the copy inside
  `Event.MarshalJSON`). P3's declaration layer therefore arrives unprejudiced. This was the
  non-goal most at risk from 78 new call sites, and it survived them.
- **Delivery modes.** `delivery` and `retries` are reserved keys in the policy schema so P10 is additive, but P1 implements only `async`.
- **Sinks.** The `sinks` block is reserved; P5 populates it.
- **The MCP frozen-`SubjectContext` fix** — being handled separately, see Appendix D.
- **The 33 REST-only write operations gaining MCP tools** — not an audit concern.
- **Auditing MCP session establishment** (`initialize`), and **auditing reads** — both out of scope.

---

## 4. The design

### 4.1 Surface-agnostic core — `internal/server/middleware/audit/`

```go
// Operation is a surface-neutral auditable operation. The OpenAPI operationId is
// the canonical key: it is 1:1 with a REST route (214 operationIds = 214 handler
// methods), and MCP tools bind onto it.
type Operation struct {
    ID           string // "createComponentType"
    Action       string // "create_component_type" — the semantic audit action
    ResourceType string // "componenttypes" — derived, so every event carries it
    Category     Category // stamped, NOT operator-configurable — see §4.3
}

// Envelope is everything a surface adapter must supply. No HTTP or MCP types.
type Envelope struct {
    Origin    Origin // api | mcp
    Actor     Actor
    Result    Result
    Resource  *Resource
    RequestID string
    SourceIP  string
    Metadata  map[string]any
}

// Emitter evaluates policies and emits. It knows nothing about either surface.
type Emitter struct {
    logger   *Logger
    policies PolicySet
}

func (e *Emitter) Emit(ctx context.Context, op *Operation, env Envelope)
```

`Emitter` and `PolicySet` must be immutable after construction — MCP calls arrive on parallel goroutines.

### 4.2 Two adapters, each supplying one resolution function

| Adapter | Resolves operation | Envelope from |
|---|---|---|
| **`httpaudit`** — P0's middleware, refactored | `r.Pattern` → operationId, exact-map lookup | HTTP status → result; `X-Request-ID`; `X-Forwarded-For`/`X-Real-IP`/`RemoteAddr`; `SubjectContext`; `r.PathValue` for the resource name |
| **`mcpaudit`** — new receiving middleware | `(toolName, scopeArg)` → operationId | error/`IsError` → result; `Extra.Header` for `X-Request-ID` and forwarded IP; ctx `SubjectContext`; a declared argument for the resource name |

> **Verify during implementation:** all generated routes are registered method-qualified (`m.HandleFunc("POST "+…)`), so `r.Pattern` should already include the method and be usable as the map key directly. Confirm the exact string form before building the key.

### 4.3 The configuration model

Four blocks, each keyed on its own natural unit. This is the heart of requirement 3.

> **Superseded 2026-08-19 — the YAML below will not validate.** P0 deliberately modelled *only* `enabled`, `defaults.publish` and `policies[].{match,set.publish}` (`internal/openchoreo-api/config/audit.go`), on the principle stated in `policy.go`'s `PartialSettings` comment: reserved keys are *not* carried as inert config that does nothing. `parsePartialSettings` **rejects** any unrecognised `set:` key with `unknown setting %q`. So `pre_action`, `delivery`, `retries`, `actor_claims`, `attributes`, `redact` and `sinks` do not exist and would each be a startup validation error today.
>
> Treat the block below as the **target shape across P1→P10**, not as writable config. Each reserved key becomes real in the phase that acts on it: `pre_action` in P10a, `delivery`/`retries` in P10b, `actor_claims`/`attributes`/`redact` in P3, `sinks` in P5. P1 adds none of them.

```yaml
audit:
  enabled: true                       # master switch (P0, shipped)

  # ── 1. WHAT EXISTS — code, not operator configuration ──────────────
  #   definitions.gen.go   operationId → action, resource kind, category
  #   mcp_bindings.go      (tool, scope) → operationId + resource-name arg
  #   exemptions.go        deliberately unaudited, each with a reason
  #
  # `category` is stamped from the operation and is NOT operator-configurable.
  # It is immutable in the published record and is what downstream systems
  # group and retain on, so changing it is a breaking change belonging in
  # code review. Operators get their flexibility from policies instead.

  # ── 2. POLICIES ────────────────────────────────────────────────────
  # Ordered; FIRST match wins, as Kubernetes audit policy resolves levels.
  # A rule matches on event fields and sets behaviour. Unset fields inherit
  # from `defaults`.
  defaults:
    publish:    true
    pre_action: false        # P10a — reserved key, not implemented in P1
    delivery:   async        # P10b — reserved key, only `async` in P1
    retries:    3            # P10b — reserved

  policies:
    # Authorization changes: block, and record intent up front.
    - match: { categories: [permission] }
      set:   { delivery: blocking, pre_action: true }

    # Silence a service account's high-volume workflow triggers.
    - match: { actions: [create_workflow_run], actor_types: [service_account] }
      set:   { publish: false }

  # ── 3. INFORMATION — reserved for P3 ───────────────────────────────
  actor_claims: []            # global
  attributes: {}              # per operation / object type
  redact: []                  # per attribute

  # ── 4. SINKS — reserved for P5 ─────────────────────────────────────
  sinks:
    log: { enabled: true }
```

**Why each axis is keyed where it is:**

| Axis | Key | Because |
|---|---|---|
| What exists | operation | A property of the API surface, not a preference. Generated, gate-enforced |
| `category` | operation (from resource kind) | Stamped and immutable per record. Operator-configurable category fragments retention — see Appendix B |
| publish / mode / pre-action | an ordered rule | Genuinely "when X, do Y". No fixed grouping fits, which is why a per-category map kept failing |
| Actor claims | global | The actor is the actor; no reason to vary per operation |
| Object attributes | operation / object type | Inherently per-type — a promotion has environments, a role binding has role and effect |
| Redaction | attribute | A property of the field |
| Transport | sink | Batching, retry, queue, endpoint auth are sink mechanics |

#### 4.3.1 Deriving `category`

> **Superseded 2026-08-19** (external review of PR #4462, folded back into `openchoreo-audit-logging-p0-plan.md`'s post-implementation corrections note): the target taxonomy this section maps 38 resource kinds onto — `resource`/`permission`/`access` — is not what P0 shipped and is not being renamed to. P0's `management`/`authorization` is now decided as permanent. Whoever picks up P1 should re-derive this section's generator mapping against `management`/`authorization` (2 categories, not 3) rather than implement the mapping below as written. EM-4b's rule (a denied-but-resolved operation keeps its category; only a rejection with no resolved operation needs a third bucket) still applies, independent of what the buckets are named.
>
> **Decided 2026-08-19 — no third category. The distinction moves to `result`, which gains `unauthenticated`.** `management`/`authorization` stays the complete, permanent `Category` set, with no qualification: **`access` is not added.**
>
> **Why `result`, not `category`.** The two axes answer different questions. `Category` classifies *what kind of operation this is* — a broad grouping over OpenChoreo's operation types, which is why it is stamped from the operation and is not operator-settable. `Result` records *what happened*. A rejected request is not a different kind of operation; it is a different outcome. Putting it in `Category` conflates the axes and forces a value into that enum that no operation can ever declare — which is precisely why the gate assertion "no `OperationDef` may declare `access`" was needed. With the distinction on `result`, that assertion disappears and every `Category` value stays declarable.
>
> **Surface parity is the decisive argument, and it is what killed the `access` design.** On MCP, `tools.ErrNoSubject` fires on a **bound** tool, so `op != nil` and the event legitimately carries `category: management`. On REST the equivalent 401 would have carried `category: access`. The same logical event would have had a different category on each surface — breaking §4.3.2's surface-parity gate, the rule this plan itself sets for admissible selectors. Distinguishing on `result` keeps both surfaces on their real category and gives them the same result value.
>
> **The information already exists and is being discarded.** Both surfaces collapse two distinct outcomes into `denied` today: REST's `determineResult` maps `401 || 403 → ResultDenied` (`middleware.go:157-166`), and MCP's `classifyResult` maps `ErrNoSubject, ErrForbidden → ResultDenied`. MCP's sentinels already name exactly the two cases — `ErrNoSubject` is "no authenticated user", `ErrForbidden` is "missing required permission" — so this is recovering a distinction the code makes one layer down, not inventing one.
>
> **Implementation.** Add `ResultUnauthenticated Result = "unauthenticated"` to `types.go`; add it to `validResults` in `config/audit.go` so policies may select it; split `determineResult` so **401 → `unauthenticated`, 403 → `denied`**; split `classifyResult` so **`ErrNoSubject` → `unauthenticated`, `ErrForbidden` → `denied`** (`ErrPDPFailure` → `failure`, unchanged).
>
> **No emitter change is needed.** `buildEvent` already sets `event.Result = env.Result` *outside* the `if op != nil` block, so a rejection carries a real result with no operation. This removes the wrinkle the `access` design created — no synthetic `Operation`, no `Category` override on `Envelope`.
>
> **This also fixes a live conflation, independent of PR-7.** On the 12 already-audited operations, a 403 PDP denial and a 401 are both recorded as `denied` right now — so 1.0b improves the trail on operations P0 already covers, before any new coverage lands. It was briefly filed as a P0 task on that basis and **moved back to P1 (task 1.0b) on 2026-08-19**: P0 is closed and shipped, PR-7 is this change's main consumer, and reopening a shipped phase for it buys nothing. **Land 1.0b before 1.6** — B5 has no result value to emit until it exists.

38 resource kinds: **4 authz → `permission`**, **34 → `resource`**. ~~`access` is a third value the framework assigns when a request is rejected before any operation resolves (EM-4b); no operation carries it.~~ *(Both sentences are superseded — see the two notes above. The real mapping is 4 authz → `authorization`, 34 → `management`, and there is no third value: rejections are `result: unauthenticated`.)*

```sh
grep -oE '"(POST|PUT|DELETE) "\+options\.BaseURL\+"[^"]*"' \
  internal/openchoreo-api/api/gen/server.gen.go \
  | sed -E 's/"//g; s/ \+options\.BaseURL\+/ /' | awk '{print $2}' \
  | sed -E 's|/api/v1(alpha1)?/||' \
  | awk -F/ '{for(i=NF;i>=1;i--) if($i !~ /^\{/ && $i !~ /^(trigger|generate-release)$/){print $i; break}}' \
  | sort -u | wc -l          # → 40, minus `evaluates` and `autobuild` = 38
```

The generator maps kind → category with a small override list naming the four authz kinds. Note the extraction trap in §2.6: the kind is the *resource* segment, not the last path segment.

#### 4.3.2 Policy selectors

> **Superseded in part 2026-08-19.** Nine selector axes shipped in P0: `categories`, `resources`, `operations`, `actions`, `origins`, `actor_types`, `actors`, `entitlements`, `results` (`audit.Selector`). Three in the table below **did not**: `resource_names`, tenancy (`namespaces`/`projects`/…) and `source_ips`. Tenancy is correctly P2 (it needs P2's tenancy fields); `resource_names` is now cheap given `Operation.RESTResourceParam` (A.2 item 11) but is not P1 scope unless a use case appears; `source_ips` this plan already said not to ship.
>
> **Updated 2026-08-24 — a `namespaces` selector is now cheap too.** `resource.namespace` ships on
> both surfaces (Appendix E.1), so the *data* a namespace selector needs already exists on every
> event; only the selector axis is missing. It stays P2 scope, since P2 owns where tenancy lives
> and adding the axis before that decision would pin it to `resource.namespace` prematurely.
> Note both remaining unshipped axes are now blocked on a *decision*, not on data.
>
> The `pre_action` / `delivery` column is **moot for P1** — neither setting exists (see §4.3's note), so every shipped selector is evaluated only against `publish`, which is decided after the handler returns. The timing gate still governs P10, when those settings arrive; it constrains nothing today.

A selector is admissible only if it passes two gates:

- **Timing.** `publish` is decided after the handler returns, so it may match on outcome. `pre_action` and `delivery` are decided *before* the action, so anything knowable only afterwards is unavailable to them.
- **Surface parity.** Requirement 3 means one rule must behave identically on both surfaces. A selector available on only one is excluded.

| Selector | `publish` | `pre_action` / `delivery` | Note |
|---|---|---|---|
| `categories`, `resources`, `operations`, `actions`, `origins` | ✓ | ✓ | Intrinsic to the operation |
| `actor_types` | ✓ | ✓ | `SubjectContext.Type`, ctx-based, reliable on both |
| `resource_names` | ✓ | ✓ | K8s's `resourceNames`. Path value / declared argument |
| tenancy (`namespaces`, `projects`, …) | ✓ | ✓ | K8s's `namespaces`. Needs P2's tenancy work |
| `actors`, `entitlements` | **rejected** | ✓ | **Escalation only** — see below |
| `results` | ✓ | **✗** | Not known before the action |
| `source_ips` | partial | partial | Absent for direct MCP clients — document, do not ship in P1 |
| arbitrary JWT claims | **✗** | **✗** | `jwt.GetClaims` is request-based with no ctx equivalent, so unavailable on MCP. Breaks surface parity |

**`actors` and `entitlements` are escalation-only, enforced at config validation.** A rule setting `publish: false` may not use them. SE-2 says producers cannot suppress their own events; a rule keyed on entitlement means anyone who obtains that group becomes unaudited, and group membership is usually delegated. A wrong match on an *escalation* rule yields more auditing — fail-safe — so the restriction is directional rather than blanket. `actor_types` stays permitted for suppression because `service_account` is not something a human subject can assume, and it covers the real use cases (the autobuild webhook, Backstage's background catalog sync).

Note also that `EntitlementValues` can be legitimately empty — `detectUserTypeFromClaims` returns `(true, []string{})` for a present-but-empty claim — so entitlement rules silently miss those subjects.

#### 4.3.3 Validation rules

> **Superseded in part 2026-08-19 — four of these seven shipped in P0.** Already enforced: `category` is never settable (`parsePartialSettings` rejects both `category` and `categories` with a specific message); a `publish: false` rule may not use `actors`/`entitlements` (`validatePolicy`); no rule may use arbitrary JWT claims (there is no such key, and unknown `set:` keys are rejected outright); and **every selector value must resolve to something real** — `collectSelectorValues` derives the legal `resources`/`operations`/`actions` sets from the live operations table and `SecurityConfig.KnownActorTypes()` supplies the legal `actor_types`, all checked through `config.MustBeOneOf`.
>
> One rule is **dropped**: "`access` is not assignable by any rule" — per §4.3.1 there is no `access` category. The rejection case is `result: unauthenticated`, and results were never operator-assignable in the first place, so nothing replaces it. One is **moot until P10**: "a `pre_action` or `delivery` rule may not use `results`" has no settings to constrain yet.
>
> That leaves **one rule for P1 to add**: the `policies`-list replacement semantics, which is documentation plus a `--dump-config` check rather than validation code.

| Rule | Reason |
|---|---|
| `category` is never settable by a policy | It is stamped and immutable in published records |
| ~~`access` is not assignable by any rule~~ | **Dropped 2026-08-19** — there is no `access` category. Rejections are `result: unauthenticated` (§4.3.1), and no rule may assign a result |
| A `publish: false` rule may not use `actors` or `entitlements` | §4.3.2 |
| A `pre_action` or `delivery` rule may not use `results` | Not known in time |
| No rule may use arbitrary JWT claims | Surface parity |
| The operator's `policies` list **replaces** the shipped list | koanf list semantics, and the only sane merge for an ordered list. Document "copy the defaults and edit"; `--dump-config` shows the effective result |
| Every selector value must resolve to something real | A typo'd action or resource name fails startup rather than silently never matching |

### 4.4 Resource resolution

> **Superseded 2026-08-19 — most of this is now mechanism P0 already built.** Three changes:
>
> 1. **`Resource` no longer carries `Type`.** It is `{ID, Name, Metadata}`, and `resource.type` is stamped from `Operation.ResourceType` in `buildEvent` at emit time — on both surfaces, for success *and* denial, whether or not a handler ever called `SetResource`. The table's "`resource.type` … free" row is now literally true rather than aspirational.
> 2. **`Operation.RESTResourceParam` closes the REST half without handler edits.** The middleware seeds `&Resource{Name: r.PathValue(op.RESTResourceParam)}` before calling `next`, mirroring the MCP adapter's pre-call seed. So for the **72 path-carrying routes** `resource.name` is a declared field on the operation, not a `SetResource` call. **B6 shrinks accordingly** — see A.2 item 11.
> 3. **The naming convention is settled: singular.** `8c8b33b7` converted all 12 defs to `"project"` / `"dataplane"` / `"environment"` / `"secret"`, and `Operation.ResourceType`'s doc comment now states the rule — singular, **not** the plural REST path segment — because `Selector.Resources` and a published event's `resource.type` read the same field, so a mismatch makes selectors silently fail to match real events. **The generator (B1) must singularise the path segment**, which is the one non-mechanical step in an otherwise mechanical derivation: `dataplanes`→`dataplane` is trivial, but `secrets`→`secret` vs `authzrolebindings`→`authzrolebinding` vs irregulars need an explicit map rather than a trailing-`s` strip. Note also that `SetResource` calls no longer pass a type at all — `&audit.Resource{ID: …, Name: …}` — so the 38 call sites B6 touches cannot reintroduce a divergent value even by accident.

The context mechanism works identically on both surfaces. The MCP middleware controls the ctx it passes to `next`, and that ctx reaches the handler untouched:

```
handleReceive → mh(ctx, …)          $SDK/mcp/shared.go:169   (middleware chain)
  → Server.callTool(ctx, req)       $SDK/mcp/server.go:732
    → st.handler(ctx, req)          $SDK/mcp/server.go:743
      → ToolHandlerFor th(ctx, req) $SDK/mcp/server.go:315
        → h(ctx, req, in)           $SDK/mcp/server.go:337
          → mcphandlers.X(ctx, …)
```

So `audit.SetResource(ctx, …)` from an mcphandler is visible to the MCP middleware, using **the same helper and pattern the REST handlers already use** at 12 call sites.

**Result-parsing does not work and must not be relied on.** `mutationResult` (`mcphandlers/transform.go:92-104`) returns only `name`, `action` and `namespace` — no UID. `CallToolResult.StructuredContent` cannot supply `resource.id`.

| Field | Source | Derivable without handler cooperation? |
|---|---|---|
| `resource.type` | The operation | **Yes** — both surfaces, free |
| `resource.name` | REST update/delete: `r.PathValue`. REST create: request body (**not until P3**). MCP: a declared argument | **72 of 114** REST routes; **all 69** MCP tools |
| `resource.id` / `uid` | Only exists after the service returns the object | **No** — handler-supplied via `SetResource` |

**Degradation is safe.** An operation whose handler does not call `SetResource` still emits action and resource type; it lacks only name and id. The coverage matrix records which operations supply full identity, so the gap is visible rather than silent.

### 4.5 The validation gate — requirement 4

`TestAuditCoverage`, six assertions in both directions:

1. Every state-modifying spec operation has a definition **or** an exemption with a reason.
2. Every registered MCP tool has a binding **or** an exemption with a reason.
3. Every MCP binding points at an operationId that exists — catches a spec rename.
4. Every exemption references something that exists — catches an orphan after a removal.
5. **Every operation resolves to a category**, by explicit resource-kind membership in the generator's mapping. A wildcard never satisfies this. *(An earlier revision added "and no `OperationDef` declares `access`" — dropped, since §4.3.1's decision means every `Category` value is legitimately declarable.)*
6. No duplicate bindings.

> **Revised 2026-08-19 — what is already enforced, and what changed.**
>
> **Assertion 3 is partly free at startup.** `BuildPatternMap` errors when an operationId has no route in the live spec, and when two operations collide on one pattern — so a spec rename already fails the server's construction, not just a test. Keep the test anyway: it fails in CI without booting a server, and it covers MCP bindings, which `BuildPatternMap` never sees.
>
> **Assertion 6 is the one to write first, and it currently cannot fail.** Per B2, `MCPBindings` overwrites on a duplicate `MCPToolName` with no error, and the existing `definitions_test.go` checks duplicate operation *IDs* only. This is the "test that can never fail" trap D8 warns about, one level down: the assertion exists in this plan, the data cannot yet violate it, and the code silently tolerates it. Land it with B2's shape change.
>
> **Assertion 5's mapping is now against two operation categories, not three** — see §4.3.1.

> **Two implementation facts the gate depends on, neither of them obvious (2026-08-24).**
>
> **The generator reads the *embedded* spec, not `openapi/openchoreo-api.yaml`.**
> `tools/auditgen` takes its input from `gen.GetSwagger()`, because oapi-codegen normalises
> operationIds to PascalCase (`handleAutoBuild` → `HandleAutoBuild`) and PascalCase is what
> `BuildPatternMap` cross-references at request time. Reading the YAML directly would generate
> IDs that **never match a route** — audit would silently stop. Hence
> `audit-gen: openapi-codegen` in `make/golang.mk`: the ordering is load-bearing, not
> incidental, and anyone "simplifying" it to read the file breaks audit without failing a test.
>
> **`resourceCategories` is exhaustive in both directions.** `checkNoOrphanCategories` errors
> when the map names a resource kind no live operation uses, and `deriveDefinition` errors on a
> kind with no entry — *"add one … rather than defaulting silently."* So the mapping cannot rot
> in either direction, which is what makes gate assertion 5 meaningful rather than decorative.
> Related: `singularOverrides` is **empty today**, kept as an explicit hook rather than pulling
> in inflector logic — `singularize` strips a trailing `s` and errors if there is none.

**Enumerating MCP tools needs no heuristic.** `toolsets.Register(server)` returns the `perms` map, whose keys are all 157 tool names.

**The read-only tools need a rule, not an exemption each.** A tool is read-only iff every action in `ToolPermission.Actions()` (`pkg/mcp/tools/types.go:565`) has verb `view` — actions are `resource:verb` strings (`internal/authz/core/actions.go:38-44`). That reduces explicit exemptions to the `get_*_creation_schema` tools, which declare `create` verbs but make no service call. A newly added write tool carries a create/update/delete verb, so the rule does not cover it and the gate fails until it is bound or exempted.

> **Counts corrected 2026-08-24 (see §0.6):** **80** read-only, not 88, and **8** explicit
> exemptions, not 5 — the 5 `get_*_creation_schema` tools plus 3 deprecated
> `get_cluster_*_creation_schema` aliases. The rule itself works exactly as described, and gate
> assertion 4 additionally rejects an exemption naming a tool the verb rule already covers, so a
> redundant exemption cannot accumulate.

---

## 5. Deliverables

> **Everything P1 builds sits on P0's core.** The `Operation`/`Envelope`/`Emitter`/`PolicySet` split, the configuration model, both adapters, the exact-map resolver and resource resolution are P0 deliverables 0.7–0.13. P1 adds no audit machinery.

**B1. Generator and generated definitions.** Spec → `definitions.gen.go`, registered in `make code.gen`, stamping operationId, action, resource kind and category. Fails on any operation whose resource kind has no category mapping, and on any orphan mapping entry. Joins the do-not-hand-edit set.

**B2. The full MCP binding table.** All 69 entries with the declared resource-name argument per binding, **including both halves of each of the 21 scope-collapsed pairs** — e.g. `create_component_type` binds to `CreateComponentType` at `scope: namespace` and `CreateClusterComponentType` at `scope: cluster`.

> **The shape change this deliverable used to require has moved into P0** — deliverables 0.10a–0.10e, added 2026-08-19. There, `OperationDef` gains `MCPScope`, bindings key on `(toolName, scope)`, the adapter resolves scope before lookup (absent → `namespace`, matching `resolveScope`), and a duplicate key becomes a loud failure instead of a silent overwrite.
>
> **Confirm those landed before starting B2.** Against the old tool-name-only key, adding a fan-out pair silently drops one half and mis-attributes the other — a namespaced create recorded as a platform-wide cluster one.
>
> What remains here is bulk declaration: 69 rows, each naming its operation, its resource-name argument, and for the 21 pairs its scope. Per §2.5 none of it is derivable, which is why this stays a hand-written table rather than being generated like B1.

**B3. Exemptions with mandatory reasons.** `evaluates`, `generateRelease`, `handleAutoBuild`, and the 5 `get_*_creation_schema` tools. Read-only tools covered by the verb rule.

> **As shipped (2026-08-24): 2 REST + 8 MCP, and `generateRelease` is *not* exempt.**
> `internal/openchoreo-api/audit/exemptions.go` exempts only `Evaluates` and `HandleAutoBuild`
> on REST. **`GenerateRelease` is audited**, because it turns out to persist a new
> `ComponentRelease` via `k8sClient.Create` — it is a genuine mutation, not the
> render-without-persisting read this plan assumed (§2.2's *"2 routes are reads expressed as
> POST"* is therefore one route, not two). It gets a real definition through the generator's
> `generateReleaseOverride`.
>
> On MCP the count is **8**, not 5: the 5 `get_*_creation_schema` tools plus 3 deprecated
> `get_cluster_*_creation_schema` aliases, each with its own reason string.
>
> Note the two-place bookkeeping this creates: `tools/auditgen`'s `excludedOperationIDs` skips
> the operation at *generation* time, and `exemptions.go`'s `RESTExemptions` carries the
> *reason* the gate reads. Same two IDs, different jobs — keep them in step.

**B4. The validation gate.** Six assertions per §4.5.

**B5. Auth-failure coverage (PR-7).** A **second, outer** HTTP adapter instance from the same constructor, emitting **only** when the status is 401 *and* no `SubjectContext` is present → `result: unauthenticated`, nil operation. Mutually exclusive with the inner instance; the wiring test asserts exactly one event in both the success and rejection cases.

> **Unblocked 2026-08-19** by §4.3.1's decision — the distinction moved to `result`, not a third category. Two consequences for this deliverable:
>
> - **The category question disappears.** The outer instance emits with a nil `Operation`, and `buildEvent` already sets `Result` outside the nil-op check, so the adapter sets `Result` and nothing else. No synthetic operation, no `Envelope` category override.
> - **Its trigger narrows to 401 alone.** A 403 means auth succeeded and the PDP refused a known subject — that is a *resolved* operation the inner instance already audits with `result: denied` (EM-4b). The outer instance must not fire on it, or the request is double-audited. The original "401/403 and no `SubjectContext`" condition was belt-and-braces against a category it no longer needs to assign.
>
> **`ResultUnauthenticated` is task 1.0b** (the enum plus both mappers — see §4.3.1), so land that first. B5 is then only the outer instance.
>
> Still open here: what `action` a rejection carries. It has no operation, so `action` is `""` today. Either accept that, or have the outer adapter stamp something readable — but if so, note that an `actions: [...]` selector could then match a value no operation declares, which is the same category-of-problem the `access` design had.
>
> **Resolved by omission, 2026-08-24 — the empty string shipped.** `buildEvent` fills
> Action/Category/OperationID/ResourceType only inside `if op != nil`, while `LogEvent` writes
> `action` and `category` **unconditionally** (`logger.go:77-78`) and omits `origin` /
> `operation_id` when empty. So a rejection publishes `"action":""` and `"category":""` — an
> asymmetry within one record. Nothing decided this; it fell out of the code.
>
> **The consequence nobody wrote down is which policy selectors can reach a rejection.** All
> four operation-derived selectors short-circuit on `rc.Operation == nil`
> (`policy.go:138-149`), and `entitlementsMatch` returns false on the nil map an anonymous
> actor carries. Traced through `Selector.matches`:
>
> | Selector | Reaches a rejection? |
> |---|---|
> | `origins` | ✅ `api` |
> | `actor_types` | ✅ `anonymous` — and `KnownActorTypes()` permits that value |
> | `results` | ✅ `unauthenticated` |
> | `actors` | ⚠️ only via the literal string `"anonymous"`. `Actors` matches `Actor.ID` and is free-form, not enum-validated — correctly so, since it holds arbitrary subject IDs |
> | `entitlements` | ❌ nil map never matches |
> | `categories`, `resources`, `operations`, `actions` | ❌ all short-circuit on the nil operation |
>
> So an operator wanting to damp a flood of expired-token retries has essentially **one usable
> rule shape**: `match: {results: [unauthenticated]}`. That is worth documenting for operators,
> and it is what **P7's read path and P8's filter UI need to know** — a downstream index will
> also see `category: ""` as a value it has to map.

**B6. `resource.name` for REST creates — now ~38 sites, not ~72.** The path-carrying routes are already covered declaratively by `Operation.RESTResourceParam` (§4.4), so `SetResource` calls are needed only where the name lives in the **request body**: the 42 POST creates, of which P0 did 4. **Place the call after the `err != nil` check**, never immediately after the service call — the PR #1465 defect. Pass `{ID, Name}` only; do not populate `Resource.Metadata` (§3).

> **As shipped: 78 sites, not ~38 — and the scope grew for a good reason (2026-08-24).**
> Body-carried creates were only half of it. The other half was `resource.id`: the UID is only
> knowable *after* a call succeeds, so **update and delete handlers need a `SetResource` call
> too**, even though `RESTResourceParam` already gives them the name. Before this, a handful of
> pre-P1 handlers did that and every P1-added one did not, so most update events shipped with
> no `resource.id` at all. Hence 78 calls across 35 files rather than 42.
>
> Three consequences:
>
> - The signature shipped as `{Namespace, ID, Name}`, not `{ID, Name}` — see Appendix E.1.
>   `Resource.Metadata` stayed empty as required (§3, verified).
> - `SetResource` **replaces rather than merges**, so a handler passing only `{ID, Name}` wipes
>   the namespace the middleware seeded pre-handler. That is the bug `8d257ca9` fixed across
>   ~30 handler files, and it is documented at `audit_wiring_test.go:180-188`. Any new call site
>   must pass the namespace explicitly.
> - The review rule's *presence* half is now machine-enforced by
>   `audit_setresource_ast_test.go` — Appendix E.3. Its *ordering* half still needs human
>   review; an AST test cannot see whether the call precedes the `err != nil` check.

**B7. `/exec/` and wirelogs — the stated blocker is gone.** `8c8b33b7` added `responseWriter.Unwrap()`, so `http.ResponseController` now reaches `Hijacker` and `Flusher` through the audit wrapper. The original justification for keeping these routes outside the OpenAPI chain no longer holds, which makes "wrap them with the standard adapter" the cheapest option rather than an impossible one — prefer it over a hand-rolled direct `Emit`. **Verify empirically before committing to it**: exec is a hijacked connection and wirelogs is a streaming response, so confirm both still work end to end through the wrapper (a `k3d` exec session and a live wirelogs tail) rather than trusting `Unwrap` on inspection. Fall back to a direct `Emit` if either breaks. `component:exec` is the highest-sensitivity action in the API; the recommendation remains to audit it rather than exempt it.

> **Shipped through the standard adapter — but the empirical check this deliverable demanded
> was not done (2026-08-24).** Both routes are audited via
> `audit.NewMiddlewareForRoutes(logger, patternMap, emitter, enabled)`, with the two operations
> hand-declared in `internal/openchoreo-api/audit/exec_wirelogs.go` carrying
> `NotInOpenAPISpec: true` so `BuildPatternMap` skips them rather than failing — while they
> still flow through `GetOperations()` for selector validation, the gate and the matrix. Their
> pattern map is owned by `handlers.NewExecWirelogsAuditMiddleware`, keyed by the *registered*
> route pattern, since that is not derivable from a spec that does not contain them. Note both
> are audited **despite being GETs**: exec is a live shell into a pod and wirelogs streams live
> traffic, so neither is the ordinary resource read the "reads are out of scope" rule is about.
>
> What exists is `exec_wirelogs_audit_test.go` — `TestFindFlusher_SeesThroughAuditWrapper`,
> `TestExecWirelogsAuditMiddleware_ResolvesBothRoutes`,
> `TestExecHandler_SetsResourceNameFromParsedPath`. All unit-level. This deliverable said
> *"verify empirically … rather than trusting `Unwrap` on inspection"*, and a `Flusher` unit
> test is not a hijacked connection. **Filed as task 1.14**, and it is the one open item with a
> real chance of finding a defect: exec is the most sensitive route in the API and the one whose
> transport the wrapper is most likely to break.

**B8. Harden `X-Request-ID`.** Accept the inbound header only if it parses as a UUID, else generate. Count rejections. Partial EM-16; the `traceparent` derivation is P2.

**B9. Coverage matrix (DO-2).** Generated from **effective** config. Lists both surfaces, every exemption with its reason, which operations supply full resource identity, and the known non-events (malformed path params, direct-MCP `source_ip`).

> **Shipped, and one design point was reversed deliberately (2026-08-24).** The matrix
> (`tools/auditcoverage` → `docs/audit/coverage-matrix.md`) is generated from **what the code
> covers, not from effective config** — its own header says so: *"not one deployment's
> `audit.policies` configuration, which can additionally suppress publishing for a matched
> event without changing what's covered here."* That is the better contract: a matrix keyed to
> one deployment's config could not be committed to the repo or reviewed in a PR.
>
> It is also **stronger than DO-2 asked for**. Because `audit-coverage-matrix` is deliberately
> *not* part of `code.gen`, `code.gen-check` cannot catch a stale matrix — so
> `tools/auditcoverage/main_test.go` closes that itself with `TestCoverageMatrix_IsFresh`, which
> re-renders and byte-compares the committed file. Its sibling `TestRender_Deterministic`
> renders twice in-process and diffs, guarding a real map-iteration-order bug. The result is a
> generated document that **cannot silently go stale** — see Appendix E.12.

---

## 6. Task breakdown

Dependencies on P0 deliverables are written `P0/0.n`.

Two tasks were added on 2026-08-19 and both are **prerequisites**, not additions at the end: 1.0 unblocks B2's bulk work, 1.0b unblocks B5.

| # | Task | Reqs | Dep |
|---|---|---|---|
| ~~1.0~~ | ~~Make `MCPBinding` express a scope-collapsed fan-out~~ — **moved into P0 as 0.10a–0.10e** (2026-08-19). A core-shape defect, and P0 owns the core; P1 now starts against a binding shape that can hold the fan-outs | PR-4, PR-3 | **P0/0.10b–e** |
| **1.0b** | **Add `ResultUnauthenticated`** — `types.go` constant, `validResults` entry so policies may select it, split `determineResult` (401 → unauthenticated, 403 → denied) and `classifyResult` (`ErrNoSubject` → unauthenticated, `ErrForbidden` → denied, `ErrPDPFailure` unchanged). *Revised 2026-08-19: this row previously read "add `CategoryAccess`" — there is no third category, the distinction is on `result` (§4.3.1)* | EM-3, EM-4b, PR-6, PR-7 | — |
| 1.1 | Confirm the authentication topology per surface (§4.9.3) | PR-7 | — |
| 1.2 | Spec→definitions generator, incl. category stamping from resource kind | PR-1, EM-4a, EM-4c | P0/0.9 |
| 1.3 | Full 69-entry MCP binding table, incl. the 21 scope-collapsed pairs | PR-4 | **P0/0.10b–e**, P0/0.12, 1.2 |
| 1.4 | Exemptions with mandatory reasons | PR-3 | 1.2, 1.3 |
| 1.5 | The validation gate, six assertions, both surfaces | PR-3 | 1.4 |
| 1.6 | PR-7: second outer HTTP instance, firing on 401-with-no-`SubjectContext` only → `result: unauthenticated`, nil operation | PR-7 | **1.0b**, 1.1, P0/0.9 |
| 1.7 | `SetResource` on the ~38 remaining body-carried REST creates (path-carrying routes are covered by `RESTResourceParam`) | PR-1 | P0/0.13 |
| 1.8 | `/exec/` + wirelogs — try the standard adapter now `Unwrap()` exists; verify a live exec session and wirelogs tail before committing | PR-1 | P0/0.9 |
| 1.9 | Harden `X-Request-ID` | EM-16 | — |
| 1.10 | **Decide** how `agent` is determined for `actor.type` | EM-6 | — |
| 1.11 | Coverage matrix from effective config, both surfaces | DO-2 | 1.5 |
| 1.12 | Confirm PR-2 and PR-6 deliver on both surfaces | PR-2, PR-6 | 1.3, 1.5 |
| **1.13** | **Demonstrate each of the six gate failures by deliberate breakage, and record the output** — exit criterion 3 requires this explicitly (*"not merely asserted in code"*). Note that assertions 3 and 6 cannot be broken from the data side; for those, demonstrate the upstream construction error instead, and record that as the reason (§0.5) | PR-3 | 1.5 |
| **1.14** | **Live-verify `/exec/` and wirelogs through the audit wrapper** — a real `k3d` exec session and a live wirelogs tail, which is what B7 asked for and the unit tests do not cover | PR-1 | 1.8 |

**Tasks 1.13 and 1.14 were added on 2026-08-24**, when the rest of P1 was found to be shipped.
They are verification, not implementation: nothing downstream depends on them, so neither
blocks P2 — but P1 should read as *shipped with two verification gaps* rather than closed,
because both are conditions its own exit criteria state.

**This caveat has not propagated to the commit that ships P1** — its message says `Closes
#4460` with no mention of 1.13/1.14. A commit message can't be edited after the fact without
rewriting history, so the record of "not fully verified yet" lives here instead: don't read
`Closes #4460` as evidence exit criteria 3 and 8 are met, only this section is.

**Task 1.10 needs a decision, not just implementation.** `actor.type` comes from `subjectCtx.Type`, driven by the `security.subjects` config — so `agent` is a subject type to configure, not a constant to add. And it **cannot be derived from the surface**: the scope doc's EM-11 states that a human can drive MCP just as an agent can drive REST. Either an IdP claim distinguishes agents, or `origin: mcp` carries the signal and `agent` waits.

---

## 7. Demo

```sh
# 1. The gate blocks an unaudited operation on EITHER surface
make test GO_RUN_ARGS="-run TestAuditCoverage ./internal/openchoreo-api/audit/..."
#    add a state-modifying route whose resource kind has no category mapping
#    → FAIL: resource kind "foobars" has no category
#    register a new MCP write tool without a binding
#    → FAIL: tool "create_foo_bar" is neither bound nor exempted

# 2. A renamed operationId breaks the build rather than drifting
#    → FAIL: mcp binding "create_component_type" targets unknown operation

# 3. Rejections
curl -X POST .../api/v1/... -H "Authorization: Bearer <expired>"   # → result=unauthenticated
#    and a PDP refusal on a real operation stays distinguishable:
#    → result=denied, category=management (EM-4b), exactly one event

# 4. The coverage matrix
make audit.coverage-matrix && cat docs/audit/coverage-matrix.md

# Pre-PR gate
make code.gen && make lint-fix && make test && make code.gen-check
```

---

## 8. Exit criteria

> **See §0.3 for the verdict against each of these.** Six of eight are met; criteria 3 and 5
> are qualified, and criteria 3 and 8 are what tasks 1.13 and 1.14 exist to close.

1. Every state-modifying REST operation is audited or exempted with a reason.
2. Every registered MCP tool is bound or exempted; the ~~88~~ **80** read-only ones by the verb rule.
3. The gate fails on: an unmapped route, an unmapped MCP write tool, a renamed operationId, an orphan mapping entry, a resource kind with no category, **and two defs claiming one MCP tool name** — each demonstrated by a deliberate breaking change, not merely asserted in code.
4. Token rejections emit `result: unauthenticated` with a nil operation; a PDP refusal on a resolved operation emits `result: denied` with that operation's real category; and no request is double-audited by the two HTTP instances. The same two outcomes are distinguishable on MCP (`ErrNoSubject` vs `ErrForbidden`).
5. `resource.name` is present on all 114 REST operations — path-derived via `RESTResourceParam` for the 72 that carry it in the path, `SetResource` for the 42 body-carried creates.
6. The only `Category` values anywhere — table, config, emitted events — are `management` and `authorization`.
7. The coverage matrix is generated and lists both surfaces with every exemption and its reason.
8. `/exec/` is either audited or its exclusion recorded in the matrix.

**Repos.** openchoreo.

---

## Appendix A — Deviations from the phased plan's P1

| # | Phased plan said | Now | Why |
|---|---|---|---|
| **D1** | Task 1.5: hand-rewrite `definitions.go` against all 114 routes — *"this is the bulk of the phase"* | **Generated** from the spec | operationIds are 1:1 with routes and mechanical. The bulk moves to the MCP binding table |
| **D2** | `Category` is a field on `ActionDefinition`, hardcoded | **Stamped from resource kind by the generator, and deliberately NOT operator-configurable** | See Appendix B item 2. An earlier draft made it config; that fragments retention |
| **D3** | Task 1.7: **service-layer** emission for MCP, since *"route-pattern middleware cannot capture MCP calls at all"* | **Protocol-level receiving middleware** | The SDK hook sees tool name, raw arguments, per-request headers **and** denials. No service-layer plumbing, and double-counting never arises. It does **not** avoid handler cooperation for resource identity |
| **D4** | *"not a second set of action definitions"* | Half true: no second definitions, but a **declared 69-entry binding table** is unavoidable | §2.5 |
| **D5** | Gate: 4 assertions, REST only | **6 assertions, both surfaces.** Two of the original four are dissolved by D6 | Requirement 4 covers MCP |
| **D6** | *(absent)* | **Replace the regex resolver** with exact-map on `r.Pattern` | Deletes the silent-`regexp.Compile` and shadowing failure modes rather than guarding them |
| **D7** | *(absent)* | **Sentinel errors in `filter.go`** | Without them `result: denied` is unachievable on MCP |
| **D8** | Gate blocks on *"route with no definition"* | Blocks on **"resource kind with no category"** | Generation makes the former impossible. An implementer told to build it writes a test that can never fail |
| **D9** | Per-category settings map (OP-1, OP-3) | **Ordered `policies` rules with event-field selectors**, first match wins | A fixed grouping never fit. `category` was being asked to serve as both a stamped label and a policy handle; separating them resolves it |
| **D10** | The core framework and MCP support are P1's | **Moved into P0** (that plan's §5b, deliverables 0.7–0.13) | Retires all architectural risk on the 12-operation surface P0 already covers, so P1 and everything after are improvements on a settled foundation rather than reshapings of it. P1 becomes coverage and enforcement only, and adds no audit machinery |

**Carried over unchanged:** 1.1 (auth topology) · category taxonomy and EM-4b · PR-7 two instances · `/exec/` · `X-Request-ID` · coverage matrix · PR-6.

**Moved into P1 from P5a:** OP-1 and OP-3, now expressed as `policies` rather than per-category switches. **P5a shrinks** — task 5.8 becomes done and 5.9 becomes "extend an existing structure".

**Effects on later phases:**

| Phase | Effect |
|---|---|
| **P2** | `origin` becomes free — each adapter knows its surface. Tenancy needs no REST handler edits (`r.PathValue` works in the middleware position) but now has **two surfaces to populate**, MCP tenancy coming from raw arguments. Tenancy also unlocks the `namespaces` policy selector. **Corrected 2026-08-24:** it *does* need handler edits — `SetResource` replaces rather than merges — and one dimension already shipped. See E.1 |
| **P3** | Fills information block 3. Rewritten around configurable claims and attributes; body extraction and the strict middleware land here |
| **P5** | Fills sinks block 4. Lighter than planned, since `policies` already exists |
| **P10** | Additive: implements the reserved `pre_action`, `delivery` and `retries` keys. No config migration |
| **P7** | The read path must expose `resource.type` and `action` filters as first-class, not category alone — see Appendix B item 2 |

---

## Appendix A.2 — As shipped in P0 (added 2026-08-19)

Appendix A above records how P1 deviates from the *phased plan*. This section records how the **shipped P0 code** deviates from **this document**, which was written against `6bd147a1` before `96fe08ca` and `8c8b33b7` landed.

### Moved out of P1 — already shipped, do not rebuild

| # | Item | Where | Effect on P1 |
|---|---|---|---|
| 1 | **Exact-map route resolution (D6)** | `pattern_map.go`; `resolver.go` deleted | Done. `BuildPatternMap` errors on an operationId with no route and on two operations colliding on one pattern |
| 2 | **MCP denial sentinels (D7)** | `pkg/mcp/tools/filter.go:32-37` | Done, and better: `ErrPDPFailure` is mapped to `failure`, not `denied`, so a PDP outage is never recorded as a policy decision. §2.6's "denied is indistinguishable on MCP" is false |
| 3 | **Selector values must resolve to something real** | `config/audit.go` `collectSelectorValues` + `MustBeOneOf` on 7 axes | §4.3.3's last rule is done. A typo'd action, resource, operationId, category, origin, result or actor type fails startup |
| 4 | **Three more §4.3.3 rules** | `parsePartialSettings`, `validatePolicy` | `category`/`categories` rejected in `set:`; `publish: false` may not use `actors`/`entitlements`; unknown keys rejected outright (which is also what forbids arbitrary JWT claims) |
| 5 | **`actor_types` validated against real subject types** | `SecurityConfig.KnownActorTypes()` | Confirms task 1.10's premise: `agent` is a **configured subject type**, not a constant to add. The decision is narrower than the plan assumed |
| 6 | **`origin` and `operation_id` are real `Event` fields** | `types.go` | Removed from §3's non-goals. Appendix C's "P2 promotes `origin` to a schema field" is stale — P2 inherits it |
| 7 | **Panic-safety on both adapters** | `middleware.go`, `mcpaudit` | Not in any plan. A panicking handler emits a `failure` event, then re-panics. Preserve this in every new composition point B5 and B7 add |
| 8 | **`Sink` interface + multi-sink fan-out** | `emitter.go` | P5's TR-4 seam exists in code (the *config* block is still reserved). P5 shrinks further than Appendix A predicted |

### Shipped differently — P1 must adapt

| # | Item | Detail | Consequence |
|---|---|---|---|
| 9 | **`MCPBinding` cannot express a scope-collapsed fan-out, and drops duplicates silently** | One `MCPToolName` per def; `bindings[d.MCPToolName] = …` is last-write-wins; the adapter looks up on tool name alone; `definitions_test.go` checks duplicate operation IDs only | **Moved into P0 as 0.10a–0.10e**, 2026-08-19 — a core-shape defect, and P0 owns the core. The 21 fan-outs are exactly two operations under one tool name, so B2 no longer needs a shape change: it adds rows to a key that can hold them |
| 10 | **`Resource` lost `Type`, gained `Metadata`** | `Resource{ID, Name, Metadata}`; `resource.type` stamped from `Operation.ResourceType` in `buildEvent` | `resource.type` is now genuinely free on both surfaces for success *and* denial. `Resource.Metadata` is an ungoverned `map[string]any` — **P1 must not populate it** (§3) |
| 11 | **`Operation.RESTResourceParam` + middleware path seeding** | Middleware seeds `&Resource{Name: r.PathValue(op.RESTResourceParam)}` before `next` | **B6 shrinks from ~72 to ~38 sites.** Path-carried names are declarative; only body-carried creates need handler calls |
| 12 | **`responseWriter.Unwrap()`** | `http.ResponseController` reaches `Hijacker`/`Flusher` through the wrapper | **B7's stated blocker is gone.** Wrapping `/exec/` and wirelogs with the standard adapter is now the cheapest option — verify empirically |
| 13 | **An empty `r.Pattern` no longer panics** | Logs an error and passes through, because `BaseRouter` is an interface a caller could satisfy with a non-`ServeMux` router | Construction time (`BuildPatternMap`) is the loud failure point, not request time. Shifts where the gate's value sits — see §4.5 |
| 14 | **Reserved config keys were deliberately not modelled** | Only `enabled`, `defaults.publish`, `policies[].{match,set.publish}` exist; `parsePartialSettings` rejects unknown keys | §4.3's YAML **would fail validation today**. It is a P1→P10 target shape, not writable config |
| 15 | **`ResourceType` convention settled: singular** | All 12 defs converted; the doc comment states the rule; `SetResource` no longer takes a type | B1's generator must **singularise** the path segment via an explicit map, not a trailing-`s` strip |

### Decided 2026-08-19

| Decision | Outcome |
|---|---|
| Where do PR-7 rejections put their category, given P0's two-value set? | **Nowhere — the question was mis-framed.** `Category` groups *operation types*; a rejection is an *outcome*. So `Result` gains **`unauthenticated`** and `Category` stays `management`/`authorization`, permanently and without qualification. An intermediate revision added `access` as a third category; that was **reversed the same day** because on MCP `ErrNoSubject` fires on a *bound* tool, so the same logical event would have carried a real category there and `access` on REST — breaking §4.3.2's own surface-parity rule. The enum and both result mappers are **P1 task 1.0b** — briefly filed as P0 since they fix a live 401/403 conflation on already-audited operations, then moved back because P0 is closed and PR-7 is the main consumer. B5 (task 1.6) depends on 1.0b. See §4.3.1 |

### Still open, unchanged by P0

PR-7's two-instance design (audit sits inside auth per `handler.go`, so one instance still cannot see rejections) · B1's generator · the gate · the coverage matrix · `X-Request-ID` hardening — and now **`source_ip`'s trusted-proxy assumption**, which `8c8b33b7` documented but did not fix.

---

## Appendix B — Scope-doc amendments

1. **PR-4 and EM-11 corrected.** MCP is a **sibling entry point onto the shared service layer**, not a wrapper over the HTTP API. So *"audited once, at the API layer"* does not hold. The fix is a protocol middleware plus a declared binding table, not the service-layer emission the phased plan's §2.4 proposed.
2. **EM-4a refined: `category` is a stamped label, not a policy key.** It is a closed **two**-value set — `management`/`authorization` *(revised 2026-08-19; was "three-value" when the set still included `access`)* — assigned per operation, and **not** operator-configurable. It groups **operation types**, not outcomes: a rejected request is not a kind of operation, which is why the authentication-vs-authorization distinction lives on `result` instead. See item 7. Operator flexibility moves to `policies`. Reason: category is immutable in published records and is what backends key retention on, so reclassifying splits history — events already indexed keep the old retention, silently. DO-4 and DO-6 should therefore steer durable retention and SIEM rules onto `resource.type` and `action`, which are intrinsic and stable, using `category` only for coarse grouping.
3. **A new §3 exclusion:** `source_ip` is unobtainable for direct (non-proxied) MCP clients.
4. **EM-6** needs a determination rule for `agent`, not just a constant — task 1.10.
5. **A new §3 exclusion:** a 400 from malformed path-parameter binding never reaches audit, because binding precedes all middleware.
6. **SE-2 strengthened in practice.** Suppression rules may not key on `actors` or `entitlements`, because a subject able to influence its own group membership could otherwise suppress its own auditing.
7. **EM-3 extended: `result` gains `unauthenticated`.** The scope doc specifies `result ∈ {pending, success, failure, denied}`. That set cannot distinguish *"we do not know who you are"* from *"we know who you are and you may not do this"* — the two most different denial outcomes in the trail, and the two an operator most needs to treat differently (a misconfigured client's expired-token retries are high-volume and low-value; an authz refusal on a real operation is the highest-value signal there is). So the set becomes `{pending, success, failure, denied, unauthenticated}`, with `denied` narrowed to mean an authorization refusal of an authenticated subject.
   This also **replaces EM-4a/EM-4b's `access` category**: the scope doc reserved a category for *"rejections at the boundary that never resolve to an action"*, but that is an outcome, not an operation type, and expressing it as a category breaks surface parity — on MCP the same rejection arrives on a *bound* tool with a legitimate category of its own. EM-4b's substantive rule survives intact and is in fact sharper: a denied attempt at a resolved operation keeps that operation's category, and is now distinguishable from an unauthenticated one by `result` rather than by the absence of a category.

---

## Appendix C — Hand-off to P2

> **Updated 2026-08-24 against the shipped code.** Four items are now facts rather than
> expectations, and two change P2's scope — full detail in Appendix E.

- **Coverage:** all 114 REST operations and 69 MCP tools audited or exempted, enforced by the gate. ✅ **True as stated** — and the 114 is 112 generated + `Exec`/`Wirelogs`.
- **The registry is the single place** an operation is described; P2 adds envelope fields to `Envelope` and the emitter, not to per-surface code. ✅ **Held.** Note the registry is now three files — `definitions.gen.go` (generated) enriched by `mcp_bindings.go` and extended by `exec_wirelogs.go` — but still one `GetOperations()`.
- **`origin` is already resolved** by construction in each adapter — P2 promotes it to a schema field. ✅ Already a real `Event` field (A.2 item 6); P2 adds only the `client` value.
- **Tenancy is cheap on REST** via `r.PathValue` in the middleware position, and available on MCP from raw arguments. Neither needs handler edits. It also unlocks the `namespaces` policy selector. ⇒ **Partly done, and not as described.** `resource.namespace` shipped on both surfaces — so **task 2.5 is a partial migration, not greenfield** — and it lands *inside* `resource`, not at the envelope top level. It **does** need handler cooperation after all: `SetResource` replaces rather than merges, so every one of the 78 call sites must pass the namespace explicitly. See E.1. ⇒ **Shrunk again, 2026-08-25.** P1 task 1.15 (§0.7) takes `project`, `component` and `resource` as well, leaving 2.5 with `environment` plus any decision to relocate the group to the envelope top level. 1.15 also shows the "cheap via `r.PathValue`" premise to be false below the namespace: REST paths are flat, so those three are captured at `AuthzChecker.Check`, not from the path — and it sidesteps the replace-not-merge trap by storing hierarchy *beside* `Resource` rather than inside it, which is the pattern 2.5 should follow for `environment`.
- **`sequence` (RE-7) belongs in P2**, per proposal #4396's core schema table — an in-memory counter beside `producer_instance`, not queue machinery. Unchanged.
- **`resource.uid` (EM-14)** builds on the `SetResource` call sites P1 extends. ⇒ **Largely shipped on REST (78 sites), barely on MCP (6).** Task 2.12 re-scopes to closing that surface-parity gap. See E.2.
- **Observer's MCP server** (`cmd/observer/main.go:311-316`) has unverified middleware setup — a dependency for PR-5. **Now verified:** observer has no audit code at all, and all 13 of its MCP tools are read-only, so there is nothing to bind. A separate plan covers extending audit there — `openchoreo-audit-logging-observer-plan.md`.
- **New for P2 — two render paths must stay in step.** `Event.MarshalJSON` and
  `Logger.LogEvent` independently produce the same wire shape, guarded by one test. `schema_version`
  (task 2.1) and every field P2 adds must land in both. See E.5.
- **New for P2 — `actor.entitlements` is an emitted field** nobody planned. Task 2.17's PII
  documentation must cover it, and Appendix D's stale-session caveat applies to it. See E.4.

---

## Appendix D — The MCP subject-resolution issue, handled separately

Filed and fixed outside this plan. Recorded here because P1's actor-selector restrictions depend on it.

**What it is.** For a stateful MCP session, every `tools/call` is authorized against the `SubjectContext` parsed from the token on the *session-creating* request. Three facts combine: `Server.Connect` receives that request's ctx (`$SDK/mcp/streamable.go:496-499`); jsonrpc2 wraps it in `notDone{ctx}`, preserving context values for the session's life (`$SDK/internal/jsonrpc2/conn.go:218-219`); and no session timeout is configured (`pkg/mcp/server.go:64` passes nil options).

**Impact.** Ordered by severity: **entitlement revocation is ineffective** against a live session, indefinitely, with no attacker involved — remove a user from a role and their open MCP session keeps the old permissions. Secondly, **a second party holding any valid token can drive a session opened by someone else** and be authorized as the opener, gated on knowing the `Mcp-Session-Id` (which is `rand.Text`, so not guessable).

**Why the SDK's own guard does not help.** The session-user-mismatch check at `$SDK/mcp/streamable.go:314-322` compares `sessInfo.userID`, populated only from the SDK's own `auth` package. OpenChoreo stores its subject under a private key of its own, so `Extra.TokenInfo` is always nil, `userID` is `""`, and the guard never fires.

**Not broken:** the HTTP-layer `jwtMiddleware` still validates a token on every POST, so this is not an unauthenticated bypass.

**Consequence for P1.** Until it is fixed, `actor.entitlements` on the MCP surface reflects initialize-time data. That is a second reason — alongside SE-2 — for the escalation-only restriction in §4.3.2. Once fixed, the restriction can be revisited on SE-2 grounds alone.

> **Sharper now that `actor.entitlements` is *emitted* (2026-08-24).** Appendix E.4 records that
> the entitlements map became a published event field, which no plan asked for. Combined with
> this issue, that means **the audit record itself can carry stale entitlements** for a
> long-lived MCP session — not just the authorization decision. An investigator reading the trail
> would see the membership as of session establishment, not as of the action. Worth a note
> wherever E.4's PII documentation lands (task 2.17).

---

## Appendix E — Shipped beyond plan (added 2026-08-24)

Appendix A records how P1 deviated from the *phased plan*; A.2 records how P0's shipped code
deviated from *this* document. **This appendix records what P1 shipped that no plan asked
for**, with the downstream phase that owns each item. Twelve items. Four of them change a later
phase's scope, and those are marked **⇒**.

### E.1 ⇒ `Resource.Namespace` — P2 tenancy, one dimension early · **owner: P2 task 2.5**

`Resource` gained `Namespace string json:"namespace,omitempty"`, populated on **both** surfaces
before the handler runs: REST from `r.PathValue("namespaceName")` (`middleware.go:147`), MCP
from the `namespace_name` call argument (`mcpaudit/audit_middleware.go:56`). Seeding pre-call is
what makes it survive a PDP denial, where no handler ever runs.

This is **task 2.5** delivered for one of its four dimensions — and it lands **inside
`resource`**, not at the envelope top level where 2.5 puts tenancy. So 2.5 is no longer
greenfield: it is a decision about whether `org`/`project`/`component`/`environment` join
`namespace` under `resource`, or whether `resource.namespace` migrates up. It also **unlocks the
`namespaces` policy selector earlier** than Appendix C predicts.

**One asymmetry to know about.** The REST seed hardcodes `r.PathValue("namespaceName")`, but
`BuildPatternMap` validates only `RESTResourceParam` against the pattern — it never checks for
`{namespaceName}`. A `RESTResourceParam` typo therefore fails loudly at construction, while an
operation not nested under `/namespaces/{namespaceName}/` silently yields an empty namespace.
This cannot simply be made an error, because cluster-scoped operations legitimately have no
namespace — but P2 will be adding three more tenancy fields the same way, and should not
rediscover it.

> **Superseded in part — 2026-08-25, by task 1.15 (§0.7).** Three of those dimensions are now
> claimed by **P1, not P2 task 2.5**: `project`, `component` and `resource`.
>
> - **The open question above is answered.** 1.15 chooses **flat siblings under `resource`** —
>   `project`/`component`/`resource` join `namespace` where it already lives. `resource.namespace`
>   does not migrate up.
> - **The asymmetry above is not removed, and is not extended either.** 1.15 adds no further
>   path-parameter reads, so `BuildPatternMap` gains nothing to validate. REST paths are flat —
>   only one path in the whole spec nests under `projects/{` — so the new dimensions cannot come
>   from `r.PathValue` at all. They are captured at `AuthzChecker.Check` instead. The paragraph
>   above anticipated "three more tenancy fields the same way"; that is precisely what 1.15 does
>   **not** do.

### E.2 ⇒ `resource.id` at 78 REST sites, 6 MCP sites · **owner: P2 task 2.12**

§3 listed `resource.uid` (EM-14) as a P2 non-goal. In fact `resource.id` is populated from the
K8s UID at **78 REST call sites across 35 handler files**, and at **6 sites in 2 files** on MCP
(`mcphandlers/projects.go`, `environments.go`). The other 67 bound tools get only the pre-call
seed — namespace and name, no UID.

So EM-14 is ~90% delivered on REST and ~3% on MCP. **That is a surface-parity gap in exactly
the kind of field this plan repeatedly insists must behave identically on both surfaces**, and
2.12 becomes "finish MCP", not "add it". Note the honest sub-case the code already documents:
delete handlers cannot supply a UID at all, because the delete service methods return only an
error — the name is the identifier that survives.

### E.3 An AST gate on `SetResource` presence · **owner: P2 2.12, P3**

`api/handlers/audit_setresource_ast_test.go` parses the handler package and asserts the call is
present — `TestHandlers_BodyCarriedCreatesCallSetResource` for operations with no
`RESTResourceParam` (no path parameter means no fallback, so an omission audits an empty
resource on every success and no runtime test would catch it) and
`TestHandlers_UpdatesCallSetResource` for the UID.

This converts P0 deliverable 0.6's **review rule into machinery**, which is what made 78 call
sites tractable. It guards presence, not position. P2's `resource.uid` work and P3's attribute
layer both inherit it.

### E.4 ⇒ `Actor.Entitlements` is now emitted · **owner: P2 task 2.17, P3 task 3.9**

`Actor` gained `Entitlements map[string][]string`, rendered as a nested group and populated only
when `EntitlementClaim != ""` (avoiding an `"entitlements":{"":null}` artefact). No plan asked
for it, and it has two consequences worth naming:

- It puts **group-membership data — PII-adjacent — into the audit record.** SE-6 (document
  PII-bearing fields, task 2.17) and SE-7 (configurable redaction, task 3.9) each gain a field
  they did not know existed.
- It makes the `entitlements` selector *inspectable in the record*, which is a genuine
  usability win: an operator can see why a rule matched.

### E.5 ⇒ `Event.MarshalJSON` — two render paths, one wire shape · **owner: P2 task 2.1, P5b**

`Event.ResourceType` is `json:"-"` and merged into the nested `resource` object only at render
time. `Event.MarshalJSON` exists **solely** so a sink calling `json.Marshal(event)` produces the
same wire shape as `Logger.LogEvent`'s slog groups — without it, a future sink would publish
`resource_type` as a *sibling* of `resource` rather than a field inside it.
`TestEvent_MarshalJSONMatchesLogEventShape` guards the agreement.

This is **P5b's sink seam, already load-bearing**: every new sink must go through
`MarshalJSON`, and P2's `schema_version` work has to keep both render paths in step. Neither
document mentions it today.

### E.6 `Loader.UnmarshalStrict` — a repo-wide config primitive · **owner: P5a task 5.9**

`internal/config/loader.go` gained `UnmarshalStrict` (`ErrorUnused: true`), so an unrecognised
key errors instead of being silently dropped — which is what makes a typo under `match:` a
startup failure rather than a rule that never fires. Audit is its first consumer, but it is
available to **every** config section now. Documented caveat: it does not carry koanf's default
decode hooks (duration parsing, `TextUnmarshaler`), because those are unexported — so a section
using it must not rely on either. P5a's OP-5 work will want it.

### E.7 `MCPAlias` / `MergeMCPAliases` — the missing 2-tools-to-1-operation mechanism

§2.5 counted *"2 tools → 1 operation (10 cases)"* as a fact but proposed no mechanism.
`OperationDef.MCPToolName` holds exactly one name, and duplicating the `OperationDef` row would
give `BuildPatternMap` a **false REST-pattern collision**. So `MCPAlias` attaches to the binding
table only and never enters `Operations()`. `MergeMCPAliases` errors on an unknown operation, a
collision with an existing binding, or a mixed scoped/unscoped tool name.

This is why the table is *69 tools, 90 binding keys*: 38 plain + 21 scope-collapsed × 2 + 10
aliases.

### E.8 The third scope hazard — mixed scoped/unscoped bindings

P0's 0.10a–e fixed *collisions* on `(tool, scope)`. It did not fix one tool name carrying
**both** an unscoped and a scoped binding: those are distinct map keys, so nothing collides —
but `resolveBinding` tries the unscoped key first, making the scoped binding **silently
unreachable**. `scopeTracker.checkAndRecord` now rejects the mix in either arrival order at both
entry points. Recorded in full in the P0 plan's 0.10e addendum, because it belongs with the
other two.

### E.9 Level-gating, PDP-error sanitisation, explicit logger

- `forceLevelHandler` wraps the application handler so `Enabled` always returns true —
  `logging.level: warn` can no longer silence audit — and it re-wraps on `WithAttrs`/`WithGroup`
  so a derived logger cannot lose the override. **Residual:** the handler is *reused*, so
  *format* still follows the application logger. Under a text handler, audit records ship
  unparseable. **P5a's TR-7 owns this.**
- **PDP errors are sanitised** before reaching the client, because the go-sdk relays error text
  as tool-call content; the detail is logged server-side instead. Not in any plan; a real
  hardening.
- **`NewLogger` and `NewToolFilterMiddleware` both require an explicit logger** — no
  `slog.Default()` fallback.

### E.10 ⇒ PR-6's hole on MCP: `?filterByAuthz=false` · **owner: P7 task 7.6**

With the authz filter skipped, a refusal arrives as a service-layer `services.ErrForbidden`
returned from the handler, and the go-sdk converts that into `CallToolResult.IsError` before the
middleware sees it — discarding the error identity. So it audits as `failure`, not `denied`.
The code documents this rather than fixing it, which is the right call.

**It qualifies exit criterion 4**, and it reaches the read path: a `results: [denied]` query
silently misses these events. P7's filter work should either surface the caveat or P3 should
close it.

> **Reclassified 2026-08-30 by PR #4571 — "unfixable at the middleware", not "unfixed".** The
> mechanism is now traced precisely: the go-sdk's **typed-tool adapter** calls
> `CallToolResult.SetError(err)` and returns a **nil** error to its caller. By the time the
> receiving middleware runs, `IsError` is all that survives — `mcpaudit.classifyResult` has
> nothing left to branch on, so no change *there* can ever recover the distinction. The two real
> options are (a) translate `services.ErrForbidden` in every MCP tool handler, or (b) return a
> structured `*jsonrpc.Error`, which changes client-visible protocol behavior. **Both are P7
> design decisions with blast radius outside audit**, which is why this stays open rather than
> being absorbed as a bug fix. The matrix's *Known non-events* entry was rewritten to say so.
>
> **The second half of this appendix item — "an HTTP-level 401 on `/mcp` emits nothing" — is
> closed** by the same PR. See §0.8.

Related but benign: `ErrPDPFailure` has **no explicit branch** in `classifyResult` — it falls
through to the generic `err != nil → failure`. The outcome is right (a PDP outage must never
read as a policy denial), but A.2 item 2 describes it as *deliberately mapped*, implying a case
statement a reader will go looking for and not find.

### E.11 The generator's embedded-spec dependency

`tools/auditgen` reads `gen.GetSwagger()`, not the YAML, because operationIds must be PascalCase
to match `r.Pattern` at request time. `audit-gen: openapi-codegen` encodes the ordering.
Reading the file directly would break audit **silently**. Full detail in §4.5's addendum.

### E.12 The coverage matrix cannot go stale

`audit-coverage-matrix` is deliberately outside `code.gen` ("reporting only"), so
`code.gen-check` cannot catch a stale matrix. `TestCoverageMatrix_IsFresh` re-renders and
byte-compares the committed file; `TestRender_Deterministic` renders twice and diffs, guarding a
real map-iteration-order bug. DO-2 is delivered better than task 1.11 specified — and the
matrix is keyed to **code coverage, not effective config**, deliberately, so that it can be
committed and reviewed. See B9's addendum.

