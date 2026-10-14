# Audit Logs Module — Phased Delivery Plan

**Companion to:** [`openchoreo-audit-logging-scope.md`](./openchoreo-audit-logging-scope.md) — the requirements register. Read it first.
**Epic:** [#3343](https://github.com/openchoreo/openchoreo/issues/3343) *(open)* · **Proposal:** [Discussion #4396](https://github.com/openchoreo/openchoreo/discussions/4396) — *this document phases that proposal* · **Earlier proposal:** [Discussion #1403](https://github.com/openchoreo/openchoreo/discussions/1403) · **Prior tracking:** [#1380](https://github.com/openchoreo/openchoreo/issues/1380) *(closed)* · **Prior implementation:** [PR #1465](https://github.com/openchoreo/openchoreo/pull/1465) *(merged 2026-01-16 — since unwired, see §2.1)*

**Status:** Draft · **Date:** 2026-08-04 · **Revised:** 2026-08-10, 2026-08-13, 2026-08-19, 2026-08-24 and **2026-08-30** (see the revision notes below)

> ### 🔄 Revision 2026-09-01 — P1.5 splits: the generated-server migration comes first
>
> **P1.5 is now two phases, in this order:**
>
> | | Phase | Document |
> |---|---|---|
> | **P1.5a** | Migrate `observer-api` onto its generated OpenAPI server | [`openchoreo-observer-generated-server-plan.md`](./openchoreo-observer-generated-server-plan.md) |
> | **P1.5b** | Extend audit logging to `observer-api`, **on top of P1.5a** | [`openchoreo-audit-logging-observer-plan.md`](./openchoreo-audit-logging-observer-plan.md) |
>
> **Why.** Auditing a hand-registered mux needs machinery that exists only to compensate for the
> absence of generated routing — an importable route-table package, and a bespoke gate asserting
> registered patterns agree with the spec, because `BuildPatternMap` cross-references operationIds
> against the spec but **never sees the mux**. Both were built (they sit staged in the working tree)
> and both are **vacuous on generated routing**, where the router and the audit pattern map derive
> from one source and cannot disagree. Building that machinery and then deleting it is waste.
> Migrating first means audit is built once, on the shape it will keep.
>
> **Consequence.** The staged observer-audit work is **set aside**, and P1.5b is rewritten against
> the post-migration architecture. Two staged pieces survive and are reclaimed rather than rebuilt —
> `tools/internal/auditgen` and `internal/auditconfig` — because both are refactors of
> *openchoreo-api's* code that know nothing about how observer registers routes.
>
> **P1.5a carries a decision that is not audit's to take.** The spec declares `BearerAuth` on the
> four internal-port alert-rule operations that are in fact served with **no authentication**, and
> `auth.OpenAPIAuth` keys off exactly that declaration — so the migration cannot be auth-neutral.
> The caller is the control plane's own `observabilityalertrule` controller, which sends no
> `Authorization` header and has no token source. That makes **task 7.22 a choice P1.5a must make**
> (enforce, and fix the controller) **or explicitly defer** (make the spec honest with `security: []`,
> and let 7.22 remove it later). See that plan's §1.
>
> The 2026-08-30 note below stands except where it says "P1.5" — read that as **P1.5b**.

---

> ### 🔄 Revision 2026-08-30 — a new phase, P1.5, is inserted before P2
>
> **P2 is no longer the live phase. `observer` is.** The next phase of work is
> [`openchoreo-audit-logging-observer-plan.md`](./openchoreo-audit-logging-observer-plan.md),
> now sequenced as **P1.5** in §3.1 — not P2's canonical event schema.
>
> **Why this ordering is right, and not merely what was chosen.** §2.12's PR-8 row and P7's own
> goal statement both say the same thing: *"Adding audit coverage to a new service requires only:
> declare actions, initialise the logger, add the middleware"* has **never been exercised twice**.
> `openchoreo-api` is the only consumer. Doing observer **before** P2 means the reusability claim
> is tested against the *current* core, while P0's shape is still fresh and cheap to change — and
> P2 then versions a schema that two services have actually produced, rather than one. Deferring
> it to P7 (where the phased plan originally parked it) would mean discovering the core's
> service-specific hardcoding after P2, P3 and P5a had all built on top of it.
>
> **The observer plan's own validation already found that hardcoding**, which is the evidence
> this ordering is load-bearing rather than tidy: the middleware and `PolicySet` are genuinely
> generic, but `tools/auditgen`'s derivation tables and
> `internal/openchoreo-api/config/audit.go`'s selector vocabulary are both hardcoded to
> openchoreo-api and must be extracted before a second service can use them. **That is PR-8's
> real cost, measured.**
>
> **Three further effects on this document:**
>
> | Item | Effect |
> |---|---|
> | **Task 7.22** (auth on Observer's internal port 8081) | Unchanged in scope, but P1.5 makes it *cheaper and more valuable*: P1.5 exempts the four port-8081 mutations for want of a real actor, and wires the audit middleware onto that mux anyway. Lifting the exemptions after 7.22 lands is then a table edit, with **no wiring change** |
> | **§2.6's "the Observer's generated server is dead"** | Promoted from an observation to a **required gate**. `BuildPatternMap` cross-references operationIds against the spec but never sees the mux, so a hand-registered pattern that drifts from its spec path silently unaudits that operation. P1.5's coverage test must assert registered-pattern ↔ spec-path agreement. This is the same test P7's item 6 already asks for — **P1.5 delivers it early**, for audit's benefit and P7's. **Superseded 2026-08-31:** the dead generated server now has its own plan — [`openchoreo-observer-generated-server-plan.md`](./openchoreo-observer-generated-server-plan.md) — which fixes the root cause instead of gating it, and would retire P1.5's gate as vacuous. It also found that **the spec declares `BearerAuth` on the four port-8081 alert-rule operations that are served unauthenticated**, which makes task 7.22 a decision that plan must take rather than defer |
> | **§2.11's "Observer's access log has no request ID at all"** | Fixed as a side effect of P1.5's logger swap, rather than waiting for P4's tracing work |
>
> **Two P1 follow-ups were also recorded on 2026-08-30**, from PR
> [#4571](https://github.com/openchoreo/openchoreo/pull/4571) — neither blocks P1.5 or P2. See
> the P1 plan's §0.8 and §0.4: **1.16** (the `SetResource`-before-conversion rule is unguarded)
> and **1.17** (no test walks `GetOperations()` asserting hierarchy coverage, which §0.7 asked
> for). #4571 itself closed the "`/mcp` 401 emits nothing" half of Appendix E.10.

> ### 🔄 Revision 2026-08-24 — P0 and P1 are both shipped
>
> **P1 is implemented**, not pending: the generator, the full MCP binding table, exemptions, the
> coverage gate, the coverage matrix, PR-7's second middleware instance, `/exec/` and wirelogs,
> and the `X-Request-ID` hardening. Ten of thirteen tasks are done, six of eight exit criteria
> met. Two verification tasks remain open — **1.13** (demonstrate each gate failure by deliberate
> breakage) and **1.14** (live-verify exec and wirelogs) — and neither blocks P2.
> [`openchoreo-audit-logging-p1-plan.md`](./openchoreo-audit-logging-p1-plan.md) **§0 is the
> as-built record**; its **Appendix E** lists the twelve things that shipped which no plan asked
> for.
>
> **Two citation corrections that affect this document.** The branch was rebased, so
> `6bd147a1`, `96fe08ca` and `8c8b33b7` are **no longer reachable** — cite PR
> [#4462](https://github.com/openchoreo/openchoreo/pull/4462) and its successors instead. And
> `main` is behind this branch's base (`git merge-base main HEAD` = `61e15120`), so §2's
> `grep`-against-`main` reproductions should be run against the branch, or scoped as
> `431092a7^..8d257ca9`.
>
> **Four of P1's unplanned deliveries change a later phase's scope.** Each is folded into the
> phase section and task table below, and named here so it is not discovered late:
>
> | Shipped in P1 | Changes |
> |---|---|
> | **`resource.namespace`** on both surfaces, nested under `resource` | **Task 2.5** becomes a *partial migration*, not greenfield — and it needs handler cooperation, because `SetResource` replaces rather than merges. Also unlocks a `namespaces` policy selector earlier than the P1 plan's Appendix C predicted |
> | **`resource.id`** at 78 REST sites but only 6 MCP sites | **Task 2.12** re-scopes from "add `resource.uid`" to "finish MCP and close a surface-parity gap" |
> | **`actor.entitlements`** as an emitted event field | **Task 2.17** (PII) and **task 3.9** (redaction) each gain a field they did not know about |
> | **`Event.MarshalJSON`** — two render paths held in step by one test | **Task 2.1** and every field P2 adds must land in both. It is also P5b's sink wire-shape contract, already load-bearing |

> The scope document deliberately stops short of phasing: *"It does not propose implementation phasing; that follows once the design is agreed."* This document is that phasing. It owns **sequencing only** — the scope doc remains the source of truth for what each requirement means.
>
> Every phase carries a **Demo** section. That is not decoration; it is the test used to cut the phases. A phase that cannot produce a runnable demonstration of something a person cares about is mis-cut, and has been folded into one that can. This is why there is no trailing "documentation and polish" phase — documentation is a deliverable *of* the phase that produces the thing being documented.

> ### ⚠️ Read §2.1 first
>
> Verification against `main` found that **the audit framework is unreachable dead code, and this is a regression rather than an unfinished feature.** The scope document's status column — assessed against PR #1465 — no longer describes this repository. Several requirements it marks ✅ or ◐ are in fact ○. Phase 0 is consequently a restoration phase, not a defect fix.

> ### 🔄 Revision 2026-08-10 — re-cut for a log-based initial delivery
>
> The initial delivery is now scoped to a **log-based audit system**: audit events are emitted as structured log records, collected by the log-collection agent, and routed to a dedicated audit stream and index in the observability module. Publishing and buffering are the collecting agent's job, not the producer's.
>
> Phases are therefore split across **two delivery tracks**, and no phase number has changed — see §3:
>
> - **Track 1 (initial delivery):** P0, P1, P2, P3, P4, **P5a**, P7, P8, **P10a**
> - **Track 2 (deferred):** **P5b** (webhook and multi-sink transports), **P6** (in-process durable queue and reliability), **P9** (client producers), **P10b** (blocking mode and write-ahead as a gate)
>
> Two consequences are load-bearing and are recorded rather than left to be discovered:
>
> 1. **P4 becomes the critical path, not a parallel track.** Deferring transport to the collecting agent only works if the agent exists, and §2.11 verified that the control plane has no log collection at all today. Everything from P5a onward now depends on it.
> 2. **SE-5 — *"an operator can distinguish 'nothing happened' from 'we stopped recording'"* — is only partially met in Track 1.** The `sequence` counter that makes a gap provable is part of the proposal's core event schema and ships with **P2**; what defers with P6 is the drop counter, the delivery metrics, the "audit degraded" condition and the heartbeat. So a gap is detectable after the fact but nothing warns an operator that one is forming. The full deferred-requirement register is **Appendix C**.

> ### 🔄 Revision 2026-08-13 — P1 split, and the configuration model
>
> **The core framework moves into P0**, cut **core-then-coverage**. P0 now delivers the surface-agnostic
> core, the configuration model and both the REST and MCP adapters, on top of the restoration work that
> shipped as `6bd147a1` — see that plan's §5b. **P1 becomes coverage and enforcement only** and adds no
> audit machinery. The rationale is in §3.1.
>
> **[`openchoreo-audit-logging-p1-plan.md`](./openchoreo-audit-logging-p1-plan.md) is authoritative for
> P1** and supersedes §4's P1 section, which is stale in three respects — see that document's Appendix A
> for the ten deviations. The two that most change the work:
>
> - **MCP is audited by a protocol-level receiving middleware**, not the service-layer emission §2.4
>   proposed. It observes the authz filter's denials, needs no service-layer plumbing, and cannot
>   double-count. But the tool↔operation mapping is **not derivable**, so a declared 69-entry binding
>   table is unavoidable.
> - **`category` is stamped from resource kind by the generator and is *not* operator-configurable.**
>   Operator flexibility lives in an ordered `policies` list with event-field selectors, first match
>   wins. Reason: category is immutable in published records and is what backends key retention on, so
>   reclassifying silently splits history and leaves already-indexed events on the old policy.
>
> **OP-1 and OP-3 move from P5a into P0**, since the policy machinery has to exist there anyway.

---

## 1. How to read this

Three repositories are in scope, and each phase names which it touches:

| Repo | Role in this feature |
|---|---|
| [`openchoreo/openchoreo`](https://github.com/openchoreo/openchoreo) | Producer framework, event schema, transports, reliability, Observer read API, control-plane telemetry |
| [`openchoreo/community-modules`](https://github.com/openchoreo/community-modules) | Audit storage and the adapter-side query implementation |
| [`openchoreo/backstage-plugins`](https://github.com/openchoreo/backstage-plugins) | Portal viewer surface, and the portal as a client producer |

Epic #3343 frames the goal as *"an extension (or similar) to the observability plane, which will allow audit logs to be stored and queried as part of an optional core module"*, with three sub-tasks. They land as follows:

| Epic sub-task | Delivered by |
|---|---|
| 1. Re-evaluate the overall auditability of OpenChoreo's operations | **P1** (coverage matrix, DO-2) |
| 2. Tutorials for extracting and publishing audit logs to an external system | **P5a** (audit stream and index routing, DO-4) in Track 1; **P5b** (webhook to an arbitrary external system) in Track 2 |
| 3. An optional Audit Logs module — UI + audit event storage | **P7** (storage + read API) and **P8** (UI) |

**Ordering principle: foundation first.** Coverage and the event schema are frozen before storage, read API and UI are built on them. The alternative — a thin vertical slice to a demo sooner — was considered and rejected: an index mapping, a query contract and a UI table built on a schema that then gains first-class tenancy, `producer`, `origin` and `audit_id` fields all get reworked. The cost of waiting is a later first screenshot. The cost of not waiting is paid three times.

**P4 is the critical path.** Control-plane observability is not in the audit scope doc and its value stands alone, which is why it belongs to a separate epic. But it is **no longer parallel**: the 2026-08-10 revision defers publishing and buffering to the log-collection agent, and §2.11 verified that the control plane has no log-collection agent today. Audit records cannot leave `openchoreo-api` without it. The reasoning is in the P4 section.

**A note on cross-references.** Bare `§4.x` in this document always means a **numbered open decision in the scope doc's §4** (`§4.1` Trust model, `§4.2` Read path, and so on) — never a section of this document. Phases are always referred to as P0–P10, with the `a`/`b` suffixes introduced by the 2026-08-10 revision distinguishing the Track 1 and Track 2 halves of P5 and P10. The scope doc has no §4.5; its §4.6 is supporting evidence rather than an open decision, which is why it appears in P10b but not in the gate table.

---

## 2. Verified current state

Assessed against `main` at `2cfe6773`. This section supersedes the scope doc's §1, which was assessed against PR #1465, and resolves the three items the scope doc's Appendix lists as *"not verified against the wider codebase"*.

### 2.1 The audit framework is dead code — and it is a regression

**Nothing imports the audit middleware.** Verified:

```sh
$ grep -rln "server/middleware/audit" --include="*.go" .
internal/openchoreo-api/audit/definitions.go        # the only importer

$ grep -rn "audit\.SetResource\|audit\.AddMetadata\|audit\.NewMiddleware\|audit\.NewLogger" --include="*.go" .
                                                     # no results
```

`GetActionDefinitions()` has zero callers. All seven handler-side helpers in `context.go` (`SetResource`, `AddMetadata`, `AddMetadataBatch`, …) have zero callers. The middleware is registered on no route group in any service.

**It was wired, and a refactor removed it.** PR #1465 merged the framework *and* wired it into the then-current handler layer. Commit `a64ea04d` — *"refactor(api): remove legacy routes and migration router (#2588)"* — deleted `internal/openchoreo-api/handlers/`, and the wiring went with it. The wiring is recoverable from history:

```sh
$ git show a64ea04d^:internal/openchoreo-api/handlers/handlers.go | grep -n "auditMiddleware\|routes.With"
73:	auditMiddleware := h.initAuditMiddleware()
85:	api := routes.With(jwtAuth, auditMiddleware)
288:// initAuditMiddleware initializes the audit logging middleware
```

**Two consequences that shape the whole plan.**

First, **P0 is a restoration phase**, not a defect fix. Frame the work as *restore, then extend*.

Second, **the regression must be made impossible to repeat.** `definitions.go` has no compile-time relationship to any route today, so the API layer was rewritten out from under the audit hook once with nothing failing. The CI gate in P1 is not a nice-to-have — it is the only thing standing between this plan and a second silent removal. That reframes PR-3 from "keeps coverage honest as the API grows" to "keeps the feature alive at all".

### 2.2 The action definitions are stale: 6 of 24 match a live route

`definitions.go` declares 24 actions written against the **legacy** route shapes. Against the routes actually registered in `internal/openchoreo-api/api/gen/server.gen.go`, only **6** still match. Representative breakage:

| Definition | Reality on `main` |
|---|---|
| `POST /api/v1/namespaces/{ns}/projects/{project}/components` | Components are now **flat under namespace**, not nested under project |
| `PATCH …/components/{componentName}` | Update is **`PUT`**, not `PATCH` |
| `POST /api/v1/authz/roles` | Now `…/namespaces/{ns}/authzroles` and `/api/v1/clusterauthzroles` |
| `PUT /api/v1/authz/role-mappings/{mappingId}` | Now `authzrolebindings` / `clusterauthzrolebindings` |
| `POST /api/v1/apply`, `DELETE /api/v1/delete` | **Do not exist** |

The verb census confirms the scale of the drift — `server.gen.go` registers **114 state-modifying routes** and **not one is `PATCH`**:

```sh
$ grep -oE 'm\.HandleFunc\("(POST|PUT|PATCH|DELETE) ' internal/openchoreo-api/api/gen/server.gen.go | sort | uniq -c
  38 m.HandleFunc("DELETE
  42 m.HandleFunc("POST
  34 m.HandleFunc("PUT
```

Every `PATCH` definition in `definitions.go` is therefore provably dead. **The baseline for PR-3 is roughly 108 of 114 state-modifying routes with no audit definition**, and `definitions.go` needs a rewrite rather than additions. (Caveat: those 114 are the generated OpenAPI routes only. `/mcp`, `/exec/` and the wirelogs `GET` are registered outside `server.gen.go` — see §2.4.)

This also means **PR-2 is ✅ on paper and ❌ in fact.** Authorization-change auditing is declared, and all six of its patterns are wrong.

### 2.3 One claimed defect is confirmed; the other cannot exist on `main`

**Confirmed — `extractActor`** (`internal/server/middleware/audit/middleware.go:126-130`):

```go
// Extract actor ID from entitlement values (first value is typically the user ID/email)
actorID := "unknown"
if len(subjectCtx.EntitlementValues) > 0 {
    actorID = subjectCtx.ID
}
```

`subjectCtx.ID` (the `sub` claim) is gated on an unrelated field, and the comment describes code that no longer exists. This is reachable in practice: `detectUserTypeFromClaims` (`auth/jwt/resolver.go:118-120`) returns `(true, []string{})` for a present-but-empty entitlement claim, so a successfully authenticated subject lands here and is recorded as `actor.id = "unknown"`. Defeats EM-7.

**Not present — `component_workflows.go`.** The file does not exist (`find . -name 'component_workflows*.go'` → nothing), and `grep -rn setAuditResource` → zero hits. The defect was real *in PR #1465* — `setAuditResource(ctx, …, workflowRun.Name, …)` before the `err != nil` check — but the file was deleted by `c30c5a4e`, and the helper by `a64ea04d`. **There is nothing to fix.** The pattern is still worth encoding as a review rule, because ~18 call sites in that diff shared the same "call immediately after the service call" shape and P0 will be re-creating them.

**A third defect worth adding.** `getRequestID` (`middleware.go:142`) reads `X-Request-ID`, and the logger middleware accepts an **inbound client-supplied** value unvalidated (`logger/middleware.go:39-47`). A caller can therefore choose the correlation ID of its own audit events. That is a forgery and collision vector against a downstream SIEM, and it matters specifically because **EM-16 proposes making `request_id` the trace correlation ID**.

### 2.4 MCP does not wrap the API at the HTTP layer — PR-4 is bigger than stated

The scope doc's EM-11 and PR-4 rest on *"MCP wraps the API rather than sitting beside it"*, concluding that MCP auditing is *"a matter of routing MCP-served requests through the same middleware and carrying `origin: mcp` through, not a second set of action definitions."*

**That is true at the service layer and false at the HTTP layer.** `mcphandlers.NewMCPHandler(svc)` calls the **service** layer directly; MCP requests never traverse the generated OpenAPI routes. Since the audit middleware resolves actions by *route pattern*, a route-pattern middleware cannot capture MCP calls at all.

There are three separate middleware chains in `cmd/openchoreo-api/main.go`:

| Chain | Composition | Audit today |
|---|---|---|
| Generated OpenAPI routes (`:227-230`) | `loggerMiddleware → authMiddleware → WebhookRawBodyMiddleware → handler` | none |
| `/mcp` (`:211-217`) | `mcpLoggerMw → mcpAuth401Mw → jwtMiddleware → handler` | none |
| `/exec/` + wirelogs (`:261-264`) | bare `jwtMiddleware`, deliberately outside the OpenAPI chain | none |

Two further constraints:

- **`/exec/` cannot use the current `responseWriter`.** It implements neither `http.Hijacker` nor `http.Flusher` (`middleware.go:31-50`), and `/exec/` was moved outside the OpenAPI chain precisely because response-writer wrappers break hijacking. `component:exec` is the highest-sensitivity action in the API and needs a different mechanism.
- **The pre-`#2588` order was `routes.With(jwtAuth, auditMiddleware)`** — audit *inside* JWT. That is exactly the ordering the scope doc identifies as why auth failures were never audited (PR-7). Restoration must not restore the ordering bug.

So PR-4 requires **service-layer audit emission** for MCP, which is a genuinely different mechanism from route-pattern middleware. Budget for it accordingly.

### 2.5 The emit path shares stdout with application logs

`logger.go:104` is `l.slogger.Info("AUDIT-LOG", attrs...)` — synchronous, in the request goroutine, after `next.ServeHTTP` returns. `internal/logging/logging.go:34-37` gives a single `slog` handler writing JSON to `os.Stdout`. The sibling access log is `Info("ACCESS-LOG", …)` on the same stream.

**The only thing distinguishing audit from application logs downstream is the message string.** TR-7 therefore needs a new handler/writer, not a config key.

There is also **no audit configuration of any kind**: `grep -rn "audit" internal/openchoreo-api/config/ cmd/openchoreo-api/config.yaml install/helm/` returns nothing. No config struct, no YAML key, no Helm value, no env var. Everything in group RE and group OP is greenfield.

> **Superseded 2026-08-24 — the whole operator surface shipped in P0/P1.** `AuditConfig`
> (`internal/openchoreo-api/config/audit.go`) carries `enabled`, `defaults.publish` and the
> ordered `policies` list with nine validated selector axes; it is decoded with
> `UnmarshalStrict` so a typo fails startup rather than silently widening a rule. It surfaces
> through `cmd/openchoreo-api/config.yaml`, `values.yaml` (with `@schema` annotations),
> the regenerated `values.schema.json`, and `templates/openchoreo-api/configmap.yaml`.
> **OP-1 and OP-3 are delivered.** OP-5 (task 5.9) extends this rather than starting it, and
> `Loader.UnmarshalStrict` is available to it as a shared primitive. Group RE remains greenfield.

**One good piece of news.** `internal/eventforwarder/dispatcher/dispatcher.go` already implements a bounded queue with a worker pool (`jobs: make(chan dispatchJob, 1024)`), payload pre-marshalling at enqueue, non-blocking enqueue with explicit counted drop, configurable exponential backoff retry, and graceful drain on context cancellation. That is most of TR-1, TR-5, RE-3, RE-5 and RE-6 already written. It is in-memory only — **no durable queue, WAL, on-disk spool or outbox pattern exists anywhere in the repo** — so RE-4 remains greenfield, but P5b/P6 should extract and reuse this rather than start over. Note both are Track 2 — the log-based Track 1 delivery has no producer-side queue at all.

### 2.6 The observability module architecture — Appendix item 3 resolved

Modules are **OpenAPI adapter contracts**, one per signal, implemented server-side by community modules:

| Contract | Generated client | Consumed by |
|---|---|---|
| `observability-logs-adapter-api.yaml` | `internal/observer/api/logsadapterclientgen/` | `service.LogsAdapter` — events use the generated client, **logs are hand-rolled HTTP** |
| `observability-metrics-adapter.yaml` | none | hand-written |
| `observability-tracing-adapter-api.yaml` | none | uses the Observer's *own* generated client |
| `finops-adapter-api.yaml` | none | **no Go consumer in the repo** |

Three facts that matter for P7:

- **Adapter URLs are flat static config, not a registry and not the CRD.** `AdaptersConfig` (`internal/observer/config/config.go:37-46`) has three hardcoded named pairs (`logs.adapter.url`, `tracing.adapter.url`, `metrics.adapter.url`). Nothing reads `ObservabilityPlane` to resolve an adapter URL. **One global adapter URL per Observer process per signal.**
- **There is no module registration or capability-discovery mechanism.** No `ObservabilityModule` CRD; `docs/proposals/0482-modular-architecture-proposal.md` is a packaging strategy with no CRD, registration API or discovery protocol. Adapter `/health` is defined in the spec and **never called**. The *only* capability signal in the entire stack is the HTTP `501` documented on `queryEvents`.
- **The adapter contract has no security scheme at all.** `grep -n "security\|Bearer\|Authorization" openapi/observability-logs-adapter-api.yaml` → zero lines, yet it documents `401` and `403` responses. Observer forwards no token to adapters (`logs_adapter.go:148` sets only `Content-Type`). Observer is the sole trust boundary; **an audit adapter inherits that model and must not be reachable from user networks.**
- **The Observer's generated server is dead.** Routes are hand-registered in `cmd/observer/main.go`, so `server.gen.go` is generated but unused. **Spec and served routes can diverge silently in Observer** — unlike `openchoreo-api`, where `gen.HandlerWithOptions` registers from generated code and a missing method breaks the build.

Also worth flagging independently of audit: **Observer's internal port 8081 serves the entire alert-rule CRUD with only logger and recovery middleware — no JWT, no authz.** State-modifying, network-authenticated only, and unaudited.

### 2.7 The query abstraction generalises by duplication — Appendix item 1 resolved

AX-1 depends on the Observer's query abstraction extending to a second event type. **It does — but by copy-paste, and asymmetrically.**

`EventsQueryRequest` (line 522) hand-copies `startTime`, `endTime`, `limit`, `sortOrder` and `searchScope` from `LogsQueryRequest` (line 401), dropping only the logs-specific `logLevels` and `searchPhrase`. There is no shared base component and no `allOf`. Adding audit means a third hand-rolled request/response pair.

Three harder problems:

- **The scope model does not fit audit.** Both requests take `searchScope: oneOf[ComponentSearchScope, WorkflowSearchScope]`; `ComponentSearchScope` requires `namespace` and is **UID-based** on the adapter side (`projectUid`, `componentUid`, `environmentUid`), with the Observer translating names→UIDs via `uid_resolver.go`. Audit events are frequently not component-scoped — a role change is org-scoped, an `access` rejection resolves to no resource at all. A new `AuditSearchScope` is required.
- **There is no real pagination.** `limit` (max 1000) plus `sortOrder`, with `total` documented as *"capped at 1000"*. AX-2 requires pagination and stable ordering; this cannot be inherited.
- **`501` is declared on `/events/query` only — not on `/logs/query`.** So logs is mandatory and events is optional, and this is load-bearing. See §6.

### 2.8 Backstage: the viewer precedent does not proxy, and the producer surfaces are narrower than assumed

- **`openchoreo-observability-backend` does not proxy the Observer.** Its router has four routes — `/resolve-urls`, `/dataplane-netpol-provider`, and `GET`/`PUT /release-binding`. The **browser calls Observer directly**, using an `x-openchoreo-direct` header that `OpenChoreoFetchApi` consumes and strips, replacing the Backstage token with the **raw IDP access token**. Following that precedent for audit means no server-side chokepoint at which Backstage could enforce permissions — a real decision for P8, not a detail.
- **No numbered or offset pagination exists in any viewer.** Infinite scroll with `hasMore` + `loadMore` only. If audit needs jump-to-page, it is net-new.
- **All 17 scaffolder actions call the OpenChoreo API** with the end user's token (`ctx.secrets.OPENCHOREO_USER_TOKEN`). There is no git-only or side-effect-free action. So the genuine CP-3 surfaces are **not** the scaffolder actions themselves but: local catalog insertion (`ImmediateCatalogService.insertEntity`), annotation writes (`AnnotationStore.setAnnotations`, table `entity_custom_annotations`, also reachable via `PATCH /entity-annotations`), **scaffolder task lifecycle** (who ran which template with which parameters — the largest genuine gap), permission DENY decisions, and `POST /exec/init` session establishment.
- **The Auditor Service is available and unused.** Backstage 1.51.0, `@backstage/backend-plugin-api@1.9.1`; `coreServices.auditor` is `@public`. Zero hits for `auditor` across all plugins. Note it derives its actor from **Backstage's own credentials**, not from `x-openchoreo-token`, and writes to Backstage's own sink — so a naive integration attributes events to the wrong subject and they never reach the Observer.
- **The end user's IDP token is forwarded** on all user-initiated paths; the service token (client-credentials) is used only for the background catalog sync. So the portal *can* support `validated` — but only if the receiving audit API verifies that token and binds the actor to it. In guest mode (`features.auth.enabled: false`) there is no token at all and events can only ever be `asserted`.

### 2.9 Community modules: no shared library, and drift is already live

- **Every module is a standalone Go module with its own copy.** `server.gen.go` is byte-identical across `observability-logs-opensearch`, `-openobserve` and `-aws-cloudwatch`. `internal/observer/client.go` differs only in comments. No module depends on `github.com/openchoreo/openchoreo`.
- **Codegen is manual against an unpinned URL.** Each `Makefile` has `SPEC := https://raw.githubusercontent.com/openchoreo/openchoreo/main/openapi/observability-logs-adapter-api.yaml`, run by hand, output committed, with `embedded-spec: true` freezing the spec bytes into the binary. A spec change on `main` breaks **nothing** until someone regenerates — at which point `var _ gen.StrictServerInterface = (*LogsHandler)(nil)` fails to compile until the new method exists.
- **The drift is observable today.** `501` landed in the core spec on 2026-07-03 (`3829ef63`). `observability-logs-gcp-cloudlogging` regenerated after and has `QueryEvents501JSONResponse`; opensearch, openobserve and aws-cloudwatch do not — and aws-cloudwatch's gen file, committed 2026-07-16, is byte-identical to opensearch's from 2026-06-09.
- **No module descriptor declares a signal type or adapter API version.** `module.yaml` is an images manifest; `Chart.yaml` keywords are prose; the website catalog's `category`/`tags` are free text. If audit support must be *discoverable*, that mechanism has to be invented.
- **`observability-logs-moesif` is the existence proof that a forwarder needs no adapter code** — chart-only, `images: []`, Fluent Bit → OTel Collector → SaaS. Its `values.yaml` holds the repo's only Fluent Bit fan-out/reshaping example.
- **Neither existing collector can ingest audit events.** Fluent Bit in `observability-logs-opensearch` is a single linear pipeline — one `tail` input, one `kubernetes` filter, one `opensearch` output, no `rewrite_tag`, no tag routing. `observability-events-otel-collector` bundles exactly one receiver (`k8seventsreceiver`) — no `otlpreceiver`, no `filelogreceiver`, no `webhookeventreceiver`.
- **The events split is the transplantable pattern:** the query-side module owns the adapter endpoint, index template and ISM retention policy; a *separate* collection module owns transport; the two are coupled only by an index/stream-name convention.

### 2.10 Authorization: a new action is silently inert unless registered

`internal/authz/core/actions.go` is the catalog — a `const` block of ~230 `resource:verb` strings plus a `systemActions` table pairing each with `LowestScope` and `IsInternal`. `AuthzRoleSpec.Actions` is a plain `[]string` with only `MinItems=1` — no enum, no CEL validation, no admission check. The Casbin PDP expands role actions against the catalog, so **a name absent from `systemActions` expands to nothing and grants nothing, with no error anywhere.**

**This trap is already sprung.** `install/helm/openchoreo-control-plane/values.yaml:1409` grants the `developer` role `portal-assistant:invoke`, which does not appear in `actions.go`. It is dead config, and no CI check catches it.

Observer duplicates the action strings in `internal/observer/authz/constants.go` with no shared constant and no compile-time check that the two agree. So adding `auditlog:view` means edits in: the `actions.go` const block, the `systemActions` table, `condition_registry.go` if environment-conditioned, `internal/observer/authz/constants.go`, a new authz decorator, and the Helm bootstrap roles (`logs:view`/`events:view` appear at four places: values.yaml `1277`, `1402`, `1454`, `1562`). `admin` has `actions: ["*"]` and picks new permissions up automatically; **every other role must be edited explicitly**, and `sre` and `rca-agent` already carry broad observability reads — decide deliberately whether audit is in or out for them.

Note also that Observer enforces authz through **service-layer decorators** (`NewLogsServiceWithAuthz`, …), wired once in `main.go` and shared by both the HTTP and MCP handlers — so a new query type gets authz on both surfaces for free *if* it follows that pattern. And `CheckAuthorization` fails **open** on `pdp == nil`.

### 2.11 Control-plane observability does not exist — and EM-16 presumes tracing that is absent

- **Fluent Bit is owned by no chart in this repository.** `grep -rni fluent install/helm/` yields exactly one hit, a stale comment in `openchoreo-observability-plane/values.yaml:7` claiming the chart deploys it. It is in fact a subchart of the external `observability-logs-opensearch` module, enabled at install time by the k3d scripts for the data, workflow and observability planes.
- **Collection is a DaemonSet tailing containers into a single `container-logs-*` index family.** There is no stream, index or source selector anywhere in `LogsQueryRequest` — no notion of separate log streams at all.
- **No OpenTelemetry span instrumentation was found in `openchoreo-api`.** This resolves the scope doc's Appendix item 2 as **outstanding**: EM-16's *"join the trace that already exists"* presumes tracing that is not there. P4 addresses it.
- Observer's access logging is weaker than `openchoreo-api`'s — `logger.Debug("HTTP request", …)` with **no request ID at all**, so Observer requests cannot currently be correlated with anything.

### 2.12 What is actually already satisfied

Re-assessed against `main`, the scope doc's ✅ marks reduce to:

| Requirement | Real status |
|---|---|
| RE-2 (never revert an action because audit failed) | ✅ trivially — nothing reverts |
| ST-1 (OpenChoreo does not store audit logs) | ✅ trivially — nothing stores |
| SE-1 (no bodies logged) | ✅ by construction of the event model |
| SE-2 (no handler-level opt-out) | ✅ by construction |
| PR-2 (authorization changes audited) | ❌ **declared but dead** — all six patterns stale, nothing wired |
| PR-6 (denials audited) | ❌ `determineResult` maps 401/403 → `denied`, but nothing is wired |
| PR-8 (cheap service onboarding) | ◐ the *structure* exists; it has never been exercised twice |
| PR-1, PR-9, EM-2/3/6, AT-1, TR-2, TR-5, ST-4 | ○ — previously ◐, now unwired |

> **Re-assessed again 2026-08-24, after P0 and P1 shipped.** This table described `main` before
> any of the work landed. Current status of the rows that moved:
>
> | Requirement | Now |
> |---|---|
> | **PR-2** | ✅ — 12 operations carry `category: authorization`, on both surfaces, gate-enforced |
> | **PR-6** | ◐ — 401 and 403 are now distinct (`unauthenticated` vs `denied`) on REST and on MCP's authz-filter path. **One documented hole:** in a `?filterByAuthz=false` MCP session a service-layer refusal audits as `failure`, not `denied` (P1 Appendix E.10) |
> | **PR-1** | ✅ — 114 REST operations and 69 MCP tools |
> | **PR-3** | ◐ — the gate exists with all six assertions; the deliberate-breakage demonstrations its exit criterion requires are outstanding (P1 task 1.13) |
> | **PR-7** | ✅ — a second, outer middleware instance emitting `result: unauthenticated` |
> | **PR-4, OP-1, OP-3, EM-4a/4b/4c, EM-7, DO-2** | ✅ |
> | **PR-8** | ◐ still — `openchoreo-api` remains the only consumer. It is **`observer` that will test this**, not P7's read path: see `openchoreo-audit-logging-observer-plan.md`, which found the core genuinely reusable but the generator and config layers hardcoded to openchoreo-api. That is the honest answer to *"how much work was that, really?"* |
> | **EM-3** | ✅ for existing values, plus `unauthenticated`. `pending` is still P10a |
> | **EM-6** | ◐ — `agent` is a *configured subject type*, not a constant, so the decision is closed; but nothing distinguishes an agent-driven call |
> | **EM-14** | ◐ **unexpectedly** — `resource.id` at 78 REST sites, 6 MCP sites. Was a P2 non-goal |
> | **EM-16** | ◐ — the `X-Request-ID` hardening half is done; the `traceparent` derivation still waits on P4 |

---

## 3. Phase overview

### 3.1 Track 1 — the log-based initial delivery

| Phase | Goal | Requirement groups | Primary repo | Size | Blocked by |
|---|---|---|---|---|---|
| **P0** ✅ | Restore the framework, guard it, and settle the core on both surfaces | EM-7, EM-4b, PR-4, PR-6, OP-1, OP-3 | openchoreo | L | — |
| **P1** ✅ | Coverage and enforcement across REST and MCP | PR-1, PR-2, PR-3, PR-7, EM-4a/4c, EM-6, DO-2 | openchoreo | L | P0 |
| **P1.5a** ⬅ **live** | Migrate `observer` onto its generated OpenAPI server | *adjacent scope* — makes the spec enforceable; carries the 7.22 decision | openchoreo | M | — |
| **P1.5b** | Extend audit to `observer` — and thereby *test* PR-8 | **PR-8** *(first real test)*, PR-5 *(partial)*, PR-1/PR-7 on a second service | openchoreo | M | P1, **P1.5a** |
| **P2** | One canonical, versioned, documented event | EM, **RE-7**, DO-1, DO-5, SE-6 | openchoreo | L | P1, **P1.5b** |
| **P3** | Declared, typed, validated object attributes | AT *(less AT-12)*, SE-7 | openchoreo | L | P2, §4.4 |
| **P4** | Control-plane telemetry to a configured obs plane | *adjacent scope* — enables TR-2, TR-7, EM-16 | openchoreo + community-modules | XL | — |
| **P5a** | Route audit to a dedicated audit stream and index | TR-2, TR-7, OP-5, ST-8, DO-4 *(OP-1/OP-3 moved to P0)* | community-modules + openchoreo | M | P2, P4 |
| **P7** | Store and query audit through the module | ST, AX, OP-6, PR-5, DO-6 | community-modules + openchoreo | XL | P2, P5a, §4.2, §4.3 |
| **P8** | Read the audit trail in the portal | AX-3, AX-6 | backstage-plugins | M | P7 |
| **P10a** | Per-category pre-action and post-action events | EM-3 (`pending`), EM-13b, RE-15, RE-15a, RE-15b | openchoreo | M | P2, §4.7 |

**The chain is P0 → P1 → P1.5a → P1.5b → P2 → P5a → P7 → P8.** P3 and P10a both depend only on P2 and can run alongside P5a/P7 by different hands. **P4 gates P5a and therefore everything after it** — see the revision note.

**P0 and P1 are both ✅ shipped; P1.5a is the live phase as of 2026-09-01** — with P4 needing to
start in parallel, since it gates everything from P5a onward and has not begun. P1.5a is the only
phase here that is not itself an audit deliverable: it is sequenced inside the chain because
auditing observer on a hand-registered mux costs machinery that generated routing makes vacuous.

> **Why P1.5b gates P2 rather than running beside it.** P2 fixes the canonical event schema and
> versions it. Extracting the generator and the audit-config vocabulary for a second service
> (P1.5b Phase A) is the only thing that will reveal whether today's `Operation` /
> `OperationDef` / selector-vocabulary shape is genuinely service-neutral or quietly
> openchoreo-api-shaped. Versioning a schema *before* that answer is known is how you end up
> with a v1 that needs a v2 for a reason you could have found a week earlier. P1.5b is sized **M**
> precisely because it is mostly extraction and wiring, not new machinery — it is a cheap test
> of an expensive assumption.

> **Core moved into P0, 2026-08-13.** P0 now carries the **surface-agnostic core, the configuration model and both the REST and MCP adapters**, in addition to the restoration work that shipped as `6bd147a1`. See [`openchoreo-audit-logging-p0-plan.md`](./openchoreo-audit-logging-p0-plan.md) §5b, deliverables 0.7–0.13.
>
> The reasoning is **core-then-coverage**: all the architectural risk sits in the core — whether the two-adapter model holds, whether the MCP middleware observes the authz filter's denials, whether `SetResource` reaches through the MCP context, whether one config genuinely serves both surfaces. P0 answers that on the 12 operations it already covers, where a mistake is cheap. Everything after P0 is then an **improvement on a settled foundation** rather than a reshaping of it.
>
> **P1 is consequently coverage and enforcement only** — the generator, all 114 REST operations, the 69-entry MCP binding table, exemptions, the validation gate, rejection events (`result: unauthenticated`) and the coverage matrix. It adds no audit machinery. [`openchoreo-audit-logging-p1-plan.md`](./openchoreo-audit-logging-p1-plan.md) is authoritative and supersedes §4's P1 section.

> **Second wave, 2026-08-19.** P0 shipped in three commits — `6bd147a1` (restoration), `96fe08ca` (the policy-driven framework), `8c8b33b7` (resource.type consistency and level-gating). The last two moved eight further items out of P1 and changed five others' shape; the consolidated list is **`openchoreo-audit-logging-p1-plan.md` Appendix A.2**. Four effects reach beyond P1:
>
> | Phase | Effect |
> |---|---|
> | **P0** | **Reopened for one change, now closed (0.10a–0.10e).** `MCPBinding` could not express a scope-collapsed MCP tool — REST has two operations per scope, MCP collapses them into one tool routing on a `scope` argument, and 21 tools do this. The key was the tool name alone and a collision was a silent map overwrite, so both scopes resolved to one operation, mis-attributing a namespaced create as a platform-wide cluster one. Fixed as `MCPBindingKey{ToolName, Scope}` with a hard error on collision. It landed in P0 because it is a **core-shape** defect |
> | **P1** | **One prerequisite task, 1.0b** — `Result` gains `unauthenticated`, splitting REST's `401 \|\| 403 → denied` and MCP's `ErrNoSubject, ErrForbidden → denied`, so an unauthenticated request is distinguishable from an authorization refusal. An intermediate revision proposed a third `access` **category** for this, reversed the same day: `Category` groups operation types rather than outcomes, and on MCP the same event arrives on a *bound* tool with a real category, so `access` would have broken surface parity. It was then briefly filed as a P0 task (it fixes a live conflation on already-audited operations) and moved back to P1, since P0 is closed and PR-7 is its main consumer. B5 (task 1.6) depends on it and fires on 401 alone. Against that, B6 shrinks — path-carried resource names are now declarative — and B7's `http.Hijacker` blocker is gone |
> | **P2** | **`origin` and `operation_id` already ship as `Event` fields**, so EM-11 is largely delivered — P2 inherits them rather than adding them. EM-16 still needs P4's tracing |
> | **P3** | `Resource` gained an ungoverned `Metadata map[string]any`. **Nothing may populate it until P3's declaration layer exists**, or P3 inherits exactly the free-form-metadata problem the AT group exists to remove (and SE-1's secret-leakage argument applies). Recorded in P1's non-goals |
> | **P5b** | **Shrinks again.** The `Sink` interface and multi-sink fan-out shipped in P0's `emitter.go`, so TR-4's extension point exists in code; P5b adds the webhook sink and the config block, not the seam |

### 3.2 Track 2 — deferred

Deferred by the 2026-08-10 revision, not descoped. Each retains its full content in §4; only its sequencing changed.

| Phase | Goal | Requirement groups | Primary repo | Size | Blocked by | Why deferred |
|---|---|---|---|---|---|---|
| **P5b** | Webhook and multi-sink publishing | TR-1, TR-3, TR-4, TR-8, OP-2, OP-7, SE-4 | openchoreo | M | P5a | Publishing is the log stream in Track 1; the collecting agent ships it |
| **P6** | Never silently lose an event | RE (async half, less RE-7), TR-6, SE-5 | openchoreo | L | P5b, §4.8 | Buffering and backpressure are the agent's job. **Leaves SE-5 partially met** — `sequence` ships in P2, the metrics and degraded condition do not. See Appendix C |
| **P9** | Portal and other clients produce events | CP, AT-12, DO-3 | backstage-plugins + openchoreo | L | P3, P7, §4.1 | No consumer until the read path and viewer exist |
| **P10b** | Blocking mode and write-ahead as a gate | RE-1 (`blocking`), RE-12, RE-13, RE-14, RE-14a, RE-17, OP-8 | openchoreo | L | P6, P10a, §4.7 | Strictness is a later opt-in; the scope doc itself calls it a dangerous default |

---

## 4. Phases

### P0 — Restore the framework and guard it

**Goal.** Get audit events emitting again, and make a second silent removal impossible.

**Requirements.** EM-7.

**Deliverables.**

1. **Re-wire the middleware** into `cmd/openchoreo-api/main.go`. The old `initAuditMiddleware()` from `a64ea04d^:internal/openchoreo-api/handlers/handlers.go:288` is the starting point. For P0, insert into the generated OpenAPI chain (`main.go:227-230`) **inside the logger** (so `X-Request-ID` is populated) and **inside auth** (so `SubjectContext` exists) — matching the pre-`#2588` position. This deliberately does not audit auth failures; P1 resolves that, and §4's note below explains why it is a design choice rather than a reordering.
2. **A minimum viable set of correct action definitions.** Do not port all 24 — most are stale (§2.2). Restore the handful that match live routes so the framework is demonstrably working, and leave the rewrite to P1.
3. **Fix `extractActor`** — remove the `len(subjectCtx.EntitlementValues) > 0` gate; assign from `subjectCtx.ID` whenever a subject context is present, falling back only when `ID` itself is empty. Fix the misleading comment.
4. **The regression guard.** A Go test in `internal/openchoreo-api/audit/` that imports `gen` and asserts the audit middleware is present in the chain. This is the cheap half of P1's CI gate and it belongs here, because the whole point of P0 is that the wiring stays wired.
5. Re-add `audit` to the config struct only as far as an on/off toggle — full per-category configuration is P5a.

**Explicitly not in this phase.** The `component_workflows.go` defect (does not exist — §2.3). The route-coverage gate, category rename, MCP and auth-failure coverage (all P1).

**Demo.**
```sh
# 1. Audit events emit again
make k3d.update.openchoreo-api
curl -X POST http://openchoreo.localhost:8080/api/v1/namespaces/default/projects \
  -H "Authorization: Bearer $TOKEN" -d '{"name":"demo"}'
make k3d.logs.openchoreo-api | grep AUDIT-LOG | jq   # → an event, actor.id = the sub claim

# 2. A token with no entitlement claim is attributed correctly, not "unknown"
make test GO_RUN_ARGS="-run TestExtractActor ./internal/server/middleware/audit/..."

# 3. The guard: delete the middleware registration line and watch CI fail
make test GO_RUN_ARGS="-run TestAuditMiddlewareWired ./internal/openchoreo-api/audit/..."
```

**Exit criteria.** At least one real request produces an audit event on `main`. `actor.id` is the `sub` claim regardless of entitlements. Removing the wiring fails a test.

**Repos.** openchoreo.

---

### P1 — Coverage and enforcement

> **✅ Shipped 2026-08-24**, with two verification tasks open (1.13, 1.14). See
> [`openchoreo-audit-logging-p1-plan.md`](./openchoreo-audit-logging-p1-plan.md) **§0** for what
> was built, which exit criteria are met, and **Appendix E** for the twelve things that shipped
> beyond plan.
>
> **Superseded 2026-08-13 by that same document.** It is authoritative for P1. Three statements below are stale and are corrected there — read its Appendix A before implementing anything from this section.
>
> | This section said | Corrected to |
> |---|---|
> | Hand-rewrite `definitions.go` against all 114 routes — *"the bulk of the phase"* | **Generated** from the OpenAPI spec. The bulk moves to the 69-entry MCP binding table, which is not derivable |
> | MCP needs **service-layer emission** | A **protocol-level receiving middleware**, which additionally observes the authz filter's denials. No service-layer plumbing and no double-counting |
> | The gate blocks on *"a route with no definition"*, with 4 REST-only assertions | **6 assertions across both surfaces**, blocking on *"a resource kind with no category"* — generation makes an unaudited route impossible, so the original condition can never fail |
>
> Two further changes: `category` is **stamped from resource kind by the generator and is not operator-configurable** (operator flexibility moved to ordered `policies` rules), and **the core framework and both adapters moved into P0** (see that plan's §5b), so P1 adds no audit machinery.

**Goal.** Extend coverage to every state-modifying operation on both surfaces and make it build-enforced, on the core P0 settled. Delivers Epic sub-task 1 as an artefact rather than an opinion.

**Requirements.** PR-1, PR-2, PR-3, PR-7 · EM-3 (existing values; `pending` is P10a), EM-4a, EM-4c, EM-6 · DO-2. Genuinely delivers PR-2 and PR-6, which are ✅ on paper only (§2.12). PR-4, PR-6, OP-1, OP-3 and EM-4b are delivered in P0.

**Deliverables.** See the P1 plan's §5. The design discussion retained below remains accurate.

1. **Auth-failure coverage (PR-7) — and this is not a reordering.** Today `authMiddleware` short-circuits 401s before any inner middleware runs, so audit must be **outside** auth to see a rejection. But it must be **inside** auth to have a populated `SubjectContext` for a resolved action. **One middleware instance cannot be both**, so a single "move it outward" change trades PR-7 for EM-7 and does not work.

   Two workable designs; **pick one in the phase plan and record it**, because an implementer meets this on day one:

   | Design | How it works | Cost |
   |---|---|---|
   | **Two instances** (recommended) | An **outer** instance that emits *only* on a 401 with no `SubjectContext` — `result: unauthenticated`, no resolved operation, anonymous actor. An **inner** instance, unchanged from P0, for everything that gets past auth. | Two registrations; the outer one must be careful to emit nothing on success or on a 403 so no request is double-audited. Clear separation, and the outer instance can skip operation resolution entirely. |
   | **One outer instance, opportunistic actor** | A single instance outside auth reading `SubjectContext` opportunistically — present on success, absent on rejection → `result: unauthenticated` with an anonymous actor. | Simpler wiring, but every audited request now depends on the actor lookup being correctly optional, and a bug there silently degrades *all* events to anonymous rather than just failing the rejection path. |

   The two-instance design is recommended because its failure mode is a missing rejection event, not a platform-wide loss of actor attribution.
2. **`/exec/` coverage** — needs a mechanism that is not the current `responseWriter`, which implements neither `http.Hijacker` nor `http.Flusher`. `component:exec` is the highest-sensitivity action in the API; leaving it unaudited should be a recorded decision, not an oversight.
3. ~~**Category taxonomy (EM-4a/4b/4c)** — replace `resource`/`auth`/`observability` with `resource`/`permission`/`access`; drop `CategoryObservability` (nothing emits it).~~ **Superseded 2026-08-19** (external review of PR #4462): P0 shipped `management`/`authorization` instead of the `resource`/`auth`/`observability` this plan assumed as the pre-P1 baseline, and that taxonomy is now decided as permanent — see `openchoreo-audit-logging-p0-plan.md`'s post-implementation corrections note and Appendix B. No P1 category rename. Encode EM-4b as an **emitter** rule regardless: a denied attempt at a *resolved* operation keeps that operation's category with `result: denied`; a rejection with no attributable operation is distinguished by **`result: unauthenticated`** rather than by a category of its own — decided 2026-08-19, implemented as P1 task 1.0b.
4. **Actor types (EM-6)** — add `agent`; the current comment enumerates `anonymous` but not `agent`. **This needs a determination rule, not a constant** — `actor.type` comes from `subjectCtx.Type`, which is driven by the `security.subjects` config, and EM-11 says a human can drive MCP just as an agent can drive REST, so it cannot be derived from the surface.
5. **Harden `getRequestID`** — stop trusting an inbound client-supplied `X-Request-ID` (§2.3), since EM-16 will promote it to the trace correlation ID.
6. **Coverage matrix (DO-2)** — generated from **effective** config, covering both surfaces, stating what is audited and explicitly what is not, with every exemption's reason.

**Demo.** See the P1 plan's §7. In outline:

```sh
# P0 established one config across both surfaces; P1 proves coverage
#   the same operation via REST and via /mcp → same action and category,
#   differing only in origin, and exactly one event per call
#   an MCP authz denial → result=denied, distinguishable from failure
#   a policy setting publish:false takes effect on both surfaces

# Coverage is enforced
make audit.coverage-matrix && cat docs/audit/coverage-matrix.md
make test GO_RUN_ARGS="-run TestAuditCoverage ./internal/openchoreo-api/audit/..."
#   add a route whose resource kind has no category  → FAIL
#   register an MCP write tool with no binding       → FAIL
#   rename an operationId                            → FAIL (orphan binding)
curl -X POST .../api/v1/... -H "Authorization: Bearer <expired>"   # → result=unauthenticated
```

**Exit criteria.** See the P1 plan's §8, split per sub-phase. In summary: every state-modifying operation on **both** surfaces audited or exempted with a reason; the gate fails on an unmapped route, an unmapped MCP write tool, a renamed operationId and an unclassified resource kind; one `policies` entry takes effect identically on both surfaces; rejected tokens emit `result: unauthenticated` while authz refusals on resolved operations emit `result: denied` with their real category, on both surfaces; no definition references `observability`; `/exec/` audited or its exclusion recorded.

**Repos.** openchoreo.

---

### P1.5a — Migrate `observer` onto its generated OpenAPI server · **Track 1** ⬅ live

**Goal.** Make `openapi/observer-api.yaml` enforceable. `internal/observer/api/gen/server.gen.go`
is generated in full — router, strict interface, typed parameter binding, embedded spec — and
**none of it serves traffic**; observer hand-registers its 18 routes on two `http.ServeMux`
instances, so the spec is documentation nothing checks and the generated code is dead weight
`make code.gen` keeps refreshing.

**Authoritative document:**
[`openchoreo-observer-generated-server-plan.md`](./openchoreo-observer-generated-server-plan.md).

**Why it is sequenced here, ahead of audit.** Auditing a hand-registered mux requires a route-table
package and a bespoke registered-pattern ↔ spec-path gate, purely because `audit.BuildPatternMap`
cross-references operationIds against the spec but never sees the mux. On generated routing both
derive from one source and cannot disagree, so both artefacts become vacuous. Building them and
then deleting them is waste — hence migrate first, audit once.

**It carries a decision that is not audit's to take.** The spec declares `BearerAuth` on the four
internal-port alert-rule operations, which are in fact served with **no authentication**;
`auth.OpenAPIAuth` keys off exactly that declaration, so the migration cannot be auth-neutral. The
caller is the control plane's own `observabilityalertrule` controller, which sends no
`Authorization` header and has no token source. **Task 7.22 therefore becomes a choice this phase
must make** — enforce and fix the controller, or make the spec honest with `security: []` and let
7.22 remove it later. That plan's §1 lays out the options and recommends the latter.

**Two further structural problems it settles:** one spec serving two ports (`HandlerWithOptions`
registers every route on whichever mux it is given), and one `StrictServerInterface` against
observer's two handler types. Recommended resolution is to split the spec.

**Exit criteria.** No hand-written pattern string remains for any spec operation; spec and served
routes cannot disagree; auth behaviour is either unchanged or changed deliberately with the
controller updated in the same release; `/mcp` and the OAuth metadata route still work outside the
generated router; middleware ordering has one definition, driven by a wiring test.

**Repos.** openchoreo.

---

### P1.5b — Extend audit to `observer` · **Track 1**

**Goal.** Bring audit production to `observer-api`, and in doing so turn PR-8 — *"adding audit
coverage to a new service requires only: declare actions, initialise the logger, add the
middleware"* — from a plausible claim into a measured one.

**Authoritative document:**
[`openchoreo-audit-logging-observer-plan.md`](./openchoreo-audit-logging-observer-plan.md).
Read its **§0 first** — it records what P1.5a deletes from this phase, and which staged artefacts
are reclaimed rather than rebuilt.

**Blocked by P1.5a.** Do not start until the migration lands.

**Shape.** Five phases: **A** extract the generator and the audit-config vocabulary into shared
libraries with *zero* behaviour change for openchoreo-api (proven by a byte-identical
`definitions.gen.go`); **B** observer's own generator, exemptions and coverage gate; **C** wire
`cmd/observer`, including the config, the logger swap and the PR-7 unauthenticated instance;
**D** instrument the one genuinely audited operation; **E** the coverage matrix and verification.

**Scale, measured.** 18 spec operations, 14 non-GET, of which **9** are POST-expressed reads
(exempt, same class as openchoreo-api's `Evaluates`), **4** are the unauthenticated port-8081
alert-rule CRUD plus the webhook (exempt for want of a real actor — see task 7.22), and
**exactly 1**, `UpdateIncident`, is genuinely audited today. The small coverage number is the
point: the work is the *extraction*, not the operations.

**What this phase is really testing.** Not observer's coverage — one operation. It tests whether
the P0 core is service-neutral. The answer so far is *mostly*: `audit.Middleware.Handler`,
`audit.NewMiddleware` and `audit.NewPolicySet` are genuinely generic, while `tools/auditgen`'s
derivation tables and `internal/openchoreo-api/config/audit.go`'s selector vocabulary are
hardcoded and must be extracted. **That gap is PR-8's real cost, and finding it before P2
versions a schema is why this phase moved ahead of P2.**

**One hazard survives P1.5a; one is deleted by it.**

1. ~~**Observer's routes are hand-registered**, so a pattern that drifts from its spec path
   silently unaudits that operation.~~ **Deleted by P1.5a** — generated routing derives the router
   and the audit pattern map from one source. No gate is needed, and if someone proposes adding
   one, the answer is that it can no longer fail.
2. **Audit must sit *inside* auth**, mirroring `APIMiddlewares`' documented ordering. Still live:
   `Middleware.Handler` captures its context before calling `next`, so a subject added downstream
   by the JWT middleware never reaches it, and every event records `actor: anonymous` while
   nothing fails. Note `[]gen.MiddlewareFunc` is applied **last-to-first** — getting that backwards
   produces exactly this failure. Assert the actor is populated, not merely that the middleware is
   present.

**Exit criteria.** Observer emits a correct, correctly-attributed audit event for
`UpdateIncident` on success and on denial; every other non-GET operation is exempted with a
reason; the coverage gate fails if a new mutating operation is added unaudited *or* if a
registered pattern stops matching its spec path; `openchoreo-api`'s `definitions.gen.go` is
byte-identical to before Phase A; and the PR-8 cost is written down.

**Repos.** openchoreo.

---

### P2 — Canonical event schema v1

**Goal.** Freeze one canonical, versioned, documented event shared by every producer. Everything downstream depends on this being right, which is why it precedes storage and UI.

**Requirements.** EM-1, EM-2, EM-4, EM-5, EM-8, EM-9, EM-10, EM-10a, EM-11, EM-12, EM-13, EM-13a, EM-13c, EM-14, EM-15, EM-16, EM-17 · **RE-7** · DO-1, DO-5 · SE-6.

**Deliberately excluded: the attribute registry** (§4.4 unresolved; AT-3 makes the split safe) and `result: pending` / the two-event model (P10a). EM-13a lands here so `audit_id` exists from the start.

> **RE-7 belongs here, not in P6.** Proposal [#4396](https://github.com/openchoreo/openchoreo/discussions/4396) lists `sequence` in the **core event schema table** — *"Counter within `producer` + `producer_instance`, so a consumer can tell a missing event from no event"* — alongside `producer` and `producer_instance` rather than among the reliability machinery. It is an in-memory integer, not queue infrastructure, so it costs nothing to include and it keeps SE-5's gap detection in Track 1 even though P6 is deferred. P6 retains the surrounding metrics, the degraded condition and the heartbeat.

**Deliverables.**

1. **Envelope**: `schema_version`; `producer` replacing `service` plus `producer_instance` (process-start UUID); **`sequence`, the per-instance counter held beside `producer_instance` so both reset together (RE-7)**; `origin` ∈ `{api, mcp, client}` — **free after P0**, since each adapter knows its own surface, rather than threaded from an MCP marker; first-class `org`/`project`/`component`/`environment`; `event_time` and `received_time`; `audit_id`; `resource.uid`; optional `reason`.
   **Also add HTTP method and path** — the current `Event` carries neither, only the resolved action name. Without them an event cannot be traced back to the request that produced it.

   > **Three corrections from P1's shipped code (2026-08-24).**
   >
   > - **`resource.namespace` already exists**, on both surfaces, seeded before the handler runs
   >   so it survives a denial — but it lives **inside `resource`**, not at the envelope top
   >   level where this deliverable puts tenancy. So the tenancy work starts with a **decision**:
   >   do `org`/`project`/`component`/`environment` join `namespace` under `resource`, or does
   >   `resource.namespace` migrate up? Migrating is a breaking change to a field already being
   >   emitted; keeping it is an inconsistency with the other three. Decide deliberately.
   > - **`resource.uid` is largely done on REST** (78 `SetResource` sites) and barely on MCP (6).
   >   It is no longer a field to add, it is a **surface-parity gap to close** — task 2.12.
   > - **Every field added here must land in two render paths.** `Logger.LogEvent` builds slog
   >   attrs by hand and `Event.MarshalJSON` builds the JSON shape independently;
   >   `TestEvent_MarshalJSONMatchesLogEventShape` is the only thing holding them in step. Add a
   >   field to one and not the other and the two paths silently diverge. Consider whether
   >   `schema_version` is the right moment to collapse them into one.
2. **Actor**: `actor.subject` (`sub`), `actor.session_id` (`sid`), `actor.issuer` (`iss`), `actor.on_behalf_of`, `actor.assurance` (always `validated` until P9).
   **This is cheap** — the full claims map and raw token are already in the request context (`jwt/context.go`: `GetClaims`, `GetClaimValue`, `GetToken`). No re-parsing, no signature re-verification. `iss` is already validated when configured; `sid` is referenced nowhere in the repo today.
   Note `SubjectContext.ID` is built with `fmt.Sprintf("%v", claims["sub"])` — an absent `sub` becomes the literal string `"<nil>"`. Fix that here.
3. **Trace correlation (EM-16)** — take `request_id` from `traceparent`'s trace-id portion when present. **Gated on P4**: §2.11 found no OTel instrumentation, so there is no existing trace to join yet.
4. **Tenancy plumbing** — populate the promoted fields consistently. The bulk of the diff; touches every handler currently using `AddMetadata` for tenancy.

   > **Revised 2026-08-24.** No handler uses `AddMetadata` for tenancy — **the helper no longer
   > exists** (`grep -rn AddMetadata` over the repo returns nothing), and `Resource.Metadata` has
   > no writer either, so there is nothing to migrate off and this deliverable's stated "bulk of
   > the diff" evaporates. What P1 created instead is a different obligation: **`SetResource`
   > replaces rather than merges**, so each of the 78 existing call sites must pass every
   > tenancy field it wants preserved. That is exactly the bug `8d257ca9` fixed for `namespace`
   > across ~30 handler files, and adding three more fields the same way repeats it three times
   > over. **Consider making `SetResource` merge, or splitting tenancy out of `Resource`
   > entirely, before plumbing the remaining three fields.** The middleware already seeds
   > tenancy pre-handler on both surfaces, so the handler ideally should not have to restate it.
5. **Generated schema reference (DO-1)**, from the declarations.
6. **OCSF `api_activity` mapping (EM-17)**, gated on §4.9.2.
7. **IdP correlation guide (DO-5)** and **PII documentation (SE-6)**.

**Demo.**
```sh
make k3d.logs.openchoreo-api | grep AUDIT-LOG | tail -1 | jq   # before/after
make audit.schema-docs && cat docs/audit/schema-reference.md   # [target added by this phase]

curl -X POST ... -H "traceparent: 00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
#   → request_id = 4bf92f3577b34da6a3ce929d0e0e4736, and the event joins the P4 trace
```

**Exit criteria.** Every listed EM requirement emitted and in the generated reference. No handler writes tenancy through `AddMetadata`. A `traceparent` request joins an existing trace. `schema_version` is `1`, with a documented bump policy.

**Repos.** openchoreo.

---

### P3 — Typed attribute registry

**Goal.** Turn per-object-type detail from free-form metadata into a declared, typed, validated registry.

**Requirements.** AT-1…AT-11, AT-13 · SE-7. (AT-12 is P9; AT-9 is specified here, delivered in P7.)

**Blocked by §4.4** — the attribute model shape decision *is* the start of this phase.

> **Two notes from P1 (2026-08-24).**
>
> **The free-form-metadata escape hatch stayed shut.** `Resource.Metadata map[string]any` exists
> on the type, but P1's non-goal held: there is **no production writer** for it, nor for
> `AuditData.Metadata` (verified by grep — the only assignment is the copy inside
> `Event.MarshalJSON`). P1 added 78 `SetResource` call sites without any of them reaching for it.
> So P3's declaration layer arrives **unprejudiced**, which is exactly what AT-6/AT-10 need and
> was the outcome most at risk.
>
> **P3 inherits an AST gate.** `audit_setresource_ast_test.go` statically asserts that handlers
> for body-carried creates, and every `update_*` handler, actually call `audit.SetResource`. It
> guards *presence*, not content — so it is the natural place to extend once attributes are
> declared: a required-attribute check (AT-7) could be enforced the same way, statically, rather
> than only at publish time.

**Deliverables.** A registry per object type with declared types (AT-1, AT-2, AT-4) and an enforced naming convention (AT-8); pre-publish validation acting as an allowlist so metadata cannot leak secrets or unbounded payloads (AT-6, AT-10); a required-attribute check (AT-7); versioning tied to the event schema (AT-11); a documented bound or namespacing strategy on field growth (AT-13); generated attribute docs extending P2's reference (AT-5, DO-1); configurable redaction/hashing (SE-7); and migration of existing ad-hoc metadata keys.

**Demo.**
```sh
make audit.schema-docs && grep -A5 'promote_component' docs/audit/schema-reference.md   # target from P2
make test GO_RUN_ARGS="-run TestAttributeValidation ./internal/server/middleware/audit/..."
make test GO_RUN_ARGS="-run TestRequiredAttributes ./..."
# then: mark an attribute sensitive, confirm the emitted value is hashed
```

**Exit criteria.** No handler writes undeclared metadata. Every ad-hoc key declared or deliberately removed. Generated docs cover every declared attribute.

**Repos.** openchoreo.

---

### P4 — Control-plane observability foundation

**Goal.** Make the control plane's own telemetry — logs, metrics, traces — publishable to a configured observability plane, the way workload telemetry already is. This is **adjacent scope**, not in the audit scope doc, and it is the one phase whose value stands alone.

**Why it is here and not later.** Three of audit's requirements quietly depend on it:

- **TR-2** asks for audit publishing *"through the existing collecting agent to the relevant collector"*. For the control plane there is no existing collecting agent — Fluent Bit is a subchart of a community module enabled for the data, workflow and observability planes, and is owned by no chart in this repo (§2.11).
- **TR-7** asks for audit on a stream separate from application logs. "Separate from what" is only answerable once control-plane logs have a pipeline at all.
- **EM-16** presumes a trace to join. There is no OTel span instrumentation in `openchoreo-api` (§2.11, resolving Appendix item 2).

Building audit transport first would mean inventing a control-plane-specific pipeline for one signal and then reconciling it with the general one. Doing this phase first means audit becomes one more stream on a pipeline that already works.

> **Revised 2026-08-10.** This phase was previously described as independent of P0–P3 and a prerequisite only for P5. **It is now the critical path.** Because the log-based initial delivery hands publishing and buffering to the collecting agent, and no collecting agent exists for the control plane, nothing after P5a can ship without it. It remains independent of P0–P3 and can still run alongside them — but it can no longer slip without slipping the whole delivery.
>
> If that sequencing is unacceptable, the alternative is to pull a **minimal control-plane log-collection slice** (tasks 4.1, 4.2, 4.7) out ahead of the rest of the phase, and let metrics, tracing and the remaining deliverables follow later. That preserves Track 1's dependency while keeping the XL epic off the critical path. Record the choice either way.

**Requirements.** None from the audit scope doc directly — this is new scope. It *enables* TR-2, TR-7 and EM-16, and its stream-separation work is what P5a builds on.

**Deliverables.**

1. **A configuration path from the control plane to an observability plane.** Today `ObservabilityPlane.spec` carries `observerURL`, `rcaAgentURL`, `finOpsAgentURL` — all *read* endpoints — and **nothing reads it to resolve a telemetry target**; Observer's adapter URLs are flat static config (§2.6). Decide whether control-plane telemetry targeting extends `ObservabilityPlane`/`ClusterObservabilityPlane` or is chart-level configuration, and record why. Extending the CRD is the more consistent choice but introduces a control-plane→plane dependency that does not exist today.
2. **Log collection for control-plane components** — `openchoreo-controller-manager`, `openchoreo-api`, `observer`, `cluster-gateway`, `event-forwarder`. All already emit structured JSON to stdout (`internal/logging`), except `observer`, which emits **text** when `LOG_LEVEL=debug` — worth normalising. The collector belongs in `install/helm/openchoreo-control-plane` (which has none today) or a new community module following the `observability-logs-moesif` chart-only pattern.
3. **Metrics** — controller-runtime already exposes Prometheus metrics; establish scrape configuration and confirm `openchoreo-api` and `observer` expose endpoints at all. Target the existing `observability-metrics-prometheus` module.
4. **Traces** — OTel SDK instrumentation in `openchoreo-api`, and ideally the Observer. This is greenfield and it is what makes EM-16 honest. Also fixes Observer's access log carrying **no request ID at all** (§2.11), so Observer requests become correlatable.
5. **Stream separation as a first-class concept.** Today there is one `container-logs-*` index family and `LogsQueryRequest` has no stream, index or source selector. Control-plane system telemetry must be distinguishable from workload telemetry — and the mechanism chosen here is the one audit reuses in P5a. **This is the highest-leverage deliverable in the phase**; get it wrong and P5a works around it.
6. **Collection-side work in community-modules.** Neither existing collector can take arbitrary input: Fluent Bit in `observability-logs-opensearch` is one input, one filter, one output with no `rewrite_tag` or tag routing; `observability-events-otel-collector` bundles only `k8seventsreceiver`. Adding a receiver means editing `builder-config.yaml` and rebuilding the OCB image under its documented `otelcol_version`/Dockerfile-tag/processor-`go.mod` lockstep constraint. `observability-logs-moesif`'s `values.yaml` holds the repo's only Fluent Bit fan-out example and is the better reference for reshaping.
7. **Optional at install, per OP-6's spirit** — a control plane with no observability plane configured must run unchanged.
8. **Helm values, `values.schema.json` regeneration, and operator documentation.** Note `helm-generate` is part of `code.gen`, so hand-editing `values.yaml` without regenerating fails `make code.gen-check`.

**Demo.**
```sh
# 1. Control-plane logs land in the configured observability plane, on their own stream
make k3d   # with an observability plane configured for the control plane
#    → query the obs backend: control-plane logs present, in an index distinct from container-logs-*

# 2. Metrics from the controller-manager are scraped and queryable

# 3. A single API request produces a trace spanning openchoreo-api → observer
#    → and the traceparent is what P2's request_id will join

# 4. Optionality: remove the configuration → the control plane runs unchanged
```

**Exit criteria.** Control-plane logs, metrics and traces reach a configured observability plane. System telemetry is on a stream distinguishable from workload telemetry, by a mechanism P5a can reuse for audit. A control plane with nothing configured is unaffected. Observer requests carry a correlation ID.

**Repos.** openchoreo (primary), community-modules.

---

### P5a — Audit stream separation and routing · **Track 1**

**Goal.** Get audit records onto their own stream, collected and routed to a dedicated audit index with its own retention. This is the log-based half of the original P5. Epic sub-task 2 for the in-cluster case; publishing to an arbitrary external system is P5b.

**Requirements.** TR-2, TR-7 · OP-1, OP-3, OP-5 · ST-8 · DO-4. SE-3 begins here.

> **Revised 2026-08-10.** The original P5 built a producer-side sink abstraction and a webhook transport. In the log-based delivery, **the producer emits and the collecting agent publishes** — so the sink interface, the webhook, per-sink health metrics and the multi-sink filter machinery all move to P5b. What stays is what a log-based system genuinely needs: a distinguishable record, collector-side routing, and a separate index.

**Deliverables.**

1. **Stream separation (TR-7)**, riding P4's mechanism. Today the only thing distinguishing audit from application logs is the message string `"AUDIT-LOG"` on a single `os.Stdout` handler (§2.5). A log-based system needs the record to be **routable by the collector** — either a dedicated writer/stream, or a stable field the collector can match on. Decide which and record it; P5b's sink interface must not require re-doing it.

   > **One more thing TR-7 now owns (2026-08-24).** P0 fixed audit's *level* gating —
   > `forceLevelHandler` makes `Enabled` always return true, so `logging.level: warn` can no
   > longer silence audit and `audit.enabled` is genuinely the only kill switch. It did **not**
   > fix *format*: `NewLogger` reuses the application logger's handler, so whatever format the app
   > configured is what audit gets. `observer` emits **text** when `LOG_LEVEL=debug` (§2.11, and
   > task 4.3), and under a text handler audit records ship unparseable by any collector. So
   > "route audit to its own stream" has to mean its own *format guarantee* too, not just its own
   > destination — which argues for a dedicated writer over a matchable field.
2. **Collector-side routing (TR-2).** §2.9 verified the reference module's Fluent Bit is a **single linear pipeline** — one `tail` input, one `kubernetes` filter, one `opensearch` output, **no `rewrite_tag` and no tag routing**. Routing audit to its own destination is therefore real work, not configuration. `observability-logs-moesif`'s `values.yaml` holds the repo's only fan-out/reshaping example and is the better reference.
3. **A dedicated audit index and retention policy** in the reference module, per §2.9's query-side/collection-side split. Retention is set here, not in OpenChoreo (ST-5).
4. **Reuse P0's `policies` as the single filter.** OP-1 and OP-3 are delivered in P0, so this phase adds no new filtering mechanism — it confirms that the one decision point P0 built is what both the log path and (later) P5b's webhook consult, satisfying "one filter definition driving all sinks" (TR-3).
5. **Configuration (OP-5)** following `openchoreo-api`'s structured koanf pattern — extend P0's `internal/openchoreo-api/config/audit.go` with per-category settings on a `map[string]CategorySettings` keyed by the `ActionCategory` constants. `--dump-config` then verifies the merged result for free. **Do not follow Observer's config pattern** — it is env-only, has a hand-maintained ~40-entry `envMappings` map, loads every value as a raw string, validates on first error, and cannot represent an empty value.

   > **Revised 2026-08-24.** A `map[string]CategorySettings` is the wrong shape now: P0/P1 shipped
   > the **ordered `policies` list** instead, precisely because a fixed per-category grouping kept
   > failing (see the P1 plan's Appendix A item D9). Per-category behaviour is expressed as
   > `match: {categories: [...]}`. So OP-5 becomes *extending the existing selector/settings
   > vocabulary*, not introducing a second, parallel configuration axis — and introducing one
   > would reopen a question already settled.
   >
   > Two primitives are available: **`Loader.UnmarshalStrict`** (an unrecognised key errors rather
   > than being silently dropped — note it drops koanf's default decode hooks, so no durations or
   > `TextUnmarshaler` in a section using it), and `parsePartialSettings`, which already rejects
   > unknown `set:` keys. That is the mechanism by which a new reserved key becomes real in the
   > phase that implements it rather than shipping as inert config.
6. **Volume and sizing guidance per category (ST-8)** and **DO-4** — configuration guides covering the audit stream, the index and retention.

**Demo.**
```sh
# 1. Audit records reach the observability plane on their own stream
curl -X POST .../api/v1/... -H "Authorization: Bearer $TOKEN"
#    → the record appears in the audit index, NOT in container-logs-* or the app log index

# 2. Retention differs from application logs
#    → show the audit index's own ISM/retention policy

# 3. Per-category enablement and per-action suppression
#    disable the `resource` category; confirm nothing reaches the audit index
```

**Exit criteria.** Audit records land in a dedicated audit index, distinguishable from application logs and from `container-logs-*`, with retention configurable in the module. Per-category enablement and per-action suppression effective. The routing mechanism is documented and reusable by P5b.

**Repos.** community-modules (primary), openchoreo.

---

### P5b — Webhook and multi-sink publishing · **Track 2 (deferred)**

**Goal.** Publish audit events to an audit or SIEM system the operator already runs, independently of the log pipeline.

**Requirements.** TR-1, TR-3, TR-4, TR-8 · OP-2, OP-7 · SE-4.

**Deliverables.**

1. **A sink interface behind `LogEvent`** (TR-4). `logger.go:104` is the one seam; adding a sink must touch no producer and no action definition.

   > **Shipped in P0 — and the wire-shape contract came with it (2026-08-24).** `Sink` and the
   > multi-sink fan-out exist in `emitter.go`, with `*Logger` as the only implementation today.
   > Two constraints a new sink must respect, both already encoded:
   >
   > - **`Sink.LogEvent` must not mutate the `*Event`** — the same pointer is handed to every
   >   configured sink, serially.
   > - **Render through `Event.MarshalJSON`, not by hand.** `Event.ResourceType` is `json:"-"` and
   >   is merged into the nested `resource` object only at render time, so a sink calling
   >   `json.Marshal` gets the right shape and one building its own would publish `resource_type`
   >   as a *sibling* of `resource`. `TestEvent_MarshalJSONMatchesLogEventShape` holds the two
   >   paths in agreement; a webhook sink is the first thing that will actually depend on it.
2. **Webhook transport (TR-1)** with TLS, endpoint auth and optional payload signing (SE-4). **Extract and reuse `internal/eventforwarder/dispatcher`** (§2.5) rather than starting over — it already has the bounded queue, worker pool, non-blocking counted-drop enqueue, and exponential backoff. Reuse `internal/observer/notifications/webhook.go`'s **ConfigMap+Secret header-resolution convention** for operator-configured auth headers (it keeps secrets out of ConfigMaps), but not its transport, which has no retry or queue at all.
3. **One filter definition driving all sinks (TR-3)** — generalise P5a's single filter so the log and webhook paths share one definition of what gets audited, with no per-sink duplication.
4. **Configurable forward targets (OP-2)** and confirmation that producer, sink and backend remain three independent extension points (OP-7).
5. **Transport health (TR-8)** — per-sink success/failure counts, queue depth, drop counts. These metric names are what P6's RE-8 extends.
6. **DO-4's worked external-system example**, which P5a's guides defer.

**Demo.**
```sh
# 1. Point OpenChoreo at a webhook receiver and watch events land
#    → the receiver prints the event, signature header present
# 2. Both sinks at once, one filter definition
#    disable the `resource` category; confirm neither sink receives it
# 3. Per-sink health
curl -s http://localhost:<metrics>/metrics | grep audit_sink_
```

**Exit criteria.** Webhook and log-collection sinks deliverable simultaneously from one filter config. The worked external example runs end to end.

**Repos.** openchoreo.

---

### P6 — Durable-asynchronous reliability · **Track 2 (deferred)**

**Goal.** Make `async` a delivery *guarantee* rather than best-effort, and make gaps detectable. The scope doc names the auditor's real question — not *"did you lose an event?"* but *"would you know if you had?"* This phase answers the second.

**Requirements.** RE-1 (`async` only), RE-3…RE-11, RE-16, RE-18 · TR-6 · SE-5. Completes SE-3.

**Blocked by §4.8** — the overflow policy. And by P5b, since the in-process queue exists to serve a producer-side sink.

> **Deferred 2026-08-10.** In the log-based delivery, buffering and backpressure belong to the collecting agent: Fluent Bit's `storage.type filesystem` covers the shipping leg, and container logs remain on disk if the pod dies, so the exposure between emission and collection is small.
>
> **What the deferral costs is SE-5**, and it is a Must: *"an operator can distinguish 'nothing happened' from 'we stopped recording'."* That was delivered by RE-7's per-instance sequence numbers, and no log agent provides them. **P10a partly compensates** — an orphaned `pending` event detects a producer that started an action and never finished it — but it cannot detect events lost in the pipeline *after* emission. Appendix C records this as a knowing gap.

**Deliverables.** A bounded in-process queue (RE-3, largely inherited from P5b's dispatcher extraction) **backed by local disk** with resume on start-up (RE-4 — genuinely greenfield; no durable queue, WAL, spool or outbox exists anywhere in the repo); configurable retry with backoff shared by both modes (RE-5); explicit, configurable, counted overflow (RE-6); **per-instance sequence numbers** scoped to `producer` + `producer_instance` (RE-7); delivery metrics extending P5b's names (RE-8); an **"audit degraded" condition** (RE-9 — `cap(jobs)` vs `len(jobs)` is already observable in the dispatcher); heartbeats (RE-10); at-least-once with `event_id` dedup documented (RE-11); a durable queue for P5b's webhook path (TR-6 — the log path inherits Fluent Bit's filesystem buffering, the webhook path has none); sizing and outage-tolerance documentation (RE-16); and **fault-injection tests** for sink outage, slow sink, overflow, and restart-with-pending-events (RE-18).

On RE-7, keep the scope doc's honesty: it detects loss only *after* emission, and a service that dies before emitting allocates no number. Document that blind spot here — it is the argument for P10a's pre-action event, which Track 1 delivers precisely because this phase does not.

**Demo.**
```sh
# 1. Kill the sink, keep working, restart — nothing is lost
#    stop the receiver
for i in $(seq 1 50); do curl -X POST .../api/v1/... ; done
kubectl -n openchoreo-control-plane rollout restart deploy/openchoreo-api
#    start the receiver → all 50 arrive, sequence 1..50, no gaps

# 2. Force overflow — the gap is counted and provable
curl -s .../metrics | grep audit_dropped_total      # non-zero, and a sequence jump

# 3. Degraded condition without reading metrics
kubectl get <resource> -o jsonpath='{.status.conditions}'   # → AuditDegraded=True

# 4. Heartbeat distinguishes quiet from broken
```

**Exit criteria.** Events survive a restart during a sink outage. Overflow is counted and produces a detectable gap. The degraded condition appears and clears. All four fault-injection scenarios tested. Sizing guidance published.

**Repos.** openchoreo.

---

### P7 — Storage and read path

**Goal.** Store audit events in the observability module and query them through the Observer API, behind a dedicated permission. The Epic's headline deliverable and the largest phase.

**Requirements.** ST-2…ST-7 · AX-1, AX-2, AX-2a, AX-4, AX-5, AX-6 (API side), AX-7, AX-8 · OP-6 · PR-5, **PR-8** · AT-9 · DO-6.

~~**PR-8 is genuinely tested here for the first time.**~~ **Moved to P1.5 (2026-08-30).** *"Adding audit coverage to a new service requires only: declare actions, initialise the logger, add the middleware."* The structure exists but has never been exercised twice — `openchoreo-api` is the only consumer. Bringing audit *production* to the Observer is what turns PR-8 from a plausible claim into a verified one, and **that is now P1.5b, ahead of P2**, not this phase. P7 still owns the Observer's audit **read** path (PR-5, AX-*); it inherits a service that already produces events rather than having to make it produce them. The "how much work was that, really?" output belongs to P1.5b.

> **P7 item 6 is also affected.** It plans to hand-register the north-bound routes and add *another* spec-agreement test, because that was the established observer pattern. **P1.5a removes that pattern** — P7 inherits generated routing and should extend the generated server rather than reintroduce a hand-rolled mux.

**Blocked by two decisions.** **§4.2** — the scope doc states plainly that **ST-6 and AX-1/AX-3 cannot both hold**: in forward-only mode nothing is retained, so the read API has nothing to serve. That is a contradiction inside the requirements and must be resolved, not inherited. **§4.3** governs PR-5, which as drafted is close to empty for a read-mostly Observer.

**And by P5a.** The 2026-08-10 revision makes this explicit: the read path can only serve events that a collector has actually landed in an audit index, so P5a's routing and index work must precede it. The contract work (tasks 7.2–7.5) can proceed in parallel, but nothing is demonstrable until P5a lands. Note also that §4.2's resolution interacts with the log-based delivery — in a forward-only configuration there is no OpenChoreo-managed index for this phase to query.

**Deliverables.**

*Contract (openchoreo):*
1. **`AuditSearchScope`** — neither existing scope works (§2.7).
2. **`/api/v1/audit/query`** on the logs adapter contract, **declaring `501 Not Implemented` from day one.** This is non-negotiable — see §6.
3. **Cursor pagination with stable ordering** (AX-2). The existing `limit` + `total`-capped-at-1000 does not satisfy it. Decide deliberately whether to add this generically (benefiting logs and events, more work now, less duplication later) or only for audit.
4. Filters for time, actor, action, category, resource, result, producer, tenancy **and declared attributes** (AX-2, AT-9), plus collapse-on-`audit_id` (AX-2a).
5. Add the new codegen line to `make/golang.mk`'s `openapi-codegen` target — a new spec is **not** picked up automatically.

*Observer (openchoreo):*
6. North-bound endpoints on `observer-api.yaml`, independently versioned (AX-8), degrading intelligibly (AX-7), with JSON/CSV export (AX-6). **Register the routes by hand in `cmd/observer/main.go`** — the generated `ServerInterface` is unused, so codegen will not error on a path you forgot to wire, and spec/route divergence is silent (§2.6). Add a test asserting every spec path is registered.
7. **A dedicated audit-read permission (AX-4)**, admin / platform-engineer / security-admin only, **not scoped by project membership**. Per §2.10 this means edits in `internal/authz/core/actions.go` (const + `systemActions`), `condition_registry.go` if environment-conditioned, `internal/observer/authz/constants.go`, a new `*_authz.go` decorator, and the Helm bootstrap roles — and **an unregistered action silently grants nothing**, as `portal-assistant:invoke` demonstrates. Also decide whether `sre` and `rca-agent`, which already hold broad observability reads, get it. `LowestScope` needs a deliberate choice: audit events are not naturally hierarchy-scoped, since a `create_namespace` event has no parent namespace to authorize against.
8. **Audit the audit reads (AX-5)** — note there is **no precedent for auditing a read** anywhere in `definitions.go`, so this is a new pattern rather than one more entry.
9. Adapter configuration as a new flat `koanf` key plus a `NewAuditAdapter` at boot, following the events precedent — methods on the existing `*LogsAdapter`, an `AuditAdapter` interface in `pkg/observability/`, and a service-layer authz decorator so HTTP and MCP are both covered. **Generalise `mapAdapterHTTPError`**, which currently hard-codes 501 → `ErrEventsNotImplemented`.
10. Module optional at install (OP-6, ST-4).

*community-modules:*
11. **Reference implementation in `observability-logs-opensearch`**: the query handler, an audit index template for the P2 schema and P3 attributes, and an ISM policy in `init/` so retention is configurable there (ST-5).
12. **All other `observability-logs-*` modules return `501`.** See §6.

*Documentation:* **conformance baseline (ST-7)** and **DO-6** compliance guidance.

**Demo.**
```sh
# 1. Query the audit trail through the Observer
curl -s -H "Authorization: Bearer $ADMIN_TOKEN" -X POST http://localhost:11080/api/v1/audit/query \
  -d '{"startTime":"...","endTime":"...","searchScope":{"namespace":"default"},"category":"permission"}' | jq

# 2. A project member without the permission
curl -s -H "Authorization: Bearer $DEV_TOKEN" -X POST .../audit/query    # → 403

# 3. Attribute filtering; 4. stable pagination; 5. collapse on audit_id
# 6. Stop the adapter → clear error, not an empty success
# 7. Uninstall the module → publishing continues, read API reports unavailable
# 8. The read itself appears in the audit trail
```

**Exit criteria.** Audit queryable end to end against OpenSearch. Permission enforced and not satisfiable by project membership. Attribute filtering and stable pagination work. A 501 module degrades intelligibly. Conformance baseline published. §4.2 resolved in writing.

**Repos.** community-modules (primary), openchoreo.

---

### P8 — Portal viewer

**Goal.** An administrator reads the audit trail in the portal without backend or vendor-console access. The UI half of Epic sub-task 3.

**Requirements.** AX-3 · AX-6 (client side) · AX-2a (presentation).

**Deliverables.**

1. **Decide the transport, first.** The `openchoreo-observability` precedent has the **browser calling Observer directly** with the raw IDP token via `x-openchoreo-direct` (§2.8). Following it needs CORS on the audit endpoint and leaves **no server-side chokepoint** where Backstage could enforce permissions. Breaking it means writing the repo's first real Observer proxy — the `/release-binding` routes are the small precedent. For audit specifically, the permission boundary argues for the proxy.
2. **The view**, reusing `RuntimeEvents/` as the closest template (timestamped discrete events, no log levels) plus the shared primitives in `@openchoreo/backstage-plugin-react` — `VirtualizedLogList`, `useRowExpansion`, `makeColumnStyle`, `EnvironmentFilter`, `TimeRangeFilter`, `ForbiddenState` — and the URL-synced filter hooks (`useUrlFilters*`) so filter state is deep-linkable.
3. **Permission gating** via `permission-backend-module-openchoreo-policy`. The chain is five files: `createPermission` in `openchoreo-common/src/permissions.ts`, an entry in the `openchoreoPermissions` array (without which CONDITIONAL decisions cannot apply), an `OPENCHOREO_PERMISSION_TO_ACTION` mapping, a barrel export, and a `use…Permission` hook cloned from `useLogsPermission`. **An unmapped permission name DENYs**, so P7's server-side action must exist first. Gate the navigation, not just the API.
4. **Pagination.** No numbered or offset paginator exists anywhere in these plugins — infinite scroll with `hasMore` only. If audit needs jump-to-page against P7's cursor pagination, that is net-new UI.
5. Collapse-on-`audit_id` with drill-down (AX-2a); JSON/CSV export (AX-6).
6. **Orphaned `pending` events** must be presented distinctly even though P10a introduces them (RE-15b) — building the affordance now is cheaper than retrofitting, and P10a's demo depends on it. P10a may land before or after this phase; the semantics are its deliverable, the presentation is this one's.
7. Follow repo conventions: a `.changeset/` entry is required and CI enforces it; a new plugin must be added to `.changeset/config.json`'s `linked` array or it versions independently; register in `packages/portal-backend/src/features.ts` and mount as a plain `React.lazy` (**not** `createRoutableExtension` — `rootRouteRef` is never bound to a real path and a routable wrapper throws); mirror the observability plugin's self-disable on a feature flag.

**Demo.** Filter by actor and category, drill into one action, export CSV, then sign in without the permission and confirm the view is absent.

**Exit criteria.** Audit trail readable in the portal. Permission-gated in navigation. Export works. An orphaned `pending` event is visibly distinct from a completed action.

**Repos.** backstage-plugins.

---

### P9 — Client producers · **Track 2 (deferred)**

**Goal.** Let producers outside the OpenChoreo backend publish into the same stream — starting with the portal, for actions that never reach an OpenChoreo API.

**Requirements.** CP-1…CP-9 · AT-12 · DO-3. Exercises EM-12's `asserted`.

**Blocked by §4.1.**

> **Deferred 2026-08-10.** Nothing consumes client-produced events until the read path (P7) and viewer (P8) exist, so building ingestion first would ship an unobservable feature. Two carry-overs keep the deferral cheap: **P2 still adds `actor.assurance` to the schema** (always `validated` in Track 1), so no breaking change is needed later; and **P8 still builds the assurance affordance**, so the viewer does not need reworking when `asserted` events first arrive.

**Deliverables.**

*openchoreo:* a versioned ingestion contract (CP-1); producer authentication with identity established by OpenChoreo, not taken from the payload (CP-4); enforcement of actor assurance per §4.1 (CP-5); schema validation with actionable rejections (CP-6); per-producer rate limiting and implausible-`event_time` rejection preserving `received_time` (CP-7); idempotent `event_id` (CP-8); client-declared attributes under P3's rules (AT-12).

*backstage-plugins:*
- **Scope this against §2.8's correction.** All 17 scaffolder actions already call the OpenChoreo API with the end user's token, so auditing them client-side would **double-count**. The genuine CP-3 surfaces are: **scaffolder task lifecycle** (who ran which template with which parameters, and whether it succeeded — the largest gap), local catalog insertion (`ImmediateCatalogService.insertEntity`), annotation writes (`AnnotationStore.setAnnotations` / `PATCH /entity-annotations`), permission **DENY** decisions and `/evaluate-with-context` outcomes, and `POST /exec/init` session establishment.
- **An Auditor Service bridge** (CP-2) with the documented mapping from `eventId` / `subEventId` / `meta` / `severityLevel`. Two problems to solve, not one: `AuditorService` derives its actor from **Backstage's own credentials**, not `x-openchoreo-token`, so identities must be reconciled; and it writes to Backstage's own logger-based sink, so **a shipper is needed or the events never reach the Observer**.
- Instrumenting `AnnotationStore` and `ImmediateCatalogService` directly catches both the scaffolder and HTTP paths in one place.
- Assurance visibly marked in the P8 view. Note guest mode (`features.auth.enabled: false`) installs `NoOpTokenService` and no token exists at all — those events can only ever be `asserted`.

*Both:* onboarding documented well enough for a third party (CP-9, DO-3).

**Demo.**
```sh
# Run a scaffolder template, find the task-lifecycle event in the portal audit view
#   → producer = backstage, origin = client, assurance visibly marked
# Replay the same event_id → no duplicate
# Post a malformed event → actionable error, nothing stored
# Exceed the rate limit → rejected, received_time preserved on what was accepted
```

**Exit criteria.** A scaffolder run's lifecycle is visible in the portal audit view without double-counting the API-side event. Assurance visibly distinguished. Duplicates idempotent. Onboarding doc sufficient for an external integrator.

**Repos.** backstage-plugins (primary), openchoreo.

---

### P10a — Per-category pre-action and post-action events · **Track 1**

**Goal.** Make an interrupted action detectable. Per-category configuration enabling a two-event model — a `pending` event before the action and a completion event after — so a service that dies mid-action leaves evidence rather than silence.

**Requirements.** EM-3 (`pending`), EM-13b · RE-15, RE-15a, RE-15b.

**Blocked by §4.7** — the pre-action event's scope and default. Only P2 is a code dependency.

> **Re-scoped 2026-08-10.** The original P10 coupled the two-event model to blocking mode, because write-ahead existed to make "the action fails" honest. Track 1 takes the **detection value without the availability risk**: pre-action events are emitted before the action and correlated by `audit_id`, but publishing never gates, delays or fails the action. Blocking is P10b.
>
> This matters more than it did, because it is the only compensation for P6's deferral. Without a pre-action event, a service that dies while performing an action produces **no record at all**, and no amount of gap detection would reveal it — the sequence number was never allocated (RE-7). P10a converts that silence into a detectable "started, never finished".

**Deliverables.**

1. **The two-event model** — `result: pending` on the pre-action event and `success`/`failure`/`denied` on the completion event, correlated by P2's `audit_id`, with **no separate `stage` field** (EM-3, EM-13b). `pending` identifies the pre-action event unambiguously.
2. **Per-category configuration of pre-action emission (RE-15a)**, extending P5a's `map[string]CategorySettings`. **Off by default** — it roughly doubles event volume and adds a second write on every request for enabled categories (RE-15). Kubernetes operators commonly omit `RequestReceived` for exactly this reason.
3. **Orphaned-`pending` semantics (RE-15b).** A mid-action crash leaves a `pending` with no completion. That is the feature — but it must be defined, not incidental: state whether anything reconciles them, and what a consumer is entitled to assume. **Consumers must not assume events come in pairs** (EM-13c): a single-phase action emits one event whose `audit_id` has no sibling.
4. **Presentation contract for the read path.** P7 collapses on `audit_id` (AX-2a) and P8 presents orphaned `pending` events distinctly (task 8.11). P10a owns the semantics those depend on, which is why it can land before or after them but must not contradict them.
5. **Volume guidance**, extending P5a's ST-8 numbers with the doubled-volume case per category.

**Demo.**
```sh
# 1. Enable pre-action events for the `permission` category only
#    → a role change produces two events sharing one audit_id: pending, then success

# 2. `resource` still produces one event
#    → no pending sibling, and the read path does not present it as incomplete

# 3. Kill the service mid-action → an orphaned `pending` with no completion
#    → visible as "started, never finished", not as a completed action

# 4. Confirm the volume cost
#    → roughly 2× event count for the enabled category only
```

**Exit criteria.** Pre-action emission configurable per category and off by default. An interrupted action leaves a detectable orphaned `pending`. A single-phase action's lone event is not presented as incomplete. Publishing never gates or delays the action.

**Repos.** openchoreo.

---

### P10b — Blocking mode and write-ahead · **Track 2 (deferred)**

**Goal.** Offer strictness as an opt-in, per-category operator choice — without making it a dangerous default. Last because it is the highest-risk phase and reuses P6's machinery entirely.

**Requirements.** RE-1 (`blocking`), RE-12, RE-13, RE-14, RE-14a, RE-17 · OP-8.

**Blocked by §4.7, P6 and P10a.**

**Deliverables.** `blocking` mode configurable per category (RE-1, RE-12, OP-8); **write-ahead** publish before the action as a *gate* (RE-14), which is what makes RE-13 honest and RE-2 coherent — P10a already emits pre-action events, so what P10b adds is the gate, not the event; action failure on retry exhaustion (RE-13); **no undo** when publishing fails post-apply, recorded as a metric and surfaced through P6's RE-8/RE-9 (RE-14a); and RE-17's plain documentation that enabling blocking for a category makes an audit outage an outage for those operations — with §4.6's evidence (Vault's unavailability when all audit devices fail, auditd's `disk_full_action halt`) placed where operators read it before enabling, not only in the design discussion.

Note RE-13 has no existing foundation — the current emit is synchronous and best-effort with no failure handling at all, so the synchronous path is new work on top of P6's queue.

**Demo.**
```sh
# 1. Set `permission` to blocking, break the sink, attempt a role change
#    → the operation fails cleanly, and the role is unchanged
kubectl get authzrolebinding <name> -o yaml    # → unmodified

# 2. The same operation with `resource` still async → succeeds, event queues
```

**Exit criteria.** Blocking configurable per category. A blocked operation fails with state provably unchanged. The availability trade documented where operators will read it.

**Repos.** openchoreo.

---

## 5. Decision gates

Nine decisions are open in the scope doc's §4, plus one new to P4. Recommendations are inputs to design review, not closures.

| Decision | Blocks | Recommendation |
|---|---|---|
| **§4.1** Trust model — must clients forward the end-user token, or is `asserted` acceptable? | **P9** *(Track 2)* | Accept `asserted` for client-only actions, **visibly distinguished in the portal**. §2.8 sharpens this: the portal *does* forward the end-user token on API paths, but the genuine client-only surfaces (scaffolder task lifecycle, annotation writes) have no user request in flight, and guest mode has no token at all. Requiring forwarding would block CP-3 outright. |
| **§4.2** ST-6 vs AX-1/AX-3 — stated as mutually unsatisfiable | **P7** | Make forward-only an explicitly **read-path-disabled** configuration: ST-6 stays supported, AX-1/AX-3 documented as unavailable in it, with the Observer reporting "no audit backend configured" per AX-7. Record it rather than discovering it. |
| **§4.3** Observer API reads — is PR-5 meaningful for a read-mostly service? | **P7** | Audit reads of production logs and traces, since that is what exfiltration looks like; **not** general Observer reads. Two notes: this makes EM-4c reversible — an observability category may be wanted after all — and §2.6 found Observer's **internal port 8081 serves alert-rule CRUD with no auth and no authz**, which is a state-modifying unaudited surface worth fixing regardless of how §4.3 lands. |
| **§4.4** Attribute model — typed Go structures, declarative schema, or both? | **P3** | Both, with the declarative schema as source of truth and typed accessors generated from it. A declarative schema is required anyway for AT-12 and DO-1; generating the typed side avoids two definitions that drift. |
| **§4.7** Pre-action event — universal, blocking-only, or configurable? | **P10a** | **Re-opened by the 2026-08-10 revision.** "Blocking-only" is no longer available as a default, because P10a exists in a track with no blocking mode. Recommend **configurable per category, off by default** — enabled for high-value low-volume categories such as `permission`. This is now a detection setting, not a blocking prerequisite. |
| **§4.8** Overflow — drop oldest, newest, or reject? | **P6** *(Track 2)* | **Drop oldest**, counted and gap-detectable, configurable. Dropping newest loses the present, where an active incident lives; rejecting turns the buffer into fail-closed by the back door. Note the existing dispatcher already drops **newest** on a full queue, so this is a deliberate change, not a default. |
| **§4.9.1** Denied and rejected events on by default? | **P1** | Yes — low volume, high signal. Handle the misconfigured-client flood with rate-limited deduplication rather than off-by-default, so the safe configuration is also the default. |
| **§4.9.2** OCSF naming — adopt `api_activity` names or keep ours plus a mapping? | **P2** | Keep OpenChoreo names plus the EM-17 mapping. OCSF names would degrade the fit to OpenChoreo's own abstractions, and the mapping is required either way. |
| **§4.9.3** Authentication topology — is any surface fronted by a BFF or session layer? | **P1** | Establish as fact before P1 starts (task 1.1). §2.8 gives most of the answer for the portal: it forwards the end-user IDP token rather than terminating a session, so it is **not** a BFF for these purposes, and PR-7 does not widen for it. Confirm for any other surface. |
| **NEW — control-plane telemetry targeting** | **P4** | Decide whether it extends `ObservabilityPlane`/`ClusterObservabilityPlane` or is chart-level configuration. The CRD is more consistent with how planes are described, but introduces a control-plane→plane dependency that does not exist today (§2.6: nothing reads the CRD to resolve a telemetry or adapter target). Record the reasoning either way. |

**Already closed** in the scope doc, recorded so they are not relitigated: the read API lives in the Observer API (AX-1); audit read is a dedicated permission limited to administrator, platform engineer and security administrator roles, not scoped by project membership (AX-4); retention is set in the module or external system (ST-5).

---

## 6. Cross-repo strategy

**The fan-out risk.** `community-modules` holds roughly seven `observability-logs-*` modules. A naive reading of P7 turns one phase into seven implementations.

**The rule: one reference implementation, everything else returns 501.** This is what the adapter contract already sanctions — but the mechanism is narrower than it looks, and getting it wrong is expensive.

**`501` is declared on `/events/query` only, not on `/logs/query`.** Verified: `501` appears exactly once in the spec, at line 181. Codegen confirms it — `QueryLogs` has no `501` response type to return, so **a module cannot legally 501 out of logs**. Only `observability-logs-gcp-cloudlogging` currently exercises partial implementation. Where the spec offers no 501, modules resort to a 500 carrying a not-implemented message (`observability-metrics-gcp-cloudmonitoring`, `observability-metrics-azure-monitor`, with tests literally named `isNotImplemented500`).

> **Therefore: `/api/v1/audit/query` MUST declare `501 Not Implemented` in the spec from day one.** If it does not, every existing module becomes contractually obligated to implement audit on its next regeneration, or is forced into the 500-with-a-message workaround. This is the single highest-consequence detail in P7.

The rest of the strategy:

- **`observability-logs-opensearch` is the reference implementation** — it is the default deployment (ST-4).
- **Every other module returns `501`** until someone has a reason to implement it.
- **The conformance baseline (ST-7)** tells operators, before they commit to a backend, whether their choice satisfies the read path.

**Nothing breaks automatically, and that cuts both ways.** Codegen is manual against an unpinned `main` URL with the spec bytes frozen into each binary, so a spec change breaks no build until a module regenerates — at which point `var _ gen.StrictServerInterface` fails to compile until the new method exists. The drift is already live (§2.9). So the 501 addition is safe to land, but P7 should **proactively open the trivial 501 PRs** against the other modules rather than leaving landmines for their next unrelated regeneration.

**Repo sequencing.** P4, P5a and P7 all touch `community-modules`, and each depends on an openchoreo-side contract landing first — the telemetry configuration in P4, the emitted record shape and routing key in P5a, the adapter contract in P7. In every case the OpenAPI or configuration change should merge in `openchoreo` before the module work starts, so the module implements against a fixed contract. P8 and P9 in `backstage-plugins` depend on P7's read API and P9's ingestion contract respectively.

**P5a raises the community-modules stakes.** In the log-based delivery, the reference module owns both the collection side (Fluent Bit routing) and the query side (the audit index and adapter endpoint), where previously P5 could have delivered a webhook without touching a module at all. There is no fallback path if the module work slips — which is a further argument for landing the routing mechanism (task 4.7) early and deliberately.

**Preserve the no-shared-library property.** Each module is standalone, so adding audit to a further module is bounded, additive work in one directory rather than a coordinated change. No phase should introduce a shared module library as a side effect.

---

## 7. Task breakdown

Issue-granularity tasks by phase. **Repo:** `oc` = openchoreo, `cm` = community-modules, `bp` = backstage-plugins.

**Which epic each task belongs to.** Most of these go under Epic [#3343](https://github.com/openchoreo/openchoreo/issues/3343) — but not all, and filing them all there would bury genuinely non-audit work where nobody looking for it will find it:

| Tasks | Epic | Why |
|---|---|---|
| P0–P3, P5–P10 | **#3343** — Audit Logs Module | Directly deliver audit scope-doc requirements |
| **P4 (4.0–4.12)** | **A new epic is needed** — control-plane observability | This is telemetry work: OTel instrumentation, metrics scraping, collector routing, a telemetry-targeting decision. It carries **no requirement IDs from the audit register** and its value stands alone. #3343 should reference it as a dependency of P5a, not absorb it. |
| **7.22** (auth/authz on Observer's internal port 8081) | **A separate security issue — now forced by P1.5a** | An unauthenticated, unaudited, state-modifying surface found during this review. **Updated 2026-09-01:** the spec already declares `BearerAuth` on these four operations, and `auth.OpenAPIAuth` keys off exactly that — so **P1.5a cannot avoid deciding this**. Either it enforces auth (7.22 lands as part of the migration, and the `observabilityalertrule` controller needs a token source) or it makes the spec honest with `security: []` (7.22 later removes it). P1.5b then wires audit onto that mux and exempts the four for want of a real actor, so un-exempting them afterwards is a table edit with no wiring change. |
| **4.12** (stale Fluent Bit comment) | Trivial — fold into any P4 PR | Documentation correction |

Two P4 tasks are worth flagging as decisions rather than implementation: **4.0** (telemetry targeting) and **4.7** (stream separation), the latter because P5a builds directly on whatever mechanism it picks.

**Track labels.** Task IDs are unchanged by the 2026-08-10 revision so existing references stay valid. Tasks belonging to **Track 2** are marked *(T2)* in the tables below; everything unmarked is Track 1. The epic checklist should carry only Track 1 issues, with Track 2 recorded as follow-on work.

### P0 — Restore the framework

| # | Task | Repo | Reqs | Dep |
|---|---|---|---|---|
| 0.1 | Re-wire the audit middleware into `cmd/openchoreo-api/main.go` | oc | PR-1 | — |
| 0.2 | Restore the subset of action definitions that match live routes | oc | PR-1 | 0.1 |
| 0.3 | Fix `extractActor`'s entitlement gate and its stale comment | oc | EM-7 | — |
| 0.4 | Add a test asserting the audit middleware is wired (regression guard) | oc | PR-3 | 0.1 |
| 0.5 | Add a minimal `AuditConfig` with an enable/disable toggle | oc | OP-1 | 0.1 |
| 0.6 | `audit.SetResource` on the six audited handlers | oc | PR-1 | 0.2 |

**0.1–0.6 shipped as `6bd147a1`.** The core-framework tasks below were added by the 2026-08-13 revision;
see the P0 plan's §5b.

| # | Task | Repo | Reqs | Dep |
|---|---|---|---|---|
| 0.7 | Surface-agnostic core: `Operation`, `Envelope`, `Emitter`, `PolicySet` | oc | — | 0.1 |
| 0.8 | Config model: `defaults` + ordered `policies` + selector validation | oc | OP-1, OP-3 | 0.7 |
| 0.9 | REST adapter on the core; exact-map resolver on `r.Pattern` | oc | PR-1 | 0.7 |
| 0.10 | MCP bindings for the P0 operation set | oc | PR-4 | 0.7 |
| 0.11 | Sentinel errors in `pkg/mcp/tools/filter.go` | oc | PR-6 | — |
| 0.12 | MCP adapter + exported constructor + wiring test | oc | PR-4 | 0.7, 0.10, 0.11 |
| 0.13 | Resource resolution on both surfaces; EM-4b in the emitter | oc | PR-1, EM-4b | 0.9, 0.12 |

**0.7–0.13 shipped as `96fe08ca` and `8c8b33b7`.** One further group was added and closed on 2026-08-19 —
the MCP binding key could not express a scope-collapsed tool. See the P0 plan's 0.10a and correction 5.
**P0 is now fully shipped**; the `ResultUnauthenticated` work briefly filed here moved to P1 task 1.0b:

| # | Task | Repo | Reqs | Dep |
|---|---|---|---|---|
| **0.10a** | Sweep the 21 scope-collapsed write tools: confirm the discriminator is always `scope` ∈ {`namespace`,`cluster`} | oc | PR-4 | — |
| **0.10b** | `OperationDef.MCPScope`; key bindings on `(toolName, scope)`, discriminator opaque in core | oc | PR-4 | 0.10a |
| **0.10c** | Resolve scope in the adapter before lookup **and before the resource seed**; absent → `namespace` | oc | PR-4 | 0.10b |
| **0.10d** | Duplicate binding key becomes a loud failure, with a test | oc | PR-3 | 0.10b |
| **0.10e** | Bind one real fan-out (`create_component_type`) so the mechanism has a live colliding case | oc | PR-4 | 0.10c, 0.10d |

### P1 — Coverage and enforcement

Task IDs are unchanged from the original flat `1.x` set, but their **content** changed with the
2026-08-13 revision: the old 1.2/1.5 became a generator, 1.7 split into a binding table plus the MCP
adapter (now P0/0.12), 1.3 moved to P0/0.13, and 1.6 became a six-assertion two-surface gate.

2026-08-19 revision: 1.0 and 1.0b added as **prerequisites** — see §3's second-wave note. 1.7 shrinks
because path-carried resource names are now declarative; 1.8's `http.Hijacker` blocker is gone.

**2026-08-24: P1 is shipped.** Status added below. Two verification tasks — **1.13** and **1.14**
— are new, and neither blocks P2.
[`openchoreo-audit-logging-p1-plan.md`](./openchoreo-audit-logging-p1-plan.md) §0 has the
evidence per task; its Appendix E has the twelve unplanned deliveries.

| # | Task | Repo | Reqs | Dep | Status |
|---|---|---|---|---|---|
| ~~1.0~~ | ~~`MCPBinding` must express a scope-collapsed fan-out~~ — **moved to P0/0.10a–e** | oc | PR-4, PR-3 | **P0/0.10b–e** | ✅ in P0 |
| **1.0b** | **Add `ResultUnauthenticated`** — constant, `validResults` entry, split `determineResult` (401→unauthenticated, 403→denied) and `classifyResult` (`ErrNoSubject`→unauthenticated, `ErrForbidden`→denied) | oc | EM-3, EM-4b, PR-6, PR-7 | — | ✅ |
| 1.1 | Establish the authentication topology per surface (§4.9.3) | oc | PR-7 | — | ⚠️ **no artefact** — carry the conclusion into P2 |
| 1.2 | **Spec→definitions generator**, incl. category stamping from resource kind (singularise the path segment) | oc | PR-1, PR-2, EM-4a, EM-4c | P0/0.9 | ✅ `tools/auditgen`, in `code.gen`. Reads the **embedded** spec, not the YAML — the ordering is load-bearing |
| 1.3 | Full 69-entry MCP binding table | oc | PR-4 | **1.0**, P0/0.12, 1.2 | ✅ 69 tools / **90 binding keys** |
| 1.4 | Exemptions with mandatory reasons | oc | PR-3 | 1.2, 1.3 | ✅ 2 REST + **8** MCP. `GenerateRelease` turned out to *persist*, so it is audited, not exempt |
| 1.5 | The validation gate — 6 assertions, both surfaces | oc | PR-3 | 1.4 | ✅ code (7 assertions) → **1.13** for the demonstrations |
| 1.6 | Second outer HTTP instance, 401-with-no-`SubjectContext` only → `result: unauthenticated`, nil operation | oc | PR-7 | **1.0b**, 1.1, P0/0.9 | ✅ both instances in one constructor, exactly-once proven under panic |
| 1.7 | `SetResource` on the ~38 body-carried REST creates only | oc | PR-1 | P0/0.13 | ✅ **78 sites** — creates, updates and deletes, adding `resource.id` |
| 1.8 | Audit `/exec/` + wirelogs via the standard adapter (`Unwrap()` now exists) — verify a live session first | oc | PR-1 | P0/0.9 | ✅ code → **1.14** for the live check |
| 1.9 | Stop trusting inbound `X-Request-ID` | oc | EM-16 | — | ✅ UUID-or-generate, with a rejection counter |
| 1.10 | **Decide** how `agent` is determined for `actor.type` | oc | EM-6 | — | ✅ **decided:** a configured subject type, not a constant. Residual: nothing *distinguishes* an agent-driven call |
| 1.11 | Coverage matrix from effective config, both surfaces | oc | DO-2 | 1.5 | ✅ **exceeded** — keyed to code coverage (deliberately, so it is reviewable) with its own freshness gate |
| 1.12 | Confirm PR-2 and PR-6 deliver on both surfaces | oc | PR-2, PR-6 | 1.3, 1.5 | ◐ PR-2 ✅; PR-6 has a documented MCP hole (`?filterByAuthz=false`) |
| **1.13** | **Demonstrate each of the six gate failures by deliberate breakage, and record it** — exit criterion 3 requires this explicitly. Assertions 3 and 6 cannot be broken from the data side; demonstrate the upstream construction error instead | oc | PR-3 | 1.5 | ○ |
| **1.14** | **Live-verify `/exec/` and wirelogs through the audit wrapper** — a real `k3d` exec session and a wirelogs tail, which is what the deliverable asked for and the unit tests do not cover | oc | PR-1 | 1.8 | ○ |

### P1.5a — Migrate `observer` onto its generated OpenAPI server

Full detail in [`openchoreo-observer-generated-server-plan.md`](./openchoreo-observer-generated-server-plan.md).
**G.0 is a decision, not a code task — settle it before anything else.**

| # | Task | Repo | Reqs | Dep |
|---|---|---|---|---|
| G.0 | **Decide the auth option (§1) and the spec topology (§2); record both** | oc | — | — |
| G.0a | Set aside the staged observer-audit changes; baseline on `92a7e1ba` | oc | — | — |
| G.1 | If the auth option is "enforce": control-plane token source for the `observabilityalertrule` controller (**= task 7.22**) | oc | 7.22 | G.0 |
| G.2 | Split the spec; shared-schema strategy; second `cfg-server.yaml` + gen package | oc | — | G.0 |
| G.3 | `make code.gen` wiring for the second gen package | oc | — | G.2 |
| G.4 | Convert `*Handler`'s 13 methods to the public `StrictServerInterface` | oc | — | G.2 |
| G.5 | Convert `*InternalHandler`'s 5 methods to the internal `StrictServerInterface` | oc | — | G.2 |
| G.6 | Explicit `ErrorHandlerFunc` rendering `gen.ErrorResponse` — otherwise a missing required FinOps parameter starts returning plain text | oc | — | G.2 |
| G.7 | Reshape ~4,740 lines of handler tests to drive the generated router | oc | — | G.4, G.5 |
| G.8 | `ObserverMiddlewares` / `InternalMiddlewares` composers + a wiring test | oc | — | G.4, G.5 |
| G.9 | Rewire `cmd/observer` via `HandlerWithOptions` per port | oc | — | G.6, G.8 |
| G.10 | Full verification, including a live `k3d` run on both ports | oc | — | all |

### P1.5b — Extend audit to `observer`

Full detail in [`openchoreo-audit-logging-observer-plan.md`](./openchoreo-audit-logging-observer-plan.md).
**Read its §0 before starting any of these** — it records what P1.5a deletes from this phase and
which staged artefacts are reclaimed rather than rebuilt. **Blocked on P1.5a.**

| # | Task | Repo | Reqs | Dep |
|---|---|---|---|---|
| 1.5b.a | **Reclaim** `tools/internal/auditgen` — spec-walking logic behind a `Config` struct. **Gate: openchoreo-api's `definitions.gen.go` byte-identical** | oc | PR-8 | — |
| 1.5b.b | **Reclaim** `internal/auditconfig` — koanf decode/validate behind an explicit `Vocabulary`; `openchoreo-api/config/audit.go` becomes a re-export | oc | PR-8 | — |
| 1.5b.c | `tools/observerauditgen` + `internal/observer/audit/definitions.gen.go`; `observer-audit-gen` in `code.gen` | oc | PR-1 | a |
| 1.5b.d | `exemptions.go` — 13 REST exemptions with reasons (9 `Query*`, 4 internal-port), or 9 if P1.5a enforced auth | oc | PR-1 | c |
| 1.5b.e | Coverage gate: defined-or-exempted, real reasons, categories, count pin, MCP name-and-count pin. **No pattern-agreement assertion — P1.5a made it vacuous** | oc | PR-3 | d |
| 1.5b.f | Observer audit config: `Audit auditconfig.AuditConfig`, loaded from the supplementary YAML, validated in `(*Config).validate()` | oc | OP-1, OP-3 | b |
| 1.5b.g | Swap `observermiddleware.Logger` for the shared `logger.Middleware`. **Delivers task 4.5** — but **not 4.3**: under `LOG_LEVEL=debug` the handler is still `slog.NewTextHandler`, so the line is correlatable and unparseable at once | oc | EM-16 | — |
| 1.5b.h | Compose audit **inside** auth in P1.5a's middleware composer (mind last-to-first ordering); propagate `NewMiddleware`'s error via `logger.Error` + `os.Exit(1)` | oc | PR-1 | c, f |
| 1.5b.i | `NewUnauthenticatedMiddleware` outside auth (`OriginAPI`) and on `/mcp` (`OriginMCP`) — PR-7 parity | oc | PR-7 | h |
| 1.5b.j | Audit middleware on the internal server too, so exemptions lift cleanly once 7.22 lands | oc | PR-1 | h |
| 1.5b.k | `audit.SetResource` in `UpdateIncident`, after the error check | oc | PR-1 | h |
| 1.5b.l | `tools/observerauditcoverage` → `docs/audit/observer-coverage-matrix.md`, with a freshness gate | oc | DO-2 | e |
| 1.5b.m | Verification pass, including a deliberate-breakage proof and an actor-populated assertion | oc | — | all |
| 1.5b.n | **Write down the PR-8 cost** — what extraction actually took, so the claim is measured | oc | PR-8 | m |

### P2 — Canonical event schema v1

> **2026-08-24 — four rows changed by what P1 shipped.** 2.5 and 2.12 are re-scoped, 2.17 gains
> a field, and 2.1 inherits a two-render-path constraint. Detail in the P2 section above and in
> the P1 plan's Appendix E.

| # | Task | Repo | Reqs | Dep |
|---|---|---|---|---|
| 2.1 | Add `schema_version`; document the bump policy — **and keep `Logger.LogEvent` and `Event.MarshalJSON` in step, or collapse them** (P1 Appendix E.5) | oc | EM-4 | — |
| 2.2 | Replace `service` with `producer`; add `producer_instance` | oc | EM-10, EM-10a | — |
| 2.3 | Add HTTP method and path to the event | oc | EM-2 | — |
| 2.4 | ~~Add `origin`~~ — **shipped in P0** as `Event.Origin` (`api`\|`mcp`) alongside `operation_id`. P2 adds only the `client` value, when P9 introduces a client producer | oc | EM-11 | — |
| 2.5 | Promote `org`/`project`/`component`/`environment` to first-class fields — **now a partial migration:** `resource.namespace` already ships on both surfaces, nested under `resource`. **Decide first** whether the other three join it there or `namespace` migrates up | oc | EM-5 | — |
| 2.6 | ~~Migrate all handlers off metadata-based tenancy~~ — **nothing to migrate:** `AddMetadata` no longer exists and `Resource.Metadata` has no writer. **Replaced by:** decide whether `SetResource` should merge rather than replace, before adding three more tenancy fields that all 78 call sites would otherwise have to restate | oc | EM-5 | 2.5 |
| 2.7 | Add `actor.subject`/`session_id`/`issuer` from the context claims map | oc | EM-8 | 0.3 |
| ~~2.8~~ | ~~Fix `SubjectContext.ID` rendering `"<nil>"` for an absent `sub`~~ — **delivered by P0** (deliverable 0.3b: `openchoreo-audit-logging-p0-plan.md` §2.3b, Appendix A item 6) | oc | EM-7 | 0.3 |
| 2.9 | Add `actor.on_behalf_of` and `actor.assurance` | oc | EM-9, EM-12 | 2.7 |
| 2.10 | Add `event_time` and `received_time` | oc | EM-13 | — |
| 2.11 | Add `audit_id`; document EM-13c | oc | EM-13a, EM-13c | — |
| 2.12 | ~~Add~~ **Finish** `resource.uid`, and add optional `reason` — **P1 shipped it at 78 REST sites but only 6 MCP sites**, so this is closing a surface-parity gap, not adding a field. Note delete operations legitimately have no UID (the delete service methods return only an error) | oc | EM-14, EM-15 | — |
| 2.13 | Derive `request_id` from `traceparent` | oc | EM-16 | 1.10, 4.4 |
| 2.14 | Generate the schema field reference | oc | DO-1, EM-1, EM-2 | 2.1–2.13 |
| 2.15 | Write the OCSF `api_activity` mapping | oc | EM-17 | 2.14 |
| 2.16 | Write the IdP correlation guide | oc | DO-5 | 2.7 |
| 2.17 | Document PII-bearing fields — **must cover `actor.entitlements`**, which P1 added as an emitted field without a plan. Note also Appendix D's caveat: on a long-lived MCP session the recorded entitlements are as of session establishment, not as of the action | oc | SE-6 | 2.14 |
| 2.18 | Add `sequence`, scoped to `producer` + `producer_instance` | oc | RE-7 | 2.2 |

### P3 — Typed attribute registry

| # | Task | Repo | Reqs | Dep |
|---|---|---|---|---|
| 3.0 | **Decide the attribute model shape (§4.4)** | oc | AT-1 | — |
| 3.1 | Attribute registry per object type with declared types | oc | AT-1, AT-2, AT-4 | 3.0 |
| 3.2 | Naming convention, enforced | oc | AT-8 | 3.1 |
| 3.3 | Pre-publish validation with allowlist semantics | oc | AT-6, AT-10 | 3.1 |
| 3.4 | Required-attribute check | oc | AT-7 | 3.3 |
| 3.5 | Version attributes with the event schema | oc | AT-11 | 3.1, 2.1 |
| 3.6 | Bound or namespace field growth; document it | oc | AT-13 | 3.1 |
| 3.7 | Extend generated docs to attributes | oc | AT-5, DO-1 | 3.1, 2.14 |
| 3.8 | Migrate ad-hoc metadata keys onto declared attributes | oc | AT-3 | 3.3 |
| 3.9 | Configurable redaction/hashing | oc | SE-7 | 3.1 |

### P4 — Control-plane observability foundation

| # | Task | Repo | Reqs | Dep |
|---|---|---|---|---|
| 4.0 | **Decide control-plane telemetry targeting: CRD vs chart config** | oc | — | — |
| 4.1 | Implement the configuration path to a target observability plane | oc | — | 4.0 |
| 4.2 | Log collection for all control-plane components | oc, cm | enables TR-2 | 4.1 |
| 4.3 | Normalise Observer's log output (text when `LOG_LEVEL=debug`) | oc | — | — |
| 4.4 | OTel SDK instrumentation in `openchoreo-api` and Observer | oc | enables EM-16 | 4.1 |
| ~~4.5~~ | ~~Add a request ID to Observer's access logging~~ — **delivered by P1.5b task 1.5b.g**: the shared `logger.Middleware` emits `request_id` on every ACCESS-LOG line. **But note the dependency inversion**: 4.5 was sequenced *after* 4.3, and now lands *before* it, so under `LOG_LEVEL=debug` — where `initLogger` returns a `slog.NewTextHandler` — the new access log ships correlatable but unparseable. **4.3 is unaffected and still open**; the middleware swap changes the handler's *caller*, not the handler | oc | — | — |
| 4.6 | Metrics exposure and scrape config for control-plane components | oc | — | 4.1 |
| 4.7 | **Stream separation: system vs workload telemetry** | oc, cm | enables TR-7 | 4.2 |
| 4.8 | Add a receiver to `observability-events-otel-collector`, or a new collection module | cm | enables TR-2 | 4.7 |
| 4.9 | Fluent Bit routing/reshaping for the control-plane stream | cm | enables TR-2 | 4.7 |
| 4.10 | Confirm optionality: no obs plane configured → unchanged behaviour | oc | — | 4.1 |
| 4.11 | Helm values, `values.schema.json` regeneration, operator docs | oc | — | 4.1 |
| 4.12 | Fix the stale Fluent Bit claim in `openchoreo-observability-plane/values.yaml:7` | oc | — | — |

### P5a — Audit stream separation and routing

| # | Task | Repo | Reqs | Dep |
|---|---|---|---|---|
| 5.3 | Separate audit from the application log stream, on P4's mechanism | oc | TR-7 | 4.7 |
| 5.6 | Emit audit so P4's collecting agent can route it | oc | TR-2 | 5.3, 4.2 |
| ~~5.8~~ | ~~Per-category enablement and per-action suppression, as one filter~~ — **delivered by P0 task 0.8** | oc | OP-1, OP-3 | P0/0.8 |
| 5.9 | Per-category `AuditConfig` on the koanf pattern; `--dump-config` verified | oc | OP-5 | 0.5, 5.8 |
| 5.12 | Confirm emission never blocks the request path | oc | TR-5, PR-9 | 5.3 |
| 5.13 | Document failure semantics (completes in P6) | oc | SE-3 | 5.12 |
| 5.14 | Audit stream, index and retention policy in the reference module | cm | TR-2, ST-5 | 5.6, 4.9 |
| 5.15 | Per-category volume and sizing guidance | oc | ST-8 | 5.14 |
| 5.16 | Audit stream, index and retention configuration guide | oc | DO-4 | 5.14 |

### P5b — Webhook and multi-sink publishing · **(T2)**

| # | Task | Repo | Reqs | Dep |
|---|---|---|---|---|
| 5.1 | Extract `eventforwarder/dispatcher` into a reusable package *(T2)* | oc | TR-4 | — |
| 5.2 | Sink interface behind `LogEvent` *(T2)* | oc | TR-4 | 5.1 |
| 5.4 | Webhook sink with TLS, endpoint auth, optional payload signing *(T2)* | oc | TR-1, SE-4 | 5.2 |
| 5.5 | Reuse the ConfigMap+Secret header convention for webhook auth *(T2)* | oc | SE-4 | 5.4 |
| 5.7 | Generalise 5.8's filter to drive all sinks *(T2)* | oc | TR-3 | 5.4, 5.8 |
| 5.10 | Confirm producer/sink/backend as three independent extension points *(T2)* | oc | OP-7 | 5.2 |
| 5.11 | Per-sink health metrics *(T2)* | oc | TR-8 | 5.4 |
| 5.17 | Configurable forward targets *(T2)* | oc | OP-2 | 5.4 |
| 5.18 | Worked external-system publishing example *(T2)* | oc | DO-4 | 5.4 |

### P6 — Durable-asynchronous reliability · **(T2 — entire phase)**

| # | Task | Repo | Reqs | Dep |
|---|---|---|---|---|
| 6.0 | **Decide the overflow policy (§4.8)** | oc | RE-6 | — |
| 6.1 | Bounded queue applying backpressure to the buffer | oc | RE-3 | 5.1 |
| 6.2 | **Disk-backed queue with resume on start-up** | oc | RE-4 | 6.1 |
| 6.3 | Configurable retry with backoff, shared by both modes | oc | RE-5 | 6.1 |
| 6.4 | Implement the chosen overflow policy, counted | oc | RE-6 | 6.0, 6.2 |
| 6.5 | Document the pre-emission blind spot *(the counter itself is task 2.18)* *(T2)* | oc | RE-7 | 2.18 |
| 6.6 | Delivery metrics extending P5b's names | oc | RE-8 | 5.11, 6.2 |
| 6.7 | "Audit degraded" condition | oc | RE-9 | 6.6 |
| 6.8 | Periodic heartbeat events | oc | RE-10 | 6.1 |
| 6.9 | Document at-least-once and `event_id` dedup | oc | RE-11 | 6.3 |
| 6.10 | Durable queue for the webhook path | oc | TR-6 | 6.2, 5.4 |
| 6.11 | Document buffer sizing, disk usage, outage tolerance | oc | RE-16 | 6.2 |
| 6.12 | Fault-injection tests: outage, slow sink, overflow, restart-with-pending | oc | RE-18 | 6.2, 6.4 |
| 6.13 | Document gap detectability end to end | oc | SE-5, RE-1 | 6.5, 6.7 |

### P7 — Storage and read path

| # | Task | Repo | Reqs | Dep |
|---|---|---|---|---|
| 7.0 | **Resolve §4.2: ST-6 vs AX-1/AX-3**, in writing | oc | ST-6, AX-1 | — |
| 7.1 | **Decide §4.3: what Observer auditing means for PR-5** | oc | PR-5 | — |
| 7.2 | Add `AuditSearchScope` to the adapter contract | oc | AX-2 | 2.14 |
| 7.3 | **Add `/api/v1/audit/query` WITH a `501` response declared** | oc | ST-3, AX-1 | 7.2 |
| 7.4 | Add the new codegen line to `make/golang.mk` | oc | — | 7.3 |
| 7.5 | Cursor pagination with stable ordering | oc | AX-2 | 7.3 |
| 7.6 | Filters: time, actor, action, category, resource, result, producer, tenancy | oc | AX-2 | 7.3 |
| 7.7 | Declared-attribute filtering | oc | AT-9 | 7.6, 3.1 |
| 7.8 | Collapse-on-`audit_id` with individual events on request | oc | AX-2a | 7.6, 2.11 |
| 7.9 | `AuditAdapter` interface + methods on `*LogsAdapter`; new koanf key | oc | ST-2, ST-3 | 7.3 |
| 7.10 | Generalise `mapAdapterHTTPError` beyond `ErrEventsNotImplemented` | oc | AX-7 | 7.9 |
| 7.11 | North-bound Observer endpoints, hand-registered, independently versioned | oc | AX-1, AX-8 | 7.9 |
| 7.12 | Test asserting every Observer spec path is actually registered | oc | — | 7.11 |
| 7.13 | Register `auditlog:view` in `actions.go` + `systemActions` + observer constants | oc | AX-4 | — |
| 7.14 | Decide `LowestScope` for audit reads; grant in the right Helm roles | oc | AX-4 | 7.13 |
| 7.15 | CI check that Helm-granted actions exist in the catalog | oc | AX-4 | 7.13 |
| 7.16 | Audit-read authz decorator on the service layer | oc | AX-4 | 7.13, 7.11 |
| 7.17 | Audit the audit reads (new pattern — no read-audit precedent) | oc | AX-5 | 7.11, 1.5 |
| 7.18 | JSON and CSV export from the API | oc | AX-6 | 7.11 |
| 7.19 | Intelligible degradation on backend unavailable or 501 | oc | AX-7 | 7.10, 7.11 |
| 7.20 | Module optional at install; publishing continues without it | oc | OP-6, ST-4 | 7.9 |
| 7.21 | Implement PR-5 per the §4.3 decision | oc | PR-5 | 7.1 |
| 7.22 | Add auth/authz to Observer's internal port 8081 alert CRUD | oc | — | — |
| 7.23 | OpenSearch adapter: audit query handler | cm | ST-2 | 7.3 |
| 7.24 | OpenSearch audit index template for the P2 schema + P3 attributes | cm | ST-2 | 7.23, 2.14 |
| 7.25 | ISM retention policy in `init/` | cm | ST-5 | 7.24 |
| 7.26 | Proactively land `501` handlers in all other `observability-logs-*` modules | cm | ST-3 | 7.3 |
| 7.27 | Publish the module conformance baseline | oc | ST-7 | 7.23 |
| 7.28 | Compliance guidance and operator-responsibility split | oc | DO-6 | 7.11 |

### P8 — Portal viewer

| # | Task | Repo | Reqs | Dep |
|---|---|---|---|---|
| 8.0 | **Decide: direct browser→Observer vs a Backstage proxy** | bp | AX-3 | 7.11 |
| 8.1 | Decide: new plugin vs route in `openchoreo-observability` | bp | AX-3 | 8.0 |
| 8.2 | Implement the chosen transport (proxy route or direct client) | bp | AX-3 | 8.0 |
| 8.3 | Declare `openchoreo.auditlogs.view` across the five-file chain | bp | AX-4 | 7.13 |
| 8.4 | `useAuditLogsPermission` hook | bp | AX-4 | 8.3 |
| 8.5 | Audit view reusing `RuntimeEvents` + shared react primitives | bp | AX-3 | 8.2 |
| 8.6 | URL-synced filters for deep-linkable filter state | bp | AX-2 | 8.5 |
| 8.7 | Gate view and navigation on the permission | bp | AX-4 | 8.4, 8.5 |
| 8.8 | Pagination UI against P7's cursor model (net-new) | bp | AX-2 | 8.5, 7.5 |
| 8.9 | Collapse-on-`audit_id` with drill-down | bp | AX-2a | 8.5, 7.8 |
| 8.10 | JSON and CSV export from the UI | bp | AX-6 | 8.5, 7.18 |
| 8.11 | Present orphaned `pending` events distinctly | bp | RE-15b | 8.5 |
| 8.12 | Register in `features.ts`, add to `.changeset` `linked`, feature-flag self-disable | bp | — | 8.1 |

### P9 — Client producers · **(T2 — entire phase)**

| # | Task | Repo | Reqs | Dep |
|---|---|---|---|---|
| 9.0 | **Decide the trust model (§4.1)** | oc, bp | CP-5, EM-12 | — |
| 9.1 | Versioned ingestion contract | oc | CP-1 | 2.14 |
| 9.2 | Producer authentication with OpenChoreo-established identity | oc | CP-4 | 9.1 |
| 9.3 | Enforce actor assurance per §4.1 | oc | CP-5, EM-12 | 9.0, 9.2 |
| 9.4 | Schema validation on entry with actionable rejections | oc | CP-6 | 9.1 |
| 9.5 | Per-producer rate limiting; reject implausible `event_time` | oc | CP-7 | 9.2 |
| 9.6 | Idempotent `event_id` handling | oc | CP-8 | 9.4 |
| 9.7 | Client-declared attributes under P3 rules | oc | AT-12 | 9.4, 3.1 |
| 9.8 | Map Auditor Service events to the OpenChoreo schema | bp | CP-2 | 9.1 |
| 9.9 | Reconcile Auditor's Backstage-credential actor with `x-openchoreo-token` | bp | CP-5 | 9.8 |
| 9.10 | Ship Auditor events to OpenChoreo (its default sink is Backstage's own) | bp | CP-1 | 9.8 |
| 9.11 | Audit scaffolder task lifecycle (template, parameters, outcome) | bp | CP-3 | 9.8 |
| 9.12 | Instrument `AnnotationStore` and `ImmediateCatalogService` | bp | CP-3 | 9.8 |
| 9.13 | Audit permission DENY and `/evaluate-with-context` outcomes | bp | CP-3 | 9.8 |
| 9.14 | Audit `POST /exec/init` session establishment | bp | CP-3 | 9.8 |
| 9.15 | Mark assurance visibly in the portal audit view | bp | EM-12 | 9.3, 8.5 |
| 9.16 | Producer integration and onboarding guide | oc | CP-9, DO-3 | 9.1 |

### P10a — Per-category pre-action and post-action events

| # | Task | Repo | Reqs | Dep |
|---|---|---|---|---|
| 10.0 | **Decide §4.7: pre-action event scope and default** *(re-opened — "blocking-only" no longer available)* | oc | RE-15a | — |
| 10.1 | Add `result: pending` and the two-event model on `audit_id` | oc | EM-3, EM-13b | 2.11 |
| 10.7 | Per-category pre-action emission, off by default | oc | RE-15a | 10.0, 10.1, 5.9 |
| 10.8 | Define orphaned-`pending` semantics and any reconciliation | oc | RE-15, RE-15b | 10.1 |
| 10.10 | Document the doubled-volume cost per enabled category | oc | RE-15 | 10.7, 5.15 |

### P10b — Blocking mode and write-ahead · **(T2)**

| # | Task | Repo | Reqs | Dep |
|---|---|---|---|---|
| 10.2 | `blocking` mode, configurable per category *(T2)* | oc | RE-1, RE-12, OP-8 | 6.3 |
| 10.3 | Synchronous publish path (none exists today) *(T2)* | oc | RE-13 | 10.2 |
| 10.4 | Promote the pre-action event to a write-ahead gate *(T2)* | oc | RE-14 | 10.1, 10.3 |
| 10.5 | Fail the action on retry exhaustion *(T2)* | oc | RE-13 | 10.4 |
| 10.6 | Never undo an applied action on post-apply failure; record it *(T2)* | oc | RE-14a | 10.4, 6.6 |
| 10.9 | Document the availability trade where operators will read it *(T2)* | oc | RE-17 | 10.5 |

---

## Appendix A — Requirement coverage

All **122** requirement IDs in the scope doc are accounted for — verified by set difference in both directions, with no missing IDs and no invented ones. Reproduce:

```sh
grep -oE '^\| (EM|AT|PR|CP|TR|RE|ST|AX|SE|OP|DO)-[0-9]+[a-z]?' openchoreo-audit-logging-scope.md \
  | sed 's/^| //' | sort -u > /tmp/scope.txt
grep -oE '(EM|AT|PR|CP|TR|RE|ST|AX|SE|OP|DO)-[0-9]+[a-z]?' openchoreo-audit-logging-phased-plan.md \
  | sort -u > /tmp/plan.txt
comm -23 /tmp/scope.txt /tmp/plan.txt   # in scope, missing from plan — must be empty
comm -13 /tmp/scope.txt /tmp/plan.txt   # invented in plan — must be empty
```

Counted by **primary** phase — the one that owns delivery. Requirements touched by more than one phase are listed separately below and counted only once here.

| Group | Count | Distribution by primary phase | Track 2 share |
|---|---|---|---|
| EM | 24 | P0×1, P1×5, P2×17, P10a×1 | — |
| AT | 13 | P3×12, P9×1 | AT-12 |
| PR | 9 | P0×1, P1×5, P5a×1, P7×2 | — |
| CP | 9 | P9×9 | **all 9** |
| TR | 8 | P5a×2, P5b×4, P6×1, *shared* ×1 | TR-1, TR-3, TR-4, TR-6, TR-8 |
| RE | 21 | P6×12, P10a×3, P10b×5, already done ×1 | RE-1(blocking), RE-3…RE-14a, RE-16, RE-17, RE-18 |
| ST | 8 | P5a×1, P7×6, already done ×1 | — |
| AX | 9 | P7×8, P8×1 | — |
| SE | 7 | P2×1, P3×1, P5a×1, P5b×1, P6×1, already done ×2 | SE-4, **SE-5** |
| OP | 8 | P5a×3, P5b×2, P7×1, P10b×1, removed ×1 (OP-4) | OP-2, OP-7, OP-8 |
| DO | 6 | P1×1, P2×2, P5a×1, P7×1, P9×1 | DO-3 |

**Spanning more than one phase**, deliberately: EM-3 (P1 existing values, P10a `pending`) · EM-7 (P0 fix, and P0's 0.3b absorbing the former 2.8) · EM-16 (P1 hardening, P2 derivation, P4 the trace to join) · RE-1 (P6 `async`, P10b `blocking`) · AX-6 (P7 API, P8 UI) · AX-2a (P7 contract, P8 presentation) · DO-1 (P2 schema, P3 attributes) · SE-3 (P5a begins, P6 completes) · AT-9 (specified P3, delivered P7) · RE-15b (semantics P10a, affordance P8) · OP-1 (P0 toggle, P0 `policies`) · DO-4 (P5a in-cluster guides, P5b external example) · TR-3 (P5a single filter, P5b generalised) · RE-15/RE-15a (P10a detection, referenced by P10b).

**Track 1 covers 85 of the 122 requirement IDs; 37 defer to Track 2.** Appendix C lists all 37 and records the one Must that Track 1 leaves only partially met.

**Requiring no work:** RE-2, ST-1, SE-1, SE-2 — all ✅ trivially, three of them because nothing is wired. **Removed:** OP-4. **Not covered by any scope-doc requirement:** P4 in its entirety, plus tasks 7.22 (Observer internal-port auth) and 4.12 (stale comment) — recorded as adjacent findings rather than smuggled in as audit requirements.

## Appendix B — Corrections to the scope document

Amendments arising from verification against `main`. The first is material enough to warrant a scope-doc revision before design review.

1. **§1 is describing a different state than this repository.** The framework is not "in progress in PR #1465" — it was merged and then **unwired by `a64ea04d` (#2588)**, and is dead code today. **Every ◐ in the status column that assumes emission is happening should be ○.** Specifically: PR-1, PR-9, EM-2, EM-3, EM-6, AT-1, TR-2, TR-5 and ST-4 are ○, not ◐; and **PR-2 and PR-6 are ○, not ✅** — declared but unreachable, with all six PR-2 patterns stale besides.
2. **The `component_workflows.go` defect no longer exists.** The file was deleted by `c30c5a4e` and the helper by `a64ea04d`. It was real in PR #1465. Keep it as a review rule for P0, since ~18 call sites shared the shape, but remove it from the defect list.
3. **PR-4 and EM-11 understate the work — and the fix is not what this document first proposed.** *"MCP wraps the API"* holds at the service layer but **not at the HTTP layer**: `mcphandlers` calls services directly, so route-pattern middleware cannot see MCP calls. **Corrected 2026-08-13:** PR-4 does **not** need service-layer emission. A protocol-level MCP receiving middleware is the right hook — it sees the tool name, the raw arguments, the per-request headers and the authz filter's denials, needs no service-layer plumbing, and cannot double-count. What it does need is a **declared 69-entry binding table**, because the tool↔operation mapping is not derivable: 21 tools fan out to two operations via an MCP-only `scope` argument, 10 collapse two tools onto one operation, and `create_component_release` targets `generateRelease`.

4. **Appendix item 1 — resolved.** The query abstraction does extend to a second event type, but **by duplication**: `EventsQueryRequest` copies the logs envelope with no shared base. Neither existing `searchScope` fits audit (both are component-centric and UID-based on the adapter side), and there is no real pagination. AX-1 is achievable but is more work than "reuse the existing abstraction" implies.
5. **Appendix item 2 — outstanding, and now scheduled.** No OpenTelemetry span instrumentation exists in `openchoreo-api`. EM-16's *"join the trace that already exists"* presumes tracing that is absent; **P4 creates it**. Note also that `request_id` is currently client-spoofable via an unvalidated inbound `X-Request-ID`, which matters once it becomes the trace correlation ID.
6. **Appendix item 3 — resolved.** Modules are per-signal OpenAPI adapter contracts consumed through generated clients. But adapter URLs are **flat static Observer config**, nothing reads `ObservabilityPlane` to resolve them, there is **no module registration or capability-discovery mechanism** beyond HTTP 501, and **the adapter contract has no security scheme at all** — Observer forwards no token and is the sole trust boundary. ST-2 reuses the mechanism cleanly; audit configuration does not belong in the plane CRD.
7. **A new §3 exclusion is needed, or a new requirement.** `/exec/` cannot be audited with the current `responseWriter` (no `Hijacker`/`Flusher`), and `component:exec` is the most sensitive action in the API. Either commit to auditing it by another mechanism or exclude it explicitly.
8. **EM-6.** `Actor.Type`'s comment enumerates `anonymous` but not `agent`. Add `agent`, or document `anonymous` as a fourth value.
9. **§4.8's recommendation is a change, not a default.** The existing dispatcher drops **newest** on a full queue. Choosing drop-oldest is a deliberate reversal.
10. **The companion design doc** `openchoreo-audit-logging-design.md`, which §1's header instructs readers to read first, **does not exist in this repository.** Either add it or drop the reference.

11. **EM-4a refined: `category` is a stamped label, not a policy key.** It stays a closed ~~three~~ **two**-value set — `management` / `authorization`, corrected 2026-08-24; the third value `access` was proposed and reversed on 2026-08-19, because a rejection is an *outcome* rather than a kind of operation, and it is now `result: unauthenticated` — assigned per operation from its resource kind, and **not operator-configurable**. Operator flexibility lives in an ordered `policies` list with event-field selectors, first match wins. Reason: category is immutable in published records and is what backends key retention on, so reclassifying splits history — events already indexed silently keep the old retention. DO-4 and DO-6 should therefore steer durable retention and SIEM rules onto `resource.type` and `action`, which are intrinsic and stable across any config history.

12. **SE-2 strengthened in practice.** Suppression rules may not key on `actors` or `entitlements`, because a subject able to influence its own group membership could otherwise suppress its own auditing. Escalation rules may, since a wrong match there yields *more* auditing.

13. **A new §3 exclusion:** `source_ip` is unobtainable for direct (non-proxied) MCP clients — `RequestExtra` carries no `RemoteAddr`.

14. **A new §3 exclusion:** a 400 from malformed path-parameter binding never reaches audit, because the generated wrapper binds parameters before any middleware runs.

## Appendix C — Deferred requirement register (2026-08-10 revision)

The 37 requirement IDs that move to Track 2. **Deferred is not descoped** — every one keeps its phase, its deliverables and its exit criteria in §4. The scope doc's status column should mark these **deferred** rather than ○, so a reader can tell a scheduling decision from an oversight.

| Group | Deferred IDs | Count | Lands in | Priority mix |
|---|---|---|---|---|
| CP | CP-1 … CP-9 | 9 | P9 | 5 M, 4 S |
| AT | AT-12 | 1 | P9 | S |
| TR | TR-1, TR-3, TR-4, TR-6, TR-8 | 5 | P5b, P6 (TR-6) | 4 M, 1 S |
| RE | RE-1, RE-3, RE-4, RE-5, RE-6, RE-8 … RE-14a, RE-16, RE-17, RE-18 *(**not** RE-7 — see below)* | 16 | P6, P10b | 13 M, 3 S |
| SE | SE-4, SE-5 | 2 | P5b, P6 | 2 M |
| OP | OP-2, OP-7, OP-8 | 3 | P5b, P10b | 3 M |
| DO | DO-3 | 1 | P9 | M |

Track 1 retains **RE-7** (in P2, task 2.18) and RE-15, RE-15a and RE-15b (in P10a); RE-2 needs no work. TR-2, TR-5 and TR-7 stay in P5a.

### SE-5 — partially met in Track 1, and cheaper to close than first assessed

**SE-5 (Must)** — *"Audit gaps are detectable: an operator can distinguish 'nothing happened' from 'we stopped recording'."* The scope doc singles this out as the requirement the whole reliability posture is traded for: *"the auditor's real question is not 'did you lose an event?' but 'would you know if you had?'"*

> **Corrected 2026-08-11.** An earlier draft of this appendix recorded SE-5 as wholly unmet in Track 1, on the grounds that RE-7's sequence numbers were deferred with P6. That was wrong. **Proposal [#4396](https://github.com/openchoreo/openchoreo/discussions/4396) lists `sequence` in the core event schema table**, not among the reliability machinery — *"Counter within `producer` + `producer_instance`, so a consumer can tell a missing event from no event."* It belongs in **P2** with the rest of the envelope, and P2 is Track 1. Task **2.18** now carries it, and task 6.5 in P6 reduces to the surrounding metrics.

**What Track 1 provides:**

| Assurance | Mechanism | Covers |
|---|---|---|
| A missing event is distinguishable from no event | `sequence` in the P2 envelope (task 2.18) | A consumer receiving 41, 42, 44 can prove an event was emitted and lost |
| An action started and never finished | P10a's orphaned `pending` events | A producer that crashed mid-action — **the case sequence numbers structurally cannot detect**, since no number was ever allocated |
| Records reached the collector | Agent-side buffering and backpressure | The shipping leg, including a sink outage |

**What remains deferred with P6** is not the detection itself but the **attribution and the alarm**: the drop counter that says *why* an event is missing (overflow versus pipeline loss), the per-sink delivery metrics, the "audit degraded" condition (RE-9) that tells an operator publishing has started failing without their reading metrics, and the heartbeat (RE-10) that separates a quiet period from a dead producer.

So an operator can prove after the fact that the trail has a hole, but gets no live signal that one is forming. **Record SE-5 as partially met**, with RE-8, RE-9 and RE-10 named as the remainder — not as wholly unmet, and not as satisfied.
