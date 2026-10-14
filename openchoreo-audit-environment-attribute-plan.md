# Audit Event — `resource.environment` and the Render-Path Collapse

**Companion to:** [`openchoreo-audit-schema-evaluation.md`](./openchoreo-audit-schema-evaluation.md) (§5 and §6 D1, which spun this out) · [`openchoreo-audit-logging-phased-plan.md`](./openchoreo-audit-logging-phased-plan.md) §P2 · [`openchoreo-audit-logging-scope.md`](./openchoreo-audit-logging-scope.md) (EM-5)
**Sequenced as:** a **prerequisite to P2**, not a deferral — P2 cannot freeze `schema_version: 1` with a tenancy field missing, and should not add eight fields through two hand-maintained render paths.
**Status:** Draft · **Date:** 2026-09-07 · **Branch:** `feat/audit-schema`

> **⚠️ This plan corrects its own parent.** The evaluation doc (§6 D1) concluded that
> `environment` *"has no writer available"* because both `Hierarchy` writers derive from
> `authz.ResourceHierarchy`, which lacks the field — and therefore that `environment` must live on
> `Resource`, seeded pre-handler, requiring `SetResource` to be made merging first.
>
> **That is wrong, and the correction makes this work substantially smaller.** `CheckRequest`
> carries `Context authz.Context` *alongside* `Hierarchy` (`internal/openchoreo-api/services/authz.go:20-23`),
> and `authz.Context.Resource.Environment` is **already populated at 15 authorization call sites**
> (`internal/authz/core/types.go:52-55`). The audit hook on that exact path —
> `audit.SetHierarchy` at `services/authz.go:48` — simply doesn't read it yet.
>
> **Consequence:** no adapter seed, no `SetResource` merge, no handler changes. The core of Piece A
> is a **~4-line change in one function**, and it lands on both surfaces at once, surviving
> denials. See §2.2 and D2.

---

## 0. Summary

| | |
|---|---|
| **Piece A** | `environment` as an audited attribute, rendered as `resource.environment` |
| **Piece B** | Collapse the two hand-maintained event render paths into one definition |
| **Environment-scoped state-modifying operations** | **18 of 116** — see the census in §2.1. Environment is *not* a universal tenancy dimension in OpenChoreo's API |
| **Where the value comes from** | The authorization check, which already has it at 15 sites. Not path parameters (there are none) and not request bodies (that is P3's territory) |
| **Core change for Piece A** | ~4 lines in `AuthzChecker.Check`, plus 3 targeted follow-ups for `/exec/`, wirelogs and alert channels |
| **`SetResource` merge (evaluation doc item 6)** | **Retired from scope** — the corrected design never seeds `Resource` pre-handler. §7 |
| **Size** | **S–M.** Piece B is the larger half |

---

## 1. Why these two are bundled

Piece B makes Piece A a one-edit change instead of a two-edit change with a silent-divergence
failure mode, and Piece A is Piece B's first real test — a new field that must appear identically
on both render paths.

They are also bundled because doing A *without* B is the worse ordering: adding a field by hand
to `Logger.LogEvent` and to `Event.MarshalJSON` separately is exactly the maintenance burden B
exists to remove, and P2 will then add ten more fields behind it.

---

## 2. Verified current state

Verified in the tree on 2026-09-07 (branch `feat/audit-schema`, at `6739d1bd9`). Every claim
below carries a citation; where a prior document said otherwise, that is called out.

### 2.1 The census — environment is a property of a few resource kinds, not a tenancy axis

**This is the finding that sizes the work.** `server.gen.go` registers **114** state-modifying
routes (`POST`/`PUT`/`DELETE`; no `PATCH`), and the full extracted route table shows the API is
**flat under namespace**. Only **three** carry an environment path parameter, and in all three
environment *is the resource being acted on*, not a dimension of it:

```
POST   /api/v1/namespaces/{namespaceName}/environments
PUT    /api/v1/namespaces/{namespaceName}/environments/{envName}
DELETE /api/v1/namespaces/{namespaceName}/environments/{envName}
```

So a generated path-parameter seed — the mechanism this plan was originally asked to evaluate —
would serve **almost nothing**. Environment reaches the API by four other routes:

| Class | How environment arrives | Operations | Evidence |
|---|---|---|---|
| **(a) The Environment CRUD itself** | Path parameter `{envName}`; environment *is* the resource | **3** | Route table above; `api/handlers/environments.go:200` |
| **(b) Release-binding family** — `releasebindings`, `projectreleasebindings`, `resourcereleasebindings` | `spec.environment` in the request body; **required and immutable** on `ReleaseBindingSpec` | **9** (POST/PUT/DELETE × 3) **+1** trigger | `api/v1alpha1/releasebinding_types.go:160-170` (`XValidation: self == oldSelf`), `projectreleasebinding_types.go:29`, `resourcereleasebinding_types.go:27`; trigger route `POST /api/v1alpha1/namespaces/{namespaceName}/releasebindings/{releaseBindingName}/trigger` |
| **(c) Observability alert channels** | `spec.environment`, required | **3** | `api/v1alpha1/observabilityalertsnotificationchannel_types.go:171` |
| **(d) Computed at request time** — `/exec/` | **Resolved**, not supplied: `resolveEnvName` derives the lowest environment from the project's deployment pipeline when unspecified | **1** | `api/handlers/exec.go:285-294`, used for the authz check at `exec.go:142` |
| **(e) Path parameter, outside the generated router** — wirelogs | `{environment}` path parameter on a hand-registered route | **1** *(a `GET`, audited deliberately)* | `api/handlers/exec_wirelogs_audit.go:20`: `WirelogsRoutePattern = "GET /api/v1/namespaces/{namespace}/environments/{environment}/wirelogs"` |
| **(f) Environment-agnostic** | — | **~96** | The remainder of the route table |

**18 environment-scoped operations of 116 audited state-modifying operations** — the 114
generated routes plus the two hand-registered ones, `/exec/` and wirelogs, which are audited
deliberately and are not in `server.gen.go`. Two consequences worth stating plainly:

1. **`resource.environment` must be optional**, and the schema reference must say so — the
   overwhelming majority of audited operations have no environment, and that is correct rather
   than a coverage hole. This is the same shape as `resource.uid`'s optionality (evaluation doc D2).
2. **Class (b) is 10 of the 19.** The release-binding family is the centre of gravity, and — per
   §2.2 — it is exactly the family that already hands an environment to the authorization layer.

**Wirelogs is already misusing a field for this.** `api/handlers/wirelogs.go:139` records the
environment as the resource *name*:

```go
audit.SetResource(r.Context(), &audit.Resource{Namespace: namespace, Name: environment})
```

Its own comment explains the namespace half — *"The middleware's pre-handler seed can't fill
resource.namespace for this route (its path parameter isn't `namespaceName`)"* — which is
independent evidence that the adapter seed is path-name-coupled and brittle. Once
`resource.environment` exists, this becomes `Environment: environment` with the correct `Name`,
and the record stops claiming an environment is a resource name.

### 2.2 The decisive finding — the environment is already in the authorization request

The evaluation doc checked `authz.ResourceHierarchy` (`internal/authz/core/types.go:30-35`),
found four fields and no environment, and concluded no writer could supply one. **It missed a
second struct on the same path.**

`authz.Context` carries root-namespaced ABAC attributes for CEL condition expressions, and its
`ResourceAttribute` holds the environment (`internal/authz/core/types.go:44-62`):

```go
// Context carries root-namespaced ABAC attributes available to CEL condition expressions.
type Context struct {
    Resource ResourceAttribute `json:"resource,omitempty"`
}

type ResourceAttribute struct {
    // Environment is the target environment (e.g. "dev", "staging", "prod").
    Environment   string `json:"environment,omitempty"`
    ComponentType string `json:"componentType,omitempty"`
    ResourceType  string `json:"resourceType,omitempty"`
    Workflow      string `json:"workflow,omitempty"`
}
```

It is declared as a policy condition attribute at `internal/authz/core/condition_registry.go:34-37`
(`Key: "resource.environment"`).

**And `CheckRequest` carries it right next to the hierarchy the audit hook already reads**
(`internal/openchoreo-api/services/authz.go:20-23`):

```go
ResourceID   string
Hierarchy    authz.ResourceHierarchy
Context      authz.Context
```

`AuthzChecker.Check` then calls the audit hook at `services/authz.go:48-53`, copying **only** the
four hierarchy fields and ignoring `req.Context` entirely. That hook's own doc comment
(`services/authz.go:44-47`) states why this is the right place for exactly this kind of value:

> *"It's recorded here, in the shared authz-check path, rather than on a REST path parameter or a
> handler's `SetResource` call, so both REST handlers and MCP tools (`pkg/mcp/mcpaudit`) get the
> hierarchy for free instead of each caller having to capture it individually."*

**Coverage available for free.** `authz.ResourceAttribute{Environment: …}` is populated at **15
`Check` call sites across 3 of the 36 `service_authz.go` files** — `releasebinding`,
`projectreleasebinding`, `resourcereleasebinding` (e.g. `services/releasebinding/service_authz.go:52-53,79,104-105,130-131,157-158`;
`services/projectreleasebinding/service_authz.go:47,74,97,120,144`). That is class (b) of the
census — 10 of the 19 environment-scoped operations — covered by a change in one function, on
both surfaces, **recorded before the PDP evaluates so it survives a denial**.

**One wrinkle, and it needs a decision (D3).** The value handed to authz is namespace-qualified,
not bare: `services.FormatDualScopedResourceName(namespace, name, false)` returns
`namespace + "/" + name` (`internal/openchoreo-api/services/utils.go:17-22`). So the authz
attribute holds `"default/production"`, while an audit record wants `"production"` —
`resource.namespace` is already its own field.

### 2.3 The two render paths (Piece B)

| Path | What it builds | Cite |
|---|---|---|
| `Logger.LogEvent` | `slog` attrs, by hand, group by group | `internal/server/middleware/audit/logger.go:57-144`, emitting at `:143` |
| `Event.MarshalJSON` | The JSON shape, independently, via `eventJSON` / `resourceJSON` | `internal/server/middleware/audit/types.go:106-182` |

Held in agreement by exactly one test, `TestEvent_MarshalJSONMatchesLogEventShape`
(`internal/server/middleware/audit/logger_test.go:183`).

**The same precedence rule is implemented three times** — the hierarchy's namespace unless
`Resource.Namespace` is non-empty:

- `logger.go:101-107` (with a comment at `:96-100` acknowledging the duplication)
- `types.go:166-179` (with a comment at `:158-164` doing the same)
- `withHierarchyNamespaceFallback`, `internal/server/middleware/audit/emitter.go:113`

`MarshalJSON` is already the published wire-shape contract for the future sink work, per its doc
comment at `types.go:136-141`. So this is not tidiness: it is **two competing definitions of the
published record**, plus a third partial one in the emitter.

`Hierarchy` is folded into the rendered `resource` group by both paths — `logger.go:108-116` and
`types.go:165-171` — which is why D1 below can put `environment` on `Hierarchy` and still have it
appear as `resource.environment` on the wire.

---

## 3. Decisions

### D1 — `environment` goes on `audit.Hierarchy`, and renders as `resource.environment`

**Decision.** Add `Environment string` to `audit.Hierarchy` (`types.go:54-59`), rendered inside
the `resource` group alongside `namespace`/`project`/`component`/`resource`.

**Why this satisfies the stated requirement anyway.** The instruction was that environment be *an
attribute of the resource*, and it is: both render paths fold `Hierarchy` into the `resource`
group (`logger.go:108-116`, `types.go:165-171`), so the wire shape is `resource.environment`
regardless of which Go struct holds it. The choice of struct is an internal one about **who can
write the field**, not about how it appears.

**Why `Hierarchy` rather than `Resource`:**

| | `Hierarchy` (chosen) | `Resource` (rejected) |
|---|---|---|
| Writer | `AuthzChecker.Check`, which **already has the value** at 15 sites (§2.2) | Handlers, from the created/updated object |
| Survives a denial? | **Yes** — recorded before the PDP evaluates (`services/authz.go:38-48`) | **No** — a denied create never reaches the handler |
| Surfaces | **Both**, free — the hook is shared by REST and `pkg/mcp/mcpaudit` | Both, but ~19 call sites each restating it |
| `SetResource` replace hazard | **None** — `Hierarchy` is a sibling precisely so `SetResource` cannot erase it (`types.go:40-53`) | Real — needs the merge fix first |
| Cost | ~4 lines in one function | ~19 handler edits + a merge change |

The denial case decides it. A **denied** attempt to bind a release into `production` is precisely
the audit record an operator most wants the environment on, and only the authz-path writer has it
then. This also aligns with the field's existing meaning: `resource.environment` is already an
*authorization condition attribute* (`condition_registry.go:34-37`), so recording it beside the
hierarchy the decision was made on keeps the audit record comparable to the policy scope — the
property `types.go:40-46` was designed for.

**Rejected alternative — envelope top level.** Out of scope by prior decision; tenancy stays
nested under `resource` (evaluation doc D1).

### D2 — Populate it in `AuthzChecker.Check`, from `req.Context.Resource.Environment`

**Decision.** Extend the existing `audit.SetHierarchy` call at `services/authz.go:48-53` to pass
the environment. This is the whole of Piece A's core.

**Consequence to accept explicitly:** coverage follows the *authz* attribute, not the census. Of
the 19 environment-scoped operations, this covers the 10 in class (b) immediately. Classes (a),
(c), (d) and (e) need the three follow-ups in D4 — and class (c), the alert channels, is
genuinely **not** currently passing an environment to authz despite having a required
`spec.environment`. That is a finding to record, not to fix here (§7).

**Observer's parallel hook.** `internal/observer/authz/helpers.go:111` also calls
`audit.SetHierarchy`, and already receives the ABAC attributes as its `authzCtx` parameter — so
it reads the environment the same way, with no signature change.

Verified: observer's environment-scoped operations are **all reads** — logs, traces, metrics,
events and finops queries (`internal/observer/service/*_authz.go`) — and reads are not audited.
Its only audited operations, the alert incidents, pass an empty `authzcore.Context`
(`alert_incident_authz.go:50`). So this records nothing in observer today. It is wired anyway,
rather than deferred, precisely because the value is already in the parameter: an audited
observer operation that becomes environment-scoped is then audited with its environment for
free, with no second change to remember.

### D3 — Record what the check already carries, verbatim

**Decision.** `Check` reads `req.Context.Resource.Environment` and records it **unchanged**. No
new field on `CheckRequest`, no transformation, no edits at the 15 wrapper sites.

**Two rejected alternatives, both drafted here first and both worse.**

| Option | Why not |
|---|---|
| A **bare `Environment` field on `CheckRequest`**, set alongside the ABAC attribute at all 15 sites | **Two sources of truth for one value.** A caller setting `Context.Resource.Environment` and forgetting the new field silently loses the environment from the audit record — a failure mode that does not exist today and that no compiler catches, since both fields are optional. Also 15 mechanical edits, and 15 more for every future environment-scoped operation |
| **Un-qualifying** the value — trimming the `namespace + "/"` prefix `FormatDualScopedResourceName` adds (`services/utils.go:17-22`) | A transformation that can disagree with the value the decision was actually made on, for a cosmetic gain. The namespace-qualified form is what authorization evaluated, and recording anything else means the audit trail and the policy no longer quote the same identifier |

**So the published value is the dual-scoped identifier** — `ns-1/production` for a
namespace-scoped environment, and the plain name for a cluster-scoped one. `resource.namespace`
carries the namespace separately, so a consumer that wants the bare name can split on `/`; what
it gains in exchange is that `resource.environment` is exactly the string the PDP matched its
conditions against.

**The property that makes this the right shape:** any operation that scopes authorization by
environment is audited with that environment **automatically**, with nothing for a caller to
remember. Coverage follows authorization, which is the same principle that already puts the
hierarchy on this hook rather than on 79 handler call sites.

One consequence to keep consistent: a handler that records an environment *before* the check runs
(wirelogs, D4) must write the same dual-scoped form, so the field has one shape whichever write
lands last.

### D4 — The four operations the authz path does not cover

| Class | Operation(s) | Treatment |
|---|---|---|
| (a) | Environment CRUD (3) | **Do nothing.** `resource.name` already carries the environment name; `resource.environment` would duplicate it. Record this in the schema reference so it does not read as a gap |
| (e) | wirelogs (1) | **Fix the existing misuse.** `wirelogs.go:139` sets `Name: environment`; change to `Environment: environment` with the correct `Name`. A small, self-contained correctness win |
| (d) | `/exec/` (1) | **Populate from the resolved value.** `exec.go` already computes `effectiveEnv` for its authz check (`:142`); pass it to audit too. This is the highest-sensitivity audited action, so it is worth the individual edit |
| (c) | alert channels (3) | **Out of scope** — see §7. They do not hand an environment to authz today, and adding one is an authorization change, not an audit change |

### D5 — Collapse mechanism: derive the slog render from `MarshalJSON`

**Decision.** Make `Event.MarshalJSON` the single definition of the published record, and build
the `slog` attrs by marshalling the event and walking the resulting object into `slog.Group`s.

**Why, over a shared field table.** Agreement becomes **structural rather than tested** — which
is what allows `TestEvent_MarshalJSONMatchesLogEventShape` to be retired as vacuous instead of
maintained against `environment` plus P2's ten further fields. It also inherits `omitempty` and
the namespace-precedence rule for free, and both are currently hand-reimplemented in the slog
path (§2.3).

**The cost, stated honestly.** One extra marshal per audit event. Acceptable *here specifically*:
audit emission is per-mutation rather than per-request, and it already runs synchronously in the
request goroutine after `next.ServeHTTP` returns, so this is not a hot path. If that ever
changes, the field-table alternative is the fallback and this decision should be revisited rather
than worked around.

**Two wire-shape questions the implementation must settle, not discover:**

1. **Timestamps.** `logger.go:60` uses `slog.Time("timestamp", …)`; `MarshalJSON` emits
   `time.Time`'s RFC3339 JSON. Under a JSON-derived render the slog attr becomes a **string**,
   not a `slog.Time`. With the repo's JSON handler (`internal/logging/logging.go`) the rendered
   bytes are expected to be identical, **but this must be asserted by test, not assumed** — it is
   the one place the collapse can silently change the published output.
2. **Key order.** Determine whether the existing agreement test asserts ordering. A `map`-based
   walk gives no stable order; if order is part of the contract (or merely desirable for
   human-readable logs), the walk must preserve `MarshalJSON`'s field order, which means walking
   a `json.RawMessage`/decoder token stream rather than a `map[string]any`.

Nested groups to preserve: `actor`, `actor.entitlements`, `resource`, `resource.metadata`,
`metadata`.

---

## 4. Piece B — the collapse, concretely

1. **Reconcile `Hierarchy` into `Resource` once, before render.** Today the fold happens in each
   render path, and `withHierarchyNamespaceFallback` (`emitter.go:113`) does a partial third
   version. Move the whole reconciliation into `buildEvent` (`emitter.go:72`) or into a single
   `func (e Event) resolvedResource() *resourceJSON` that `MarshalJSON` calls — so the precedence
   rule exists **once**.
   *Note the constraint that created the split:* `Hierarchy` must stay a sibling **field** on
   `Event` and `AuditData` so `SetResource` cannot erase it (`types.go:40-53`). Reconciling at
   render time is compatible with that; merging the structs at *write* time is not.
2. **Make `MarshalJSON` authoritative**, built on that single reconciliation.
3. **Rewrite `Logger.LogEvent`** to marshal then walk into attrs, preserving group structure and
   settling D5's two questions.
4. **Retire or repurpose the agreement test.** If the shapes are now structurally identical,
   `TestEvent_MarshalJSONMatchesLogEventShape` becomes vacuous — replace it with a **golden-file
   test** on the rendered log line, which is what actually protects the published contract and
   which P2's ten new fields will each extend.

---

## 5. Piece A — the environment attribute

1. `Environment string \`json:"environment,omitempty"\`` on `audit.Hierarchy` (`types.go:54-59`).
2. Render it inside the `resource` group. **With Piece B done this is one edit**, in
   `resourceJSON` plus the single reconciliation — placed after `namespace` and before `project`,
   matching the existing tenancy order.
3. Add a bare `Environment` to `CheckRequest`, set at the 15 wrapper sites, and pass it from
   `AuthzChecker.Check` (`services/authz.go:48-53`) into the existing `audit.SetHierarchy` call
   (D2, D3).
4. Fix `wirelogs.go:139` and `exec.go` per D4.
5. Document in the generated schema reference: optional, present only for environment-scoped
   operations, with the four classes of §2.1 named — including why the Environment CRUD does not
   set it.

---

## 6. Tasks

**Task 1 is the golden file, and it comes first** — it must capture today's rendered output
*before* any render change, or it cannot prove the collapse was shape-neutral. §10's first risk
row is the reason.

| # | Task | Piece | Dep |
|---|---|---|---|
| 1 | **Golden-file test on the rendered log line, against today's output** — the baseline the collapse is measured against | B | — |
| 2 | Single-source the `Hierarchy`→`Resource` reconciliation (incl. the namespace-precedence rule) in one function; delete the two duplicate implementations and reconcile `withHierarchyNamespaceFallback` | B | 1 |
| 3 | Make `MarshalJSON` the authoritative definition on top of task 2 | B | 2 |
| 4 | Rewrite `Logger.LogEvent` as marshal-then-walk; settle timestamp rendering and key order (D5). **Golden file from task 1 must be unchanged** | B | 3 |
| 5 | Retire `TestEvent_MarshalJSONMatchesLogEventShape` as vacuous, once agreement is structural | B | 4 |
| 6 | Add `Hierarchy.Environment`; render as `resource.environment` | A | 3 |
| 7 | Read `req.Context.Resource.Environment` into `audit.SetHierarchy` in `AuthzChecker.Check` (D2, D3) — **no wrapper-site edits** | A | 6 |
| 8 | Same in observer's `CheckAuthorization`, from its `authzCtx` parameter (D2) | A | 6 |
| 9 | Fix `wirelogs.go:139` — `Environment` instead of `Name`, in the dual-scoped form (D4) | A | 6 |
| ~~10~~ | ~~Populate environment for `/exec/`~~ — **nothing to do:** exec already passes an environment to its check (`exec.go:140-144`), so task 7 covers it | A | — |
| 10 | Schema-reference entry: optional, the five classes, and why Environment CRUD omits it | A | 6 |
| 11 | Test that a **denied** release-binding create records `resource.environment` — the property D1 was chosen for | A | 7 |
| 12 | Test that an operation with no ABAC environment records none | A | 7 |

---

## 7. Non-goals

- **`SetResource` merge semantics.** **Retired from scope by D1** — nothing seeds `environment`
  onto `Resource`, so the replace hazard is never triggered. It remains a latent trap for any
  *future* pre-handler-seeded `Resource` field, and `SetResource`'s doc comment already documents
  it (`context.go:26-36`). Evaluation-doc §7 item 6 should be marked retired, not deferred.
- **Request-body field extraction.** Reading `spec.environment` from a request body in middleware
  is explicitly P3's territory (its non-goals name *"body-field extraction and the oapi-codegen
  strict middleware"*). D2 deliberately takes the value from the authz layer instead, which needs
  no body access.
- **Giving alert channels an authz environment attribute** (class (c), 3 operations). They carry
  a required `spec.environment` (`observabilityalertsnotificationchannel_types.go:171`) but pass
  no environment to authz. Adding one changes what policies can express — an authorization
  decision. **Recorded here as a finding**; it should become its own issue, and once done these
  three operations gain the audit field for free.
- **An `environments` audit-policy selector.** The policy vocabulary in
  `internal/auditconfig/audit.go` has nine validated selector axes, and once
  `resource.environment` is an event field a tenth becomes *possible* — the same way P1's
  `resource.namespace` unlocked a `namespaces` selector earlier than predicted. **Deliberately
  out of scope here.** This plan's job is to get the field emitted correctly on both surfaces;
  adding a selector is an operator-facing config change with its own validation, schema
  (`values.schema.json`) and documentation surface, and it should be judged on operator demand
  once the field exists. Recorded so its absence reads as a decision.
- **`Resource.Metadata` / `Event.Metadata`.** `environment` is a real struct field, never a
  metadata key. Both maps remain writer-less until P3 delivers the typed registry.
- **Any change to the category taxonomy** (`management` / `authorization`) — settled and permanent.
- **Promoting tenancy to the envelope top level** — decided against (evaluation doc D1).
- **P2's other twelve items.** This plan is P2's prerequisite, not P2.

---

## 8. Demo

```sh
# 1. Piece B: the published shape is unchanged by the collapse
make test GO_RUN_ARGS="-run 'TestLogEvent|TestEvent_' ./internal/server/middleware/audit/..."
#   → golden-file test passes; the old agreement test is gone as vacuous

# 2. Piece A: a release-binding create records the environment
make k3d.update.openchoreo-api
curl -X POST http://openchoreo.localhost:8080/api/v1/namespaces/default/releasebindings \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"name":"demo-rb","environment":"production", ...}'
make k3d.logs.openchoreo-api | grep AUDIT-LOG | tail -1 | jq '.resource'
#   → { "type": "releasebindings", "namespace": "default",
#       "environment": "production", "name": "demo-rb", "uid": "..." }
#   note: "production", NOT "default/production"  (D3)

# 3. The property D1 was chosen for: a DENIED attempt still carries it
curl -X POST .../releasebindings -H "Authorization: Bearer $LIMITED_TOKEN" \
  -d '{"name":"demo-rb","environment":"production", ...}'
make k3d.logs.openchoreo-api | grep AUDIT-LOG | tail -1 | jq '.result, .resource.environment'
#   → "denied", "production"

# 4. Surface parity: the same operation via MCP
#   → same resource.environment, differing only in origin

# 5. Wirelogs no longer records the environment as a resource name
#   → resource.environment set, resource.name is the real target

# 6. Pre-PR gate sequence (.claude/CLAUDE.md)
make code.gen && make lint-fix && make test && make code.gen-check
```

---

## 9. Exit criteria

1. **One** definition produces the published record; the duplicate namespace-precedence
   implementations are gone and the golden-file test guards the wire shape.
2. `resource.environment` is emitted on the release-binding family, `/exec/` and wirelogs,
   carrying the same dual-scoped identifier the authorization decision used (D3).
3. **A denied release-binding create records `resource.environment`** — task 12.
4. Surface parity: the same operation through REST and MCP produces the same
   `resource.environment`.
5. `make code.gen && make lint-fix && make test && make code.gen-check` all pass.
6. The schema reference documents the field as optional and names the operation classes that omit
   it — including the Environment CRUD, deliberately.
7. The evaluation doc's §7 items 0, 6 and 7 are reconciled: 0 and 7 delivered, **6 retired** with
   its reason.

---

## 10. Risks and open questions

| Risk | Judgement |
|---|---|
| **The collapse silently changes the published log shape** — the timestamp rendering in D5 is the real exposure | The one genuine risk here. Mitigated by task ordering: the golden file (task 1) is captured against today's output **before** task 4 changes the render |
| ~~The namespace trim drifts from `FormatDualScopedResourceName`~~ | **Eliminated by D3** — the bare value is passed through rather than reconstructed, so there is no coupling to drift |
| **Key ordering is part of a contract nobody wrote down** | Resolve by inspection during task 3 (D5 question 2). If order matters, walk a token stream rather than a map |
| **Coverage tracks the authz attribute, not the census** | Deliberate and documented (D2). 10 of 19 free; 2 more by targeted edit; 3 out of scope (alert channels); 3 correctly omitted (Environment CRUD). **A future operation that is environment-scoped but does not pass one to authz gets no environment silently** — worth considering a coverage assertion, but only after the alert-channel issue settles what "environment-scoped" means declaratively |
| **Observer may have no environment-scoped operation** | Then task 10 is a comment, not code. Verify rather than assume |

---

## Appendix — corrections to the evaluation doc

| `openchoreo-audit-schema-evaluation.md` says | Corrected |
|---|---|
| §6 D1: `environment` *"has no writer available"*; both `Hierarchy` writers derive from `authz.ResourceHierarchy`, which lacks it | **Wrong.** `CheckRequest` carries `Context authz.Context` beside `Hierarchy` (`services/authz.go:20-23`) and `authz.Context.Resource.Environment` is populated at 15 sites (`internal/authz/core/types.go:52-55`). The writer exists; the audit hook just doesn't read it |
| §6 D1 / §7 item 6: `SetResource` must be made merging first | **Retired.** The corrected design never seeds `Resource` pre-handler |
| §7: `environment` is *"new plumbing"* sized as an open question | **~4 lines in one function** for 10 of 19 operations, plus 2 targeted edits |
| §4: "the other three tenancy fields already emit" | Confirmed. And `environment` renders as `resource.environment` from `Hierarchy`, so the nesting decision stands unchanged |
