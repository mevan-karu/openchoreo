# Audit Logs Module — Complete Scope and Requirements

**Companion to:** [`openchoreo-audit-logging-design.md`](./openchoreo-audit-logging-design.md) — read that first for the problem, goals and high-level solution.
**Epic:** [#3343](https://github.com/openchoreo/openchoreo/issues/3343) · **Proposal:** [Discussion #1403](https://github.com/openchoreo/openchoreo/discussions/1403) · **Tracking:** [#1380](https://github.com/openchoreo/openchoreo/issues/1380) · **In progress:** [PR #1465](https://github.com/openchoreo/openchoreo/pull/1465)

**Status:** Draft for design review · **Date:** 2026-07-30

> This document holds the complete scope of the audit logging feature in one place — current state, every requirement, what is deliberately excluded, and the decisions still open. It does not propose implementation phasing; that follows once the design is agreed.
>
> **Status column:** ✅ done · ◐ partial · ○ not started — assessed against PR #1465 as reviewed on 2026-07-30.
> **Priority:** **M**ust · **S**hould · **C**ould

---

## 1. Current state

PR #1465 adds a reusable audit framework at `internal/server/middleware/audit/` and wires it into `openchoreo-api`.

**What it provides**

- `types.go` — an `Event` with `event_id` (UUID v7), `timestamp`, `actor` (type, ID, entitlements), `action`, `category`, `resource` (type, ID, name), `result`, `request_id`, `source_ip`, `service` and free-form `metadata`. Categories are `resource`, `auth`, `observability`; results are `success`, `failure`, `denied`. Note that `observability` is defined but unused — every entry in `definitions.go` is `resource` or `auth`.
- `resolver.go` — maps an HTTP method and route pattern to an action definition.
- `middleware.go` — resolves the action, extracts the actor from the auth context, wraps the response writer to capture the status, and emits one event per request. Status maps to result: 2xx → success, 401/403 → denied, otherwise failure.
- `context.go` — `SetResource`, `AddMetadata`, `AddMetadataBatch` for handlers to supply resource and contextual detail.
- `logger.go` — emits through `slog` as an `AUDIT-LOG` entry with grouped attributes.
- `definitions.go` — around 22 action definitions for `openchoreo-api`, covering resource operations and authorization role and role-mapping changes.
- `auth` — `SubjectContext` gains an `ID`, populated from the JWT `sub` claim.

**Known gaps in the current implementation**

| Gap | Detail |
|---|---|
| MCP routes not audited | The audit middleware is applied to the `api` route group but not the `mcp` group |
| Auth failures not audited | JWT middleware runs before the audit middleware, so rejected tokens never reach it |
| Unmatched routes skipped silently | A route with no action definition is passed through with no warning and no build failure |
| Tenancy in free-form metadata | Organization, project and component are metadata keys, set inconsistently across handlers |
| No schema version | The event carries no version field for downstream consumers |
| Shared with application logs | Same logger, same output, same destination and retention |
| Single transport | `slog` only; no webhook or alternative sink |
| No read path | No query API, no UI, no access control over audit data |

Two implementation defects worth fixing independently of this scope:

1. `component_workflows.go` — `setAuditResource(ctx, "component_workflow_run", workflowRun.Name, …)` is called before the `err != nil` check, so a failed `TriggerWorkflow` returning `(nil, err)` will panic.
2. `middleware.go` — `extractActor` only assigns `actorID` when `len(subjectCtx.EntitlementValues) > 0`, so a valid token with no entitlements yields `actor.id = "unknown"` despite `subjectCtx.ID` holding the `sub` claim.

---

## 2. Requirements

### 2.1 Event model and schema (EM)

| ID | Requirement | Pri | Status |
|---|---|---|---|
| EM-1 | A single canonical audit event schema is shared by all producers — OpenChoreo services, MCP servers and clients. | M | ◐ |
| EM-2 | Core fields: `event_id` (UUID v7), `timestamp`, `actor`, `action`, `category`, `resource`, `result`, `request_id`, `source_ip` and the emitting component. The last is named `service` in PR #1465 and is **superseded by `producer`** (EM-10), since clients emit events too. | M | ◐ |
| EM-3 | `result` ∈ `{pending, success, failure, denied}`. Failed and denied attempts are recorded, not only successes. `pending` is added by the two-event model (EM-13b) and is not in PR #1465. | M | ◐ |
| EM-4 | The envelope carries an explicit `schema_version` so consumers and external systems can pin a version. | M | ○ |
| EM-4a | **`category` is a policy grouping, not a subsystem label.** It is the key operators configure enablement, delivery mode and suppression against, and the key a downstream system can apply its own retention against, so its values must divide events by how they should be *treated*, not by which component produced them. The set is `resource` (platform resource changes), `permission` (roles, role mappings, permission grants) and `access` (rejections at the boundary that never resolve to an action). This supersedes the `resource` / `auth` / `observability` set in PR #1465, where `authz` and `access` would have been near-synonyms and `observability` mixed a subsystem label into a policy axis. | M | ◐ |
| EM-4b | A denied attempt at a resolved action stays in that action's own category with `result: denied`. `access` is reserved for rejections with no action to attribute, such as an expired or malformed token. Without this rule the two overlap and per-category policy becomes unpredictable. | M | ○ |
| EM-4c | The `observability` category defined in PR #1465 is **dropped** until something emits it. No action definition currently uses it, and the question of whether Observer reads are audited is still open (§4.3). Reintroduce it then if needed. | S | ◐ |
| EM-5 | Tenancy — `org`, `project`, `component`, `environment` — are **first-class fields**, not metadata keys. Every meaningful query and every access decision filters on these. | M | ○ |
| EM-6 | `actor.type` distinguishes `user`, `service_account` and `agent` (MCP/AI). | M | ◐ |
| EM-7 | Actor identity is taken from the validated token's `sub` claim and does not depend on any other claim being present. | M | ◐ |
| EM-8 | Identity claims `actor.subject` (`sub`), `actor.session_id` (`sid`) and `actor.issuer` (`iss`) are recorded, so events can be matched against the identity provider's own login records. | M | ○ |
| EM-9 | `actor.on_behalf_of` records delegation — an agent or client acting for a user. | S | ○ |
| EM-10 | `producer` identifies the emitting **component**, kept distinct from `actor`. `producer_instance` identifies which running instance, so a restart is not mistaken for a sequence gap (RE-7). | M | ○ |
| EM-10a | `producer_instance` is a UUID generated at process start and held in memory beside the sequence counter, so the two reset together. The Kubernetes downward API (`metadata.name`, `metadata.uid`) is an alternative for in-cluster services but does not cover external producers such as the Backstage portal, and a container restart within the same pod would reuse the pod UID while resetting an in-memory counter. Pod name may be recorded alongside for troubleshooting. | M | ○ |
| EM-11 | `origin` identifies the **interface the call arrived through**: `api`, `mcp`, `client`. **MCP wraps the API rather than sitting beside it**, so an MCP-driven action is audited once, at the API layer, and `origin` must be carried down from the MCP entry point to where the event is built. `producer` alone cannot distinguish the two, since both report `openchoreo-api`, and `actor.type` cannot either, because a human can drive MCP just as an agent can drive REST. | M | ○ |
| EM-12 | `actor.assurance` ∈ `{validated, asserted}` records whether OpenChoreo verified the actor identity or an external producer asserted it. See §4.1. | M | ○ |
| EM-13 | Both `event_time` (per the producer) and `received_time` (when the pipeline accepted it) are recorded. **Justified specifically by client producers and durable buffering** — a client's clock may be skewed or back-dated, and an event buffered through a sink outage can arrive hours after it occurred. For in-process OpenChoreo events the two will be near-identical, which is expected. | M | ○ |
| EM-13a | An `audit_id` correlates the events of a single action, distinct from the per-event `event_id`. This mirrors Kubernetes' `auditID`, shared across the `RequestReceived` / `ResponseComplete` events of one request. `request_id` cannot serve this purpose — client-produced events have no HTTP request. | M | ○ |
| EM-13b | `result` is `pending` on the pre-action event and `success` / `failure` / `denied` on the completion event. **No separate `stage` field is required** — `pending` identifies the pre-action event unambiguously. Introduce an explicit stage only if a third phase is ever added. | M | ○ |
| EM-13c | A single-phase action emits one event whose `audit_id` has no sibling. Consumers must not assume events come in pairs. | M | ○ |
| EM-14 | `resource.uid` accompanies `resource.id`, so a delete-then-recreate is distinguishable from an update. | S | ○ |
| EM-15 | Optional `reason` free-text, for break-glass operations. | C | ○ |
| EM-16 | **`request_id` doubles as the trace correlation ID; no separate `trace_id` field.** A UUID is 128 bits, the same width W3C Trace Context requires, and the mapping is 1:1 — one inbound call is one trace at our boundary. Where an incoming request carries a `traceparent` header, `request_id` is taken from its trace-id portion so the audit event joins the trace that already exists; otherwise it is generated as today. Generating our own regardless would give one logical operation two identifiers and break that join. | M | ○ |
| EM-17 | A documented field mapping from the OpenChoreo schema to OCSF `api_activity`, easing onboarding to external SIEM systems. | S | ○ |

### 2.2 Object attributes (AT)

*Different objects need different extra detail. Today each handler adds this as free-form metadata in its own way — `source_environment` / `target_environment` for a promotion, `role` / `entitlement` / `effect` / `hierarchy` / `context` for a role mapping, `commit` for a workflow run. This group makes that structured, declared and dependable.*

| ID | Requirement | Pri | Status |
|---|---|---|---|
| AT-1 | Each object type can define its own set of additional attributes, carried alongside the core event fields. | M | ◐ |
| AT-2 | Attribute sets are **declared in a registry per object type**, in the same way routes are mapped to actions today — not decided ad hoc by individual handlers. | M | ○ |
| AT-3 | Adding a new object type or a new attribute **does not change the core event schema**. | M | ○ |
| AT-4 | Attributes carry declared types (string, number, boolean, timestamp, list) so backends can index them correctly and consumers can parse them safely. | M | ○ |
| AT-5 | Declared attributes are documented and discoverable — a consumer can tell which attributes to expect for a given action or object type. | M | ○ |
| AT-6 | Attributes are validated against their declaration before publishing. Undeclared or wrongly-typed attributes are rejected or flagged, never silently stored. | M | ○ |
| AT-7 | If an attribute is declared as required for an action, a check catches any code path that fails to set it. | S | ○ |
| AT-8 | Attribute names follow a consistent convention across services and object types, so the same concept is not named three ways. | M | ○ |
| AT-9 | Attributes are filterable through the read API, not merely stored. | M | ○ |
| AT-10 | The attribute declaration acts as an **allowlist**: only declared attributes are emitted, which prevents secrets or unbounded payloads leaking in through metadata. | M | ○ |
| AT-11 | Attribute sets are versioned with the event schema, so they can evolve without breaking existing consumers. | S | ○ |
| AT-12 | Client producers can declare and publish attributes for their own object types, under the same rules. | S | ○ |
| AT-13 | Growth in the number of distinct attribute fields is bounded and documented. *Dynamically-created fields can cause mapping problems in index-based backends; the design should state a limit or a namespacing strategy.* | S | ○ |

### 2.3 Production in OpenChoreo services (PR)

| ID | Requirement | Pri | Status |
|---|---|---|---|
| PR-1 | Every state-modifying operation on OpenChoreo HTTP APIs emits exactly one audit event. | M | ◐ |
| PR-2 | Authorization changes — role create/update/delete, role-mapping changes, permission grants — are audited under the `permission` category. | M | ✅ |
| PR-3 | **A CI check fails the build if any state-modifying route has no audit action definition.** Without this, coverage silently falls behind as the API grows. | M | ○ |
| PR-4 | Actions driven through the MCP interfaces are audited. Since MCP wraps the API, this is a matter of routing MCP-served requests through the same middleware and carrying `origin: mcp` through, not a second set of action definitions. | M | ○ |
| PR-5 | The Observer API produces audit events. *Scope depends on §4.3.* | S | ○ |
| PR-6 | Authorization denials are audited, under the denied action's own category with `result: denied` (EM-4b). | M | ✅ |
| PR-7 | Token-validation rejections at the API boundary are audited under the `access` category, since no action resolves. *Requires the middleware ordering to change so rejected requests reach the audit middleware.* | M | ○ |
| PR-8 | Adding audit coverage to a new service requires only: declare actions and attributes, initialise the logger, add the middleware. | M | ✅ |
| PR-9 | Audit emission behaviour follows the configured delivery mode for the category — see group RE. In the default `async` mode it never blocks or fails a request. | M | ◐ |

### 2.4 Client producers (CP)

| ID | Requirement | Pri | Status |
|---|---|---|---|
| CP-1 | A documented, versioned ingestion contract lets producers outside the OpenChoreo backend publish audit events into the same stream. | M | ○ |
| CP-2 | A documented mapping between Backstage's Auditor Service event shape (`eventId`, `subEventId`, `meta`, `severityLevel`) and the OpenChoreo schema. | M | ○ |
| CP-3 | Client-only actions that never reach an OpenChoreo API — scaffolder template runs, catalog changes, plugin actions — are auditable. | M | ○ |
| CP-4 | The ingestion path is authenticated, and producer identity is established by OpenChoreo rather than taken from the payload. | M | ○ |
| CP-5 | **Producers cannot forge actor identity.** Either the client forwards the end-user token for OpenChoreo to validate, or the event is marked `actor.assurance: asserted`. | M | ○ |
| CP-6 | Ingested events are schema-validated on entry; non-conforming events are rejected with actionable errors, never silently dropped or partially stored. | M | ○ |
| CP-7 | Ingestion is rate-limited per producer and rejects implausible `event_time` values while preserving `received_time`. | S | ○ |
| CP-8 | `event_id` is handled idempotently, so producer retries do not create duplicates. | S | ○ |
| CP-9 | Producer onboarding is documented well enough for a third party to integrate a new client without reading OpenChoreo source. | S | ○ |

### 2.5 Publishing and transport (TR)

| ID | Requirement | Pri | Status |
|---|---|---|---|
| TR-1 | **Webhook transport** — audit events publishable as events to an operator-configured endpoint, so OpenChoreo fits into an audit or SIEM system the user already runs. | M | ○ |
| TR-2 | **Log collection transport** — audit events publishable through the existing collecting agent to the relevant collector, reusing the current pipeline. | M | ◐ |
| TR-3 | Transports are independently configurable and can run at the same time, driven by **one definition of what gets audited** — no per-sink duplication of filtering logic. | M | ○ |
| TR-4 | Transport is an extension point: adding a sink requires no change to producers or to the event model. | S | ○ |
| TR-5 | In the default `async` mode, publishing never blocks the request path — a slow or failing sink degrades audit completeness, never request latency or availability. Operators can opt into `blocking` per category (RE-1, RE-12), accepting the availability trade knowingly. | M | ◐ |
| TR-6 | Webhook delivery has defined reliability semantics — see group RE. The log path inherits Fluent Bit's filesystem buffering and backpressure handling; the webhook path needs an equivalent durable queue, since it has none today. | M | ○ |
| TR-7 | Audit events travel on a **stream separate from application and access logs**, so retention and access rules can differ. | M | ○ |
| TR-8 | Transport health is observable: per-sink success and failure counts, queue depth, drop counts. | M | ○ |

### 2.6 Reliability and delivery guarantees (RE)

*The central question: if an audit event cannot be published, what happens to the action? See §4.6 for the vendor comparison behind these requirements.*

| ID | Requirement | Pri | Status |
|---|---|---|---|
| RE-1 | **The delivery guarantee is an operator-configurable mode, not a fixed property of the system** — following the Kubernetes model. At minimum: `async` (never blocks; durable queue guarantees eventual delivery) and `blocking` (the request is not completed until the event is published). `async` is the default. | M | ○ |
| RE-2 | **An action is never reverted because its audit event failed to publish.** Where an action must not proceed unaudited, it is **failed up front** (RE-13), not undone afterwards. A revert is itself an unaudited state change and converts a logging failure into a data-integrity failure. | M | ✅ |
| RE-3 | **In `async` mode**, events are held in a bounded in-process queue so a slow or failing sink applies backpressure to the buffer, never to the caller. | M | ○ |
| RE-4 | The queue is **backed by local disk**, so a service restart or a sink outage does not lose events already accepted. Events are resumed on start-up. This internal queue is what makes `async` a delivery *guarantee* rather than best-effort. | M | ○ |
| RE-5 | Delivery is retried with backoff. The retry count and backoff are configurable, and the same retry configuration governs both modes — in `async` it delays delivery, in `blocking` it determines when the action fails (RE-13). | M | ○ |
| RE-6 | Overflow behaviour is **explicit, configurable and counted** — never an unremarked drop. See §4.7 for the choice of policy. | M | ○ |
| RE-7 | Each producer instance numbers its events in sequence, so a consumer receiving 41, 42, 44 can **detect that an event was emitted and lost** rather than having to trust that none was. Scoped to `producer` + `producer_instance` (EM-10), because a pod restart resets the counter and would otherwise look like a catastrophic gap. **It detects loss only after emission** — a service that dies before emitting allocates no number, so nothing appears missing. That blind spot is the main argument for enabling the pre-action event beyond blocking categories (§4.7). | M | ○ |
| RE-8 | Delivery health is exposed as metrics: queue depth, retry counts, drop counts, per-sink success and failure, and time since last successful delivery. | M | ○ |
| RE-9 | An **"audit degraded" condition** is surfaced when publishing is failing or the buffer is above a threshold, so the problem is visible without inspecting metrics. For categories in `blocking` mode this is an early warning that operations are about to start failing. | M | ○ |
| RE-10 | Periodic heartbeat events are emitted, so a silent period can be distinguished from a producer or pipeline outage. | S | ○ |
| RE-11 | Duplicate delivery is acceptable and expected; consumers deduplicate on `event_id` (with CP-8). At-least-once is the delivery contract, not exactly-once. | M | ○ |
| RE-12 | **The mode is configurable per category**, not only globally. This lets an operator block on high-value low-volume events such as authorization changes while routine operations stay asynchronous — bounding the availability risk rather than applying it platform-wide. | M | ○ |
| RE-13 | **In `blocking` mode, if publishing still fails after the configured retries, the action fails.** The caller receives an error and the operation does not succeed unaudited. | M | ○ |
| RE-14 | **In `blocking` mode the event is published before the action is performed** (write-ahead), so a failure means the action genuinely did not happen rather than returning an error for something that did. Kubernetes does exactly this — `blocking-strict` fails only at the RequestReceived stage, before the request is delegated down the handler chain, leaving state unchanged. This is what makes RE-13 honest to the caller and RE-2 coherent. The outcome is recorded in a second event once the action completes. | M | ○ |
| RE-14a | **If publishing fails *after* the action has been applied, the action is not undone** — in any mode. The failure is recorded as a metric and surfaced through RE-8 and RE-9, matching Kubernetes' behaviour at post-RequestReceived stages. Blocking mode narrows this window; it does not eliminate it. | M | ○ |
| RE-15 | Write-ahead recording roughly doubles event volume for blocking categories, adds latency, and requires reconciling actions left in a pending state — actions whose intent was recorded but whose outcome never was. The reconciliation behaviour must be defined. | M | ○ |
| RE-15a | **Whether the pre-action event is emitted is configurable per category**, independently of the delivery mode. Kubernetes operators commonly omit `RequestReceived` for exactly this volume reason. Blocking categories require it (RE-14); others may enable it purely to make interrupted actions detectable. See §4.7. | M | ○ |
| RE-15b | A mid-action crash leaves a `pending` event with no completion. This is a **feature** — it converts silence into a detectable "started, never finished" — but the read path and any reconciliation must handle orphaned pending events explicitly rather than presenting them as completed actions. | M | ○ |
| RE-16 | Buffer sizing, disk usage and the resulting retention window under sink outage are documented, so operators can size the spool for their expected outage tolerance. | S | ○ |
| RE-17 | The availability implication of `blocking` mode is documented plainly: enabling it for a category means an audit outage becomes an outage for those operations. Operators should be able to make that trade knowingly. | M | ○ |
| RE-18 | Reliability behaviour is testable: the design admits fault injection for sink outage, slow sink, buffer overflow, restart-with-pending-events, and blocking-mode retry exhaustion. | S | ○ |

### 2.7 Storage and analysis (ST)

| ID | Requirement | Pri | Status |
|---|---|---|---|
| ST-1 | **OpenChoreo does not store or analyse audit logs.** Storage is handled by the configured observability module or the user's own system. | M | ✅ |
| ST-2 | Audit logs are stored and analysed through the **observability module architecture**, the same way logs, metrics and traces are — no new infrastructure and no new deployment model. | M | ○ |
| ST-3 | The backend is pluggable and substitutable. Neither the event model nor the read path may assume a specific vendor. | M | ○ |
| ST-4 | The default deployment uses the existing observability module, so the out-of-box experience needs nothing extra. | M | ◐ |
| ST-5 | **Retention is out of OpenChoreo's hands.** It is set in the observability module or the external system holding the events, using that system's own tooling. OpenChoreo's obligation is TR-7: keeping audit events on their own stream so a different policy *can* be applied to them. | M | ○ |
| ST-6 | A **forward-only mode** is supported: publish to the configured sink and retain nothing in OpenChoreo-managed storage. *Note the interaction with AX-1 in §4.2.* | S | ○ |
| ST-7 | A documented conformance baseline states the minimum a module must support — time-range query, field filtering, attribute filtering, pagination — so operators know whether their chosen backend will satisfy the read path. | M | ○ |
| ST-8 | Volume and sizing guidance is documented per event category so operators can capacity-plan. | S | ○ |

### 2.8 Access, query and viewing (AX)

| ID | Requirement | Pri | Status |
|---|---|---|---|
| AX-1 | **The read API is part of the Observer API**, which already abstracts queries over the configured observability module. No separate service. | M | ○ |
| AX-2 | Query supports filtering by time range, actor, action, category, resource, result, producer, tenancy and **declared object attributes**, with pagination and stable ordering. | M | ○ |
| AX-2a | The read path presents the events of one action as a single logical entry by default, collapsing on `audit_id`, with the individual events available on request. Showing two rows for one promotion would confuse a portal user. | S | ○ |
| AX-3 | **OpenChoreo clients — the Backstage portal first — can present audit logs through this API.** Holders of the AX-4 permission must not need backend or vendor-console access to see the audit trail. | M | ○ |
| AX-4 | **Reading audit logs requires a dedicated permission**, granted to administrator, platform engineer and security administrator roles only. It is separate from any permission over the resources themselves: working in a project does not confer the right to read its audit trail. Audit records disclose activity across teams, so this is a sensitive privilege in its own right. | M | ○ |
| AX-5 | Every audit-log read is itself recorded — who viewed which records, when. | S | ○ |
| AX-6 | Export from the API and clients in JSON and CSV. | S | ○ |
| AX-7 | The API degrades intelligibly when the backend is unavailable or lacks a capability — a clear error, never a partial result presented as complete. | M | ○ |
| AX-8 | The read API is versioned independently of the event schema, so schema changes do not break client integrations. | S | ○ |

### 2.9 Security and privacy (SE)

| ID | Requirement | Pri | Status |
|---|---|---|---|
| SE-1 | **No request or response bodies are logged.** Detail is captured only through declared attributes (AT-10). Secrets, tokens and credentials must never appear in an audit event. | M | ✅ |
| SE-2 | Producers cannot suppress their own audit events; what is audited is governed by operator configuration, not handler-level opt-out. | M | ✅ |
| SE-3 | Failure semantics are explicit and documented. See group RE for the full model — fail-open by default, with an optional stricter mode scoped per category. | M | ○ |
| SE-4 | Events in transit are protected — TLS, webhook endpoint authentication, and optionally payload signing so consumers can verify origin. | M | ○ |
| SE-5 | Audit gaps are detectable: an operator can distinguish "nothing happened" from "we stopped recording". Delivered through RE-7 to RE-10. **This is what we trade blocking for — the auditor's real question is not "did you lose an event?" but "would you know if you had?"** | M | ○ |
| SE-6 | PII handling is documented, including which fields may contain personal data (`actor.id`, `actor.subject`, `source_ip`) and how operators meet data-subject obligations against an external backend. | S | ○ |
| SE-7 | Configurable redaction or hashing for designated sensitive attributes. | C | ○ |

### 2.10 Operability (OP)

| ID | Requirement | Pri | Status |
|---|---|---|---|
| OP-1 | Enablement configurable per event category, not only globally. | S | ○ |
| OP-2 | Configurable forward target or targets. | M | ○ |
| OP-3 | Per-action suppression, so operators can silence high-volume low-value actions. | S | ○ |
| OP-4 | *Removed. Retention is configured in the module or external system, not in OpenChoreo — see ST-5.* | — | — |
| OP-5 | Configuration is declarative and consistent with existing OpenChoreo patterns, and should not require a restart where avoidable. | S | ○ |
| OP-8 | The delivery mode (`async` / `blocking`), retry count and backoff are operator-configurable per category, alongside the other controls in this group. | M | ○ |
| OP-6 | **The storage and viewing module is optional at install time; production and publishing are always active.** With no module installed, events still reach the configured sink. | M | ○ |
| OP-7 | Adding a producer, a sink, or a backend module are three independent extension points. | M | ○ |

### 2.11 Documentation (DO)

| ID | Requirement | Pri | Status |
|---|---|---|---|
| DO-1 | Schema and attribute reference generated from the declarations, not hand-maintained. | M | ○ |
| DO-2 | A coverage matrix stating which actions on which surfaces are audited — and explicitly, which are not. | M | ○ |
| DO-3 | Producer integration guide covering both OpenChoreo services and external clients. | M | ○ |
| DO-4 | Transport and module configuration guides, including a worked example of publishing to an external audit system. | M | ○ |
| DO-5 | A guide to matching OpenChoreo audit events against identity provider login records, stating plainly that OpenChoreo does not produce authentication events. | M | ○ |
| DO-6 | Compliance guidance mapping capabilities to the evidence auditors commonly ask for, and what remains the operator's responsibility. *Exact control identifiers to be confirmed with a compliance reviewer.* | S | ○ |

---

## 3. Out of scope

- **Storing or analysing audit logs in OpenChoreo.** We produce and publish; the configured module or the user's own system does the rest.
- **Retention.** How long audit logs are kept is set in the module or external system holding them, not in OpenChoreo (ST-5). Our obligation is the separate stream (TR-7) that makes a distinct policy possible.
- **Changes made outside OpenChoreo's APIs** — direct `kubectl` edits, GitOps changes, other controllers. These are already covered by the Kubernetes API server's own audit logging. Because our publishing is pluggable, users can route both into the same system.
- **Authentication events.** Logins, logouts, token issuance and refresh happen directly between client and identity provider. OpenChoreo is a resource server and never observes them. We audit what is visible at our boundary — denied requests and rejected tokens — and make our records joinable to the IdP's (EM-8, DO-5).
- **Auditing read requests on the control-plane API.** Volume is high, value low. The Observer API is a possible exception — §4.3.
- **Reconciliation and controller activity.** Out of scope alongside out-of-band changes.
- **Cryptographic tamper-evidence** — hash chaining, signing, write-once storage. A property of the backend the operator chooses.
- **Full request and response body capture.** Excluded on secret-leakage and volume grounds.
- **Alerting and anomaly detection.** Belongs with the observability alerts work ([#1024](https://github.com/openchoreo/openchoreo/discussions/1024)).
- **Being a compliance product.** We provide the trail; we do not certify.

---

## 4. Open decisions

### 4.1 Trust model for client-produced events

Accepting events from producers we do not control means a client could claim any actor. The proposed primitives are: separate `producer` from `actor` (EM-10), authenticate the producer independently of the payload (CP-4), and record `actor.assurance` as `validated` or `asserted` (EM-12).

**To decide:** must clients forward the end-user token for OpenChoreo-side validation, or is `asserted` acceptable for client-only actions? Are asserted events visibly distinguished in the portal so a reader is never misled about attribution strength?

### 4.2 Read path over a store we do not own

**Decided:** the read API is part of the Observer API (AX-1), and reading requires a dedicated permission held by administrator, platform engineer and security administrator roles (AX-4).

**Still to decide:** how thin can the module conformance baseline (ST-7) safely be? And in forward-only mode (ST-6), where nothing is retained in an OpenChoreo-managed module, the read API has nothing to serve — **ST-6 and AX-1/AX-3 cannot both hold in that configuration**, and the resolution should be recorded rather than discovered later.

### 4.3 Observer API reads

PR-5 asks the Observer API to produce audit events, but Observer is read-mostly and read auditing is out of scope. Reading production logs and traces is arguably worth recording — it is what data exfiltration looks like. Decide explicitly, because as drafted PR-5 is close to empty for Observer.

### 4.4 Attribute model shape

**To decide:** are attributes a typed Go structure per object type, a declarative schema consulted at runtime, or both? Typed structures give compile-time safety for OpenChoreo services; a declarative schema is needed anyway for client producers (AT-12) and for generated documentation (DO-1). How the two relate is the core design question for this group.

### 4.6 Reliability posture — the vendor evidence

Four postures exist, in increasing strictness:

| Posture | Behaviour | Who does this |
|---|---|---|
| **Best-effort** | Emit and hope. No buffer, no retry, no gap detection. | OpenChoreo today |
| **Durable-asynchronous** | Never block; buffer to disk, retry, guarantee eventual delivery, detect gaps. | Kubernetes `batch` mode; OpenTelemetry Collector `sending_queue` + `file_storage`; Fluent Bit `storage.type filesystem`; AWS CloudTrail |
| **Blocking on response** | Perform the action, but withhold the response until the audit event is written. | Kubernetes `blocking` — the default for its log backend |
| **Fail-closed** | Refuse the action **before performing it** unless it can be audited. | HashiCorp Vault; Linux auditd with `disk_full_action halt`; Kubernetes `blocking-strict` at the RequestReceived stage |

**Kubernetes** is the most useful model because it declines to take a position and makes the posture configurable per backend, with buffer sizes and throttling exposed. That is the design we follow (RE-1, RE-12) — the posture is an operator setting, not a property we fix.

Note that the last two rows are not really alternatives to each other. Durable-asynchronous and blocking are **the two modes we expose**; the queue, retry and gap-detection machinery underneath is shared, and the mode only decides what happens when retries are exhausted. In `async` the event stays queued and delivery is eventually guaranteed; in `blocking` the action fails (RE-13).

Kubernetes also emits **multiple events per request** — `RequestReceived`, `ResponseStarted` (long-running requests only), `ResponseComplete` and `Panic` — all sharing one `auditID`, with `omitStages` available to skip any of them. Our two-event model (EM-13a, EM-13b) is a narrower version of this; see §4.7 for how widely the pre-action event should be enabled.

**Vault** shows the cost of maximal strictness. It requires at least one enabled audit device to persist a log before completing a request; when all fail, Vault is effectively unavailable. HashiCorp publishes a dedicated tutorial on recovering blocked audit devices, and [issue #5361](https://github.com/hashicorp/vault/issues/5361) records a blocked socket device freezing Vault even while another device was healthy. **auditd** shows the same at the OS level — `disk_full_action` accepts `halt`, which shuts the machine down, and cloud instances have been observed stopping shortly after boot as a result.

The conclusion for OpenChoreo: strictness is a legitimate operator choice but a dangerous default, and it should be scoped narrowly (RE-12) rather than applied platform-wide.

**Why not revert (RE-2).** No comparable system reverts an action because its audit failed — and Kubernetes shows precisely how blocking mode avoids needing to.

`blocking-strict` fails the request only when audit logging fails at the **RequestReceived** stage, generated as soon as the audit handler receives the request and before it is delegated down the handler chain. Nothing has been applied at that point, so the API server's state is unchanged; that guarantee was the stated purpose of the feature when it was proposed ([kubernetes#65763](https://github.com/kubernetes/kubernetes/pull/65763)). When audit fails at a later stage, after the action has been applied, Kubernetes does **not** undo it — the event is dropped and an error is recorded against `apiserver_audit_error_total`.

Blocking mode therefore works by **refusing early, not rolling back late**. The real choice is *when to decide* — publish before acting and refuse if that fails, or act and guarantee the record eventually lands. By the time a post-hoc failure is known the action has been applied and controllers may already be reconciling it; a revert is itself an unaudited state change that could fail for the same reason, and it converts a logging failure into a data-integrity failure.

This is why RE-14 makes write-ahead a requirement of `blocking` mode rather than an optional extra. Without it, "the action fails" would mean returning an error to the caller for an action that already happened — which is worse than either honest alternative, and would leave the audit trail disagreeing with reality.

### 4.7 Is the pre-action event universal or blocking-only?

Kubernetes emits several events per request — `RequestReceived`, `ResponseStarted` (long-running requests only), `ResponseComplete` and `Panic` — sharing one `auditID`, with `omitStages` to skip any of them. Omitting `RequestReceived` is a common production setting because it halves volume.

We adopt a narrower two-event version, correlated by `audit_id` and distinguished by `result: pending` (EM-13a, EM-13b). The question is how widely the pre-action event is enabled.

- **Blocking-only.** Minimum volume. It exists solely to make RE-13 honest. Interrupted actions in asynchronous categories remain invisible.
- **Universal.** Every action is detectable even if the service dies mid-request, at roughly double the event volume and a second write on every request.
- **Configurable, defaulting to blocking-only.** Mirrors `omitStages`, and is the recommendation embedded in RE-15a — but confirm the default.

Worth weighing: the crash-detection benefit is not hypothetical. Without a pre-action event, a service that dies while performing an action produces no record at all, and no amount of gap detection will reveal it — the sequence number was never allocated (RE-7).

**Do not confuse this with OpenChoreo's long-running operations.** Deploy, promote and workflow runs look like they need a second phase, but they do not in this sense: the API call returns quickly and the real work finishes later in the controllers. Their genuine outcome is a reconciliation event, which is out of scope (§3). The two-event model here covers only the window between receiving a request and completing it.

### 4.8 Overflow policy

**To decide:** when the durable buffer is full, do we drop the oldest events, drop the newest, or begin rejecting requests? Dropping the oldest loses the start of an incident; dropping the newest loses the present; rejecting requests turns the buffer into a fail-closed mechanism by the back door. Each is defensible. It should be an operator setting with a documented default, not an accident of implementation — and whichever is chosen, RE-6 requires it be counted and RE-7 requires the gap be detectable.

### 4.9 Remaining decisions

1. **Are denied and rejected events on by default?** Low-volume and high-signal argues yes; a misconfigured client flooding token rejections argues for rate-limited deduplication rather than off-by-default.
2. **OCSF naming** — adopt `api_activity` field names outright, or keep OpenChoreo names plus the EM-17 mapping?
3. **Authentication topology per service** — the authentication non-goal assumes clients complete OIDC directly against the IdP and OpenChoreo validates bearer tokens as a resource server. If any surface is fronted by a BFF, token-exchange or session layer, that component does observe authentication events and PR-7 widens for it alone.

**Closed:** the read API lives in the Observer API (AX-1); audit read is a dedicated permission limited to administrator, platform engineer and security administrator roles, not scoped by project membership (AX-4); retention is set in the module or external system rather than by OpenChoreo (ST-5).

---

## 5. References

**OpenChoreo** — [Epic #3343](https://github.com/openchoreo/openchoreo/issues/3343) · [Discussion #1403](https://github.com/openchoreo/openchoreo/discussions/1403) · [Issue #1380](https://github.com/openchoreo/openchoreo/issues/1380) · [PR #1465](https://github.com/openchoreo/openchoreo/pull/1465) · [Architecture](https://openchoreo.dev/docs/overview/architecture/) · [Discussion #1024](https://github.com/openchoreo/openchoreo/discussions/1024)

**Reliability mechanisms** — [Kubernetes audit backends and modes](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/#batching) · [kubernetes#65763 — reject incoming requests upon audit failure](https://github.com/kubernetes/kubernetes/pull/65763) · [Vault: recover blocked audit devices](https://developer.hashicorp.com/vault/tutorials/monitoring/blocked-audit-devices) and [issue #5361](https://github.com/hashicorp/vault/issues/5361) · [auditd.conf(5)](https://man7.org/linux/man-pages/man5/auditd.conf.5.html) · [OpenTelemetry Collector resiliency](https://opentelemetry.io/docs/collector/resiliency/) and [exporterhelper](https://github.com/open-telemetry/opentelemetry-collector/blob/main/exporter/exporterhelper/README.md) · [Fluent Bit buffering and storage](https://docs.fluentbit.io/manual/administration/buffering-and-storage) and [backpressure](https://docs.fluentbit.io/manual/administration/backpressure)

**Other tools** — [Kubernetes auditing](https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/) · [Vault audit devices](https://developer.hashicorp.com/vault/docs/audit) · [Grafana auditing](https://grafana.com/docs/grafana/latest/setup-grafana/configure-security/audit-grafana/) · [Argo CD security](https://argo-cd.readthedocs.io/en/stable/operator-manual/security/) · [Backstage Auditor Service](https://backstage.io/docs/backend-system/core-services/auditor/) and [issue #23950](https://github.com/backstage/backstage/issues/23950) · [Keycloak auditing](https://docs.redhat.com/en/documentation/red_hat_build_of_keycloak/26.2/html/server_administration_guide/configuring_auditing_to_track_events) · [Harbor audit log](https://goharbor.io/docs/2.14.0/administration/audit-log/) · [Humanitec audit logs](https://developer.humanitec.com/app-humanitec-io/docs/platform-orchestrator/security/audit-logs/)

**Standards** — [OCSF `api_activity`](https://schema.ocsf.io/1.3.0/classes/api_activity)

---

## Appendix — Verification status

**Verified from source:** everything in §1 was read directly from the PR #1465 diff, including the two defects noted.

**Verified from vendor documentation:** Kubernetes audit policy levels, multi-backend behaviour, the four audit stages (`RequestReceived`, `ResponseStarted` for long-running requests only, `ResponseComplete`, `Panic`) sharing one `auditID` with `omitStages` to skip them, the `batch` / `blocking` / `blocking-strict` modes, and that `blocking-strict` fails the request only at the RequestReceived stage — before the request is delegated down the handler chain, leaving state unchanged — with later-stage failures dropping the event and recording an error rather than undoing the action; Vault's requirement that at least one audit device persist before a request completes, and the resulting unavailability when all fail; auditd's `disk_full_action` values including `halt`, and reports of cloud instances stopping as a result; the OpenTelemetry Collector's `sending_queue` with the `file_storage` extension and `retry_on_failure`; Fluent Bit's `storage.type filesystem` buffering, `storage.max_chunks_up` and backpressure handling; Argo CD's delegation to apiserver audit and Kubernetes Event retention; Backstage Auditor Service and issue #23950; Keycloak event storage, per-stream expiration and Listener SPI; Harbor's operator controls; Grafana's Enterprise-only status, exporters and scoping rule; Humanitec's API behaviour and successful-only storage; OCSF category and class structure.

**Cited but not re-verified — confirm before publishing:** Vault's HMAC treatment of sensitive values; AWS CloudTrail's log file integrity validation via digest files (its asynchronous, non-blocking delivery model *is* confirmed); the exact SOC 2 and ISO 27001/27002 control identifiers referenced in DO-6.

**Not verified against the wider codebase.** Requirements were checked against PR #1465, but the following need a reviewer with repository access: whether the Observer API's query abstraction generalises to a second event type (AX-1 now depends on this, since the read API is committed to living there); current OpenTelemetry span instrumentation in `openchoreo-api` (EM-16); and how observability modules are registered and configured today, which determines how cleanly ST-2 can reuse that mechanism.
