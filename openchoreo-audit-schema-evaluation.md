# Audit Event Schema — v1 Freeze Evaluation

**Companion to:** [`openchoreo-audit-logging-phased-plan.md`](./openchoreo-audit-logging-phased-plan.md) §P2 (the proposal being evaluated) and [`openchoreo-audit-logging-scope.md`](./openchoreo-audit-logging-scope.md) §2.1 (what each `EM-x` means).
**Spun out of this evaluation:** [`openchoreo-audit-environment-attribute-plan.md`](./openchoreo-audit-environment-attribute-plan.md) — the render-path collapse (§5) and the `environment` attribute (§6 D1), which are P2's prerequisites rather than part of it.
**Phase:** P2 — Canonical event schema v1 · **Branch:** `feat/audit-schema-v1`
**Status:** In implementation · **Date:** 2026-09-04, revised 2026-09-07

> **Why this document exists.** P2's task table has 18 rows. This evaluates them against the
> event the code actually emits today, and separates the rows that *must* be in v1 from the rows
> that can arrive later without breaking a consumer. Everything here marked "verified" was read
> out of the tree at `6739d1bd9`, with a `file:line` citation. Where the plan and the code
> disagree, the code wins and the disagreement is called out.

> ### ✅ Revision 2026-09-07 — items 0 and 7 are implemented; item 6 is retired
>
> The prerequisite phase in
> [`openchoreo-audit-environment-attribute-plan.md`](./openchoreo-audit-environment-attribute-plan.md)
> is **built**, on branch `feat/audit-environment-attribute`. Three of §7's rows are closed and
> two of §9's criteria change:
>
> | §7 item | Now |
> |---|---|
> | **0** — collapse the two render paths | **✅ done.** `Logger.LogEvent` renders `Event.MarshalJSON`'s output (`logger.go:62`); the agreement test is deleted, replaced by a byte-exact golden test |
> | **6** — make `SetResource` merge | **retired.** `environment` lives on `Hierarchy`, which `SetResource` cannot erase, so the replace hazard is never reached |
> | **7** — `environment` | **✅ done.** Read from `req.Context.Resource.Environment` in `AuthzChecker.Check` (`services/authz.go:52`), and the same in observer (`observer/authz/helpers.go:116`) |
>
> **Two things implementation established that this document could only assume:**
>
> 1. **The two render paths had already drifted** — §5 argued they *could*; they had. The
>    marshalled form ordered `origin`/`operation_id`/`resource` before `result`, and emitted
>    `"resource":null` where the log stream omitted the group. The single test guarding them
>    compared selected keys and missed both. The collapse is byte-neutral for the **log stream**
>    (golden test captured against the old implementation), and normalises the marshalled form,
>    which has no consumer yet.
> 2. **`resource.environment` publishes the dual-scoped `{namespace}/{name}` form**, not the bare
>    name — it is stored exactly as authorization evaluated it, so the record quotes the string
>    the PDP matched. **This is a schema-visible decision the reference (item 12) must document**,
>    and it is the one field whose value shape differs from its sibling tenancy fields.
>
> **Item 11 (`sequence`) gets cheaper as a result.** §5's "two edits per field" cost is gone;
> every remaining field row is now a single edit in `eventJSON`/`resourceJSON`.

> ### 🔧 Revision 2026-09-07 (second) — P2 opens; three decisions taken, one item deferred
>
> **Branch:** `feat/audit-schema-v1`, off `feat/audit-environment-attribute`. (`feat/audit-schema`
> already existed as a stale placeholder at `527703391` with no unique commits; left untouched.)
>
> Reading the tree before implementing settled the three decisions §6 had left open or
> self-contradictory, and retired one v1 item:
>
> | Decision | Taken |
> |---|---|
> | `schema_version` type — §7 said `1` (int), §10.1 said `"1.0"` | **`"1.0"`, string**, CloudTrail's major/minor rule verbatim. See D6 |
> | `http.path` value — never picked | **The request path as sent, query string excluded.** Settled through three candidates; see D7 |
> | Which name today's `timestamp` keeps | **Neither as-is.** `event_time` is a *new capture* at adapter entry; `received_time` inherits today's emit-time value. See D8 |
> | **Item 9 — `actor.assurance`** | **Deferred to P9.** It is derivable from `actor.type` today, so it fails §1's own test. See §8 |
> | **Item 11 — `sequence`, and `producer_instance` from item 2** | **Deferred to P6, as a pair.** See D9 |
> | **`received_time`, half of item 3** | **Deferred to P4/P5a.** The rename and re-capture ship; the second timestamp doesn't. See D8's second revision |
> | **Item 4 — `audit_id`** | **Deferred to P10a (2026-09-08).** 1:1 with `event_id` until a second event per operation exists, so it ships with the pre-action event |
> | **D5's `observer` → `observer-api`** | **Retired on review.** `producer` names the deployed workload, which `observer` already was; the rename would have broken D5's own rule |
> | **D2's `uid` definition** | **Broadened on review** to "server-generated and never reused" — observer's incident id is a `uuid.NewString()`, so it qualifies, and `name` equals `uid` where a resource has no user-chosen name |
>
> **The cut that reshapes the phase — apply §1's test consistently and v1 loses three fields.**
> §4 justified `sequence` on cost alone (*"marginal cost now ≈ 0; later it's a second PR through
> §5's two paths"*). **Item 0 removed that argument**: `eventJSON` is now a single definition, so
> adding it in P6 is equally cheap. And the mechanism it belongs to does not exist yet —
> **P4 and P5a have not started, so today's only sink is an in-process `slog` writer to stdout**
> (`emitter.go:62` → `logger.go:90`). There is no drop-prone hop between producer and consumer,
> so sequence numbers emitted now would prove the integrity of a transport that isn't there.
>
> That produces the line v1 is actually cut on:
>
> | | Kept in v1 | Reason |
> |---|---|---|
> | **Content** — describes a request, unrecoverable once it's over | `schema_version`, `event_time`, the three renames, `actor.issuer`/`session_id`, `http.method`/`path` | A request that already happened cannot be re-described later. The renames and `event_time`'s re-capture are **major** bumps under D6 if deferred |
> | **Pipeline metadata and machine correlation** — introduce it with the component it describes | ~~`producer_instance`~~, ~~`sequence`~~ → P6 · ~~`received_time`~~ → P4/P5a · `audit_id` **kept** | Additive under D6, so deferral is free and checkable. Each describes a collection pipeline that doesn't exist yet |
>
> **`audit_id` was kept here, then deferred on 2026-09-08.** §4 argues a correlation key "cannot
> group events already written" — hollow, because before P10a there is exactly **one** event per
> operation, so historical events have nothing to be grouped with. It was first kept as a **1:1
> stamp** so P10a would inherit a populated field rather than a key beginning mid-stream. That
> uniformity argument lost to the same test that retired everything else in this row: the field
> is additive, so **it ships with the pre-action event that gives it a job.** Until then the
> consumer rule is that an event is its own group.
>
> **Four things reading the code established that §4 assumed:**
>
> 1. **Item 8 is more expensive than "no re-parse" implies.** Claims *are* in request context
>    (`jwt/context.go:18`), but the only accessor is `GetClaims(r *http.Request)`
>    (`jwt/context.go:24`) while `ExtractActor` takes a `context.Context` (`adapter.go:26`). And
>    `auth.SubjectContext` carries **no claims at all** — only ID/Type/entitlements
>    (`auth/interface.go:12-17`), with `resolver.go:92` reading `sub` and discarding the rest.
>    Fix is to promote `iss`/`sid` onto `SubjectContext` where it is built (`resolver.go:93`),
>    keeping the audit package ignorant of token formats.
> 2. **`timestamp` does not mean what its comment says.** `buildEvent` stamps `time.Now()`
>    (`emitter.go:83`) and the REST adapter emits from a `defer` *after* `next.ServeHTTP`
>    (`middleware.go:183`) — so the value is response-complete time, not "when the action
>    occurred" (`types.go:92`). For a hijacked exec/wirelogs connection it can be minutes late.
>    Renaming it to `event_time` would have silently blessed that. D8.
> 3. **`resource.id` → `uid` is ~73 edits, not ~127.** All 55 MCP sites route through one helper
>    (`mcphandlers/helpers.go:28`), so MCP is **one** edit; REST is 72 struct literals. All
>    compiler-caught.
> 4. **`sid` has no reader, writer, or config anywhere in the tree**, and there is no evidence the
>    bundled Thunder IdP issues it. `actor.session_id` ships, then gets verified against a live
>    token; if absent it is a documented omission class, not a silent empty field. `iss` is
>    guaranteed by OIDC, so `actor.issuer` always populates.
>
> **Mechanical cost, measured:** every wire-visible row means editing all 7 golden `want` strings
> in `record_shape_test.go`, which pin exact bytes *and* field order. That is the phase's real
> cost, and it is a feature — each ordering decision lands in review.

---

## 0. The short answer

**The event shape is ready to freeze. It is not ready to freeze *as-is*.** The architecture
survived its only real test; the naming and the tenancy shape did not get tested and are where
the remaining risk sits.

| | Count | |
|---|---|---|
| P2 tasks as planned | 18 | |
| **Already delivered** (plan is stale) | **2** | 2.4, 2.8 — and 2.12 *in substance*, see finding 1 |
| **v1-critical** — absence or wrong shape is a breaking change later | **~~8~~ 6** | §7, after D9 |
| **Additive-safe** — defer with a recorded reason | **~~3~~ 5** | §8 |
| **Blocked on another phase** | **2** | 2.13 (P4), 2.15 (§4.9.2) |
| **Documentation deliverables of v1** | **3** | 2.14, 2.16, 2.17 |

*The rows count the plan's **18 planned tasks**; §7 counts **14 work items**. They do not map
1:1 — task 2.5 splits into a decision plus a semantics fix (D1), 2.9 splits across v1 and
deferred, and 2.12 reduces to a rename. Read the two as different units.*

§7's 14 items are: 3 documentation, 3 renames of live fields, 2 prerequisites, and — **after the
2026-09-07 revisions retired items 0, 6, 7, 9 and 11** — only **2 genuinely new fields**
(`schema_version` alone, after `audit_id` also went), plus one new *capture* of an existing field (`event_time`), two
`actor.*` claims and an optional `http` group. The phase is much smaller than its task table
implies, and most of that shrinkage came from applying §1's own additive-safe test to the rows
§4 had waved through on cost.

Five findings change its shape, and none could have been written a month ago:

1. **Task 2.12's parity gap is already closed.** The plan says `resource.uid` ships at "78 REST
   sites but only 6 MCP sites" and re-scopes P2 to closing that. MCP in fact has **55** call
   sites through a shared `setAuditResource` helper (`mcphandlers/helpers.go:25`), landed in P1's
   own #4571 and test-enforced. A *different* gap remains — 7 REST sites carry no UID at all.
2. **The core is proven service-neutral**, so the schema is safe to version. §3.
3. **`resource.id` holds the Kubernetes UID, not a logical id.** EM-14 asks for `uid` *alongside*
   `id` assuming `id` is a name. It isn't — `name` plays that role. v1 needs a rename, not a
   field, plus an honest note that deletes carry no UID. §6 D2.
4. **`environment` was the one genuinely missing tenancy field**, and it is environment-scoped on
   only **18 of 116** state-modifying operations — a property of a few resource kinds, not a
   tenancy axis. **Now implemented.** *(Twice revised: this finding first said no writer existed
   and that `SetResource` merge was a prerequisite — both wrong, see the D1 supersession note.)*
5. **`actor.subject` is already emitting as `actor.id`** — `ExtractActor` assigns it from the
   validated `sub` claim (`adapter.go:40-51`). Only `session_id` and `issuer` are new. Third
   instance of the plan asking for a field the code already has under another name.

**Independently corroborated.** §10 checks the result against Kubernetes audit, CloudTrail, OCSF
and CloudEvents: OCSF's required `metadata` object has a counterpart for every v1 envelope field
but one, Kubernetes models resources as `name` + `uid` with no third identifier (exactly D2), and
CloudTrail's published major/minor rule **is** the additive-safe axis §1 uses to cut the set.

And one prerequisite that is not a field at all: **two hand-maintained render paths** had to be
collapsed before eight fields were added to both of them. §5 — **done**, see the revision note.

---

## 1. The axis that decides "minimal"

Every one of the 18 tasks is worth doing. That is not the question a *freeze* phase asks. P2
exists because P5a routes on this shape, P7 indexes it and P8 renders it, so the discriminating
test is:

> **Does adding this field later force a `schema_version` bump, an index remapping, or a
> query-contract change?**

- **v1-critical** — the field is a *rename or reshape* of something already emitting, or its
  absence cannot be repaired retroactively (a correlation key can't group history already
  written).
- **Additive-safe** — a consumer that ignores the field today keeps working when it appears.
  Defer, and record why.

Applying that test is what produces the 8, not a judgement about which fields matter most.

---

## 2. Verified current state — the event as emitted

`Logger.LogEvent` (`internal/server/middleware/audit/logger.go:57-144`) is the authoritative
render; it emits `slog.Info("AUDIT-LOG", …)` at `logger.go:143`. `Event` is
`internal/server/middleware/audit/types.go:86-102`.

```json
{
  "event_id":  "…",                       // UUID v7, stamped in buildEvent (emitter.go:72)
  "timestamp": "…",
  "actor":    { "type": "…", "id": "…", "entitlements": { … } },
  "action":       "create_project",
  "category":     "management" | "authorization",
  "result":       "success" | "failure" | "denied" | "unauthenticated",
  "request_id":   "…",
  "source_ip":    "…",
  "service":      "openchoreo-api" | "observer",
  "origin":       "api" | "mcp",
  "operation_id": "CreateProject",
  "resource":  {
    "type": "projects", "namespace": "…", "environment": "<ns>/<env>",
    "project": "…", "component": "…",
    "resource": "…", "id": "<k8s UID>", "name": "…"
  }
}
```

> `environment` was added 2026-09-07 (see the revision note) and is the only tenancy field
> carrying a namespace-qualified value. Line citations in this section predate that change; the
> current anchors are `logger.go:62` (`LogEvent`), `types.go:57` (`Hierarchy`), `types.go:90`
> (`Event`), `types.go:148` (`MarshalJSON`).

Four structural facts about that shape, each load-bearing below:

| Fact | Evidence |
|---|---|
| **Tenancy is nested under `resource`, and comes from two places.** `namespace` from `Resource.Namespace`, and `project`/`component`/`resource` from a sibling `Hierarchy` struct folded into the `resource` group **only at render time** | `types.go:54-59`, `types.go:125-134`, `logger.go:91-133`. The comment at `types.go:40-53` records why it is a sibling: `SetResource` *replaces* the whole `*Resource`, so a hierarchy stored inside it would be wiped by any later handler call |
| **`Hierarchy` mirrors `authz.ResourceHierarchy`** so a record is directly comparable to a policy scope — **no longer field-identical:** `Environment` was added 2026-09-07 and has no counterpart there, coming instead from `authz.Context.Resource` | `types.go:57` vs `internal/authz/core/types.go:30-35` |
| **`resource.id` is the Kubernetes UID.** `Resource` has `ID` and `Name` and no `UID` field | `types.go:33-38`; call sites pass `ID: string(created.UID), Name: created.Name` (e.g. `api/handlers/clusterworkflowplanes.go:107`, `projecttypes.go:81`) |
| **`resource.metadata` and top-level `metadata` are plumbed but have no writer**, so with `omitempty` they never reach the wire | `types.go:37`, `types.go:101`, and the `AuditData` doc comment at `types.go:184-198` states it outright. P1's non-goals forbid populating either before P3 |

**Two producers exist:** `audit.NewEmitter("openchoreo-api", …)` at `cmd/openchoreo-api/main.go:213`
and `audit.NewEmitter("observer", …)` at `cmd/observer/main.go:591`.

---

## 3. The decisive evidence — the core is service-neutral

The plan sequenced P1.5b *before* P2 for one stated reason: extracting the framework for a second
service is *"the only thing that will reveal whether today's `Operation` / `OperationDef` /
selector-vocabulary shape is genuinely service-neutral or quietly openchoreo-api-shaped."*
P1.5b shipped as `6739d1bd9`. **The answer is in the diff.**

| Area | Change in `6739d1bd9` |
|---|---|
| **`internal/server/middleware/audit/` — the event, emitter, envelope, policies, logger** | **3 lines, in a doc comment.** Nothing else in the package changed |
| Generator | Extracted to `tools/internal/auditgen/` (+328), per-service config split out |
| Config vocabulary | Extracted to `internal/auditconfig/` (+271), `openchoreo-api/config/audit.go` −272 |
| Observer wiring | `internal/observer/audit/`, `audit_wiring_test.go` (+396), coverage gate (+222) |

**55 files, 3661 insertions — and the event schema needed nothing.** The service-specific
hardcoding the observer plan predicted was real, and it was *all* in the generator and config
layers, exactly where that plan located it. The `Event`/`Envelope`/`Emitter` triple carried a
second producer unmodified.

**What this licenses, and what it does not.** It licenses versioning: a shape that absorbed a
second producer without a change is not going to need a v2 for a reason a third producer
discovers. It does **not** license freezing the current *field names* — `NewEmitter("observer")`
was the only schema-visible decision the second producer made, and §6 D5 shows it was made
inconsistently. The architecture was tested. The vocabulary wasn't.

---

## 4. Field-by-field — today vs proposed vs v1

`EM` references are the scope doc's §2.1. "Today" is verified against the tree; a ✅ means
emitting now, not planned.

| Field | Task | EM | Today (verified) | Proposed | v1? | Why |
|---|---|---|---|---|---|---|
| `schema_version` | 2.1 | EM-4 | ○ absent | new, `"1.0"` | **✅ v1** | Cannot be retrofitted *by definition* — an unversioned event can't later be identified as v1. The one field with no additive-safe path. **String, major.minor** — D6 |
| `producer` | 2.2 | EM-10 | ◐ `service` string (`types.go:100`) | rename **only** | **✅ v1** | **The rename is the argument, not the new field.** Two producers, zero consumers, and it's free today. After P7 indexes `service` it's a migration. **The *values* are unchanged** — D5's `observer` → `observer-api` was retired on review, since `producer` names the deployed workload and `observer` already did |
| ~~`producer_instance`~~ | 2.2 | EM-10a | ○ absent | new | **✗ defer to P6** | **Reversed 2026-09-07.** Only meaningful as `sequence`'s scoping key — a random per-process UUID with nothing to join to otherwise — so it moves with it. D9 |
| `event_time` | 2.10 | EM-13 | ◐ one `timestamp`, and it holds **response-complete** time (`emitter.go:83` + `middleware.go:183`), not what `types.go:92` claims | rename **and re-capture** at adapter entry | **✅ v1** | A **reshape** — the field's value changes meaning, which is a major bump under D6 if deferred. Not a pure rename, and not a split either: **`received_time` was cut** — D8 as revised |
| ~~`received_time`~~ | 2.10 | EM-13 | ○ absent | new | **✗ defer to P4/P5a** | **Reversed 2026-09-07.** Additive, and OCSF defines its counterpart `logged_time` as a **collector's** stamp — so a producer-side version approximates a field belonging to a component P4 hasn't built. Same objection D9 raised to `sequence`. Two timestamps microseconds apart also invite a wrong latency subtraction. D8 |
| ~~`audit_id`~~ | 2.11 | EM-13a, EM-13c | ○ absent | new | **✗ defer to P10a** | **Reversed 2026-09-08.** §4's original reason — a correlation key "cannot group events already written" — is hollow: one event per operation today means history has nothing to group, so `audit_id` would be 1:1 with `event_id` until the pre-action event exists. It lands **with** the two-event model that gives it a job. EM-13a's point stands and is recorded for P10a: `request_id` cannot substitute, being **client-suppliable** (`adapter.go` bounds its *shape*, not its uniqueness) and already 1:many on MCP, where one HTTP request carries several `tools/call` operations |
| **Tenancy shape** | 2.5, 2.6 | EM-5 | ✅ **decided and implemented** — stays nested under `resource` | promote 4 fields to envelope | **✅ settled** | An **index-mapping decision**, not a field addition. Decided against promotion; §6 D1 records why |
| `environment` | 2.5 | EM-5 | ✅ **shipped 2026-09-07** — on `Hierarchy`, rendering as `resource.environment` | new | **✅ done** | Read from `authz.Context.Resource.Environment` in `AuthzChecker.Check`. Carries the dual-scoped `{namespace}/{name}` form, which item 12 must document |
| `resource.uid` naming | 2.12 | EM-14 | ◐ **as `resource.id`**, at 72 of 79 REST sites; MCP parity done (55 sites) | rename decision | **✅ v1 (rename)** | The *surface-parity* gap the plan describes is closed; a **coverage** gap remains at the 7 sites that pass `Name` only — all deletes and non-CRUD mutations where the service returns no object. D2. **~73 edits**, not ~127: MCP's 55 sites share one helper |
| `actor.session_id`/`issuer` | 2.7 | EM-8 | ○ absent (`Actor` is type/id/entitlements, `types.go:12-16`) | new | **✅ v1** | Additive-safe in principle, but the claims are already in request context — no re-parse, no signature check. **Deferring costs more than doing it**, and 2.16's IdP guide depends on it. **Costs more than the plan implies:** no `context`-based claims accessor exists and `SubjectContext` carries no claims — see revision 2 finding 1. `sid` is unverified against any real IdP (finding 4) |
| ~~`actor.subject`~~ | 2.7 | EM-8 | ✅ **already emitting, as `actor.id`** | — | **✗ don't add** | `ExtractActor` assigns `actor.id` from the validated `sub` claim (`adapter.go:40-51`, with a `"<nil>"` guard). Adding `actor.subject` duplicates it. **Recommend: document `actor.id` as the `sub` claim** rather than emit the same value twice. Third instance of the plan naming a field it already has |
| ~~`actor.assurance`~~ | 2.9 | EM-12 | ○ absent | new | **✗ defer to P9** | **Reversed 2026-09-07.** `ExtractActor` only ever sees a `SubjectContext`, and one exists only where `jwt.Middleware` already validated the signature — so `assurance == "validated"` ⟺ `actor.type != "anonymous"`, a constant restating a published field. Adding it is a **minor** bump under D6's own rule, so P9 can add it alongside `on_behalf_of`, the field that gives it meaning. The reference records *absent means validated* |
| `http.method`, `http.path` | 2.3 | EM-2 | ○ absent — only the resolved `action`/`operation_id` | new | **◐ v1, optional** | EM-2 calls them core, but **MCP has no request line.** Optional, present at an HTTP boundary only — §6 D3 as refined. `path` is the **request path as sent, query excluded**, so it adds no identifier the `resource` group lacks and nothing new to the PII register — D7 |
| ~~`sequence`~~ | 2.18 | RE-7 | ○ absent | new | **✗ defer to P6** | **Reversed 2026-09-07.** The cost argument that put it in v1 died with item 0, and **it detects head and interior loss but never tail loss** — the received run `1..80` of an emitted `1..100` looks complete. Closing that needs P6's heartbeat or emitted-count, so the producer half should be designed with the detector, not two phases ahead of it. D9 |
| `actor.on_behalf_of` | 2.9 | EM-9 | ○ absent | new | ✗ defer | Additive-safe, and **nothing can populate it** until P9 introduces a delegating client. A `Should`, per the scope doc |
| `reason` | 2.12 | EM-15 | ○ absent | new | ✗ defer | Additive-safe. A `Could`, and it needs a break-glass UX that doesn't exist |
| `origin: client` | 2.4 | EM-11 | ✅ `api`\|`mcp` shipped in P0 (`types.go:80-83`) | + `client` | ✗ defer | Plan already marks this done bar the third value, which P9 introduces. Adding an enum value is additive by construction |
| `request_id` ← `traceparent` | 2.13 | EM-16 | ◐ hardened, not derived | derive | **⛔ blocked** | P4 has not started; §2.11 verified there is no trace to join. **Not deferred — blocked.** Say so rather than folding it into a set that can't be finished |
| OCSF mapping | 2.15 | EM-17 | ○ | doc | **⛔ blocked** | Gated on scope §4.9.2. Also *should* follow the rename decisions, not precede them |
| Schema reference | 2.14 | DO-1, EM-1/2 | ○ | generated doc | **✅ v1** | This is what "freeze" means in practice. Nothing else in v1 is verifiable without it |
| IdP correlation guide | 2.16 | DO-5 | ○ | doc | **✅ v1** | Cheap, and it's the *reason* 2.7 exists |
| PII register | 2.17 | SE-6 | ○ | doc | **✅ v1** | Must cover **`actor.entitlements`**, which P1 shipped as an emitted field with no plan (`logger.go:67-73`), plus the new `actor.*` claims. Freezing a schema without documenting what's personal in it is the wrong order |
| `resource.metadata`, `metadata` | — | — | ◐ plumbed, **no writer, never on wire** | — | ✗ **exclude from v1** | §6 D4 |

### The stale row — task 2.12 is already delivered

The plan states `resource.uid` ships at *"78 REST sites but only 6 MCP sites"* and re-scopes P2
to *"finish MCP and close a surface-parity gap."* Verified against the tree:

```
REST  audit.SetResource( in api/handlers/, non-test   →  79
MCP   setAuditResource(ctx  in mcphandlers/, non-test →  55  across 13 files
```

MCP routes through a shared helper, `mcphandlers/helpers.go:25`, whose own doc comment records
the design — *"The MCP audit middleware seeds name and namespace from the call's raw arguments,
but the UID exists only once the write has happened, so every create/update handler adds it
here"* — and names the test that enforces it, `TestMCPHandlers_WritesCallSetResource`. It landed
in **P1's own #4571** (`git log` on that file → `56b48aa7e`).

**So the "6 sites" figure predates the helper refactor within P1 itself.** Task 2.12 reduces to
the naming decision in D2 plus the deferred `reason` field. This is the single largest saving in
the phase, and it is only visible by counting the tree rather than reading the plan.

---

## 5. Item zero — collapse the two render paths first

> **✅ Done 2026-09-07.** `MarshalJSON` is the single definition (`types.go:148`) and
> `Logger.LogEvent` walks its output into attrs (`logger.go:62`). The section below is kept as
> the argument that motivated it, and one prediction it made needs correcting: the two paths had
> **already** diverged, not merely become able to. See the revision note at the top.

Not a field, and it must go first because it prices everything above it.

| Path | Builds | Cite |
|---|---|---|
| `Logger.LogEvent` | `slog` attrs, by hand, group by group | `logger.go:57-144` |
| `Event.MarshalJSON` | the JSON shape, independently, via `eventJSON`/`resourceJSON` | `types.go:106-182` |

They are held in agreement by **exactly one test**, `TestEvent_MarshalJSONMatchesLogEventShape`
(`logger_test.go:183`). Both paths independently reimplement the same non-trivial precedence
rule — the hierarchy's namespace unless `Resource.Namespace` is non-empty — and both carry a
comment saying so (`logger.go:96-100`, `types.go:158-164`).

**v1 adds 8 field-level changes. Through two hand-maintained paths that is 16 edits**, each with
a silent-divergence failure mode that one test stands between. The plan already raises the
question — *"Consider whether `schema_version` is the right moment to collapse them"* — and the
answer is yes, for a reason the plan doesn't give: `MarshalJSON` is **already** the sink wire-shape
contract for P5b, so the duplication isn't a tidiness problem, it's two competing definitions of
the published record.

**Recommendation — and pick the mechanism now, because the implementer meets it on day one.**
Make `MarshalJSON` the single definition and **derive the slog render from it: marshal the event,
then walk the resulting object into `slog.Group`s.** Preferred over a shared field table because
agreement becomes structural rather than tested — which is what lets exit criterion 2 retire
`TestEvent_MarshalJSONMatchesLogEventShape` as vacuous instead of maintaining it against eight
new fields. It also inherits `omitempty` and the namespace-precedence rule for free, and both are
currently reimplemented by hand in the slog path (`logger.go:91-133`).

The cost is one extra marshal per audit event. Acceptable here specifically: audit emission is
per-mutation, not per-request, and it already runs synchronously in the request goroutine after
`next.ServeHTTP` — so this is not a hot path. If that ever changes, the field-table alternative
is the fallback.

Do it as the *first* commit of P2, before any new field. It converts every row in §7 from two
edits to one.

---

## 6. Nine decisions v1 must take

Each is cheap now and a migration after P7 indexes the shape. D1–D5 were raised by the
evaluation; **D6–D9 were taken on 2026-09-07**, when implementation reached the point of
needing them.

**D1 — Does tenancy stay nested under `resource`, or move to the envelope? And what writes
`environment`?**

On the nesting, **recommend: keep it nested; do not migrate.** Three reasons. (a) `namespace` is
live on both surfaces — moving it is a breaking change to a field already being consumed, for
cosmetic consistency. (b) The nesting is not accidental: `Hierarchy` mirrors
`authz.ResourceHierarchy` field-for-field (`types.go:40-46`) so a record is directly comparable
to a policy scope, and that comparability is worth more than a flat envelope. (c) A `resource.*`
prefix indexes and queries as well as a top-level one, so P7 and P8 are indifferent.

> **⚠️ Superseded 2026-09-07 — the rest of D1 is wrong, and the correction shrinks the work.**
> The census in
> [`openchoreo-audit-environment-attribute-plan.md`](./openchoreo-audit-environment-attribute-plan.md) §2.2
> found a **second struct on the same path**: `CheckRequest` carries `Context authz.Context`
> beside `Hierarchy` (`internal/openchoreo-api/services/authz.go:20-23`), and
> `authz.Context.Resource.Environment` (`internal/authz/core/types.go:52-55`) is **already
> populated at 15 authorization call sites**. So a writer does exist — `AuthzChecker.Check`
> simply doesn't read it when it calls `audit.SetHierarchy` (`services/authz.go:48`).
>
> **Therefore:** `environment` goes on `Hierarchy` after all (still rendering as
> `resource.environment`, since both paths fold `Hierarchy` into the `resource` group), it needs
> **~4 lines in one function** rather than handler plumbing, it **survives denials**, and
> **task 2.6 / item 6 is retired, not required** — nothing seeds `Resource` pre-handler. The
> table below is kept because its `Hierarchy`-writer analysis is still correct as far as it goes;
> what it missed is that the writers have access to more than `ResourceHierarchy`.
>
> **Implemented 2026-09-07** exactly as corrected — `services/authz.go:52` and
> `observer/authz/helpers.go:116`. Everything from *"The reasoning that led there"* to the end of
> D1 describes the rejected design and is retained only as the record of why it was rejected.

**The reasoning that led there — correct about `Hierarchy`'s writers, wrong about what they can
see.** `Hierarchy` has exactly two writers, and neither takes `environment` *from
`ResourceHierarchy`*:

| Writer | Source | Cite |
|---|---|---|
| `SetHierarchy` | `AuthzChecker.Check`, from `authz.ResourceHierarchy` — **which has no `environment`** | `context.go:97`; callers `openchoreo-api/services/authz.go:48`, `observer/authz/helpers.go:111` |
| `SeedHierarchy` | MCP raw tool arguments, pre-authz — mirrors the same 4-field shape, and has **no REST caller** | `context.go:64`; sole caller `pkg/mcp/mcpaudit/audit_middleware.go:67` |

Putting `environment` on `Hierarchy` therefore needs either a third writer or an *authz-model*
change (environment is an authz **condition** today, not a hierarchy level) — well outside P2.
Putting it on `Resource` is the right home, since it is resource identity rather than an authz
scope. **And that revives task 2.6**, which §7 of the first draft of this document wrongly
dismissed: both adapters seed a `*Resource` pre-handler (`middleware.go:153`,
`mcpaudit/audit_middleware.go:57`), and `SetResource` **replaces that whole struct rather than
merging** — its own doc comment says so (`context.go:26-36`) — so a seeded `environment` is wiped
by any of the 72 handler calls that follow. That is precisely the bug `8d257ca9` fixed for
`namespace` by restating it at ~30 call sites.

**Recommend: make `SetResource` merge** (or add a `MergeResource` and migrate call sites to it)
as part of v1, *before* `environment` is seeded. It is a change to one function, it is what makes
any pre-handler-seeded tenancy field survive, and the alternative — restating `environment` at
every call site that touches an environment-scoped resource — repeats a bug the repo has already
paid for once. **Task 2.6 is in scope; only its framing changes** from "migrate handlers off
metadata" to "fix the replace semantics that block seeding."

**D2 — `resource.id` holds a UID. Rename it, or document it?**
EM-14 wants `uid` *alongside* `id` so a delete-then-recreate is distinguishable, on the
assumption that `id` is a logical identifier. Verified: `id` is `string(obj.UID)`, and there is
no third identifier — in Kubernetes the logical identifier *is* the name, which
`resource.name` already carries at all 79 sites. So EM-14's intent maps onto **`name`
(present always) + `uid` (generation-distinguishing)**, with `name` playing the role EM-14 called
`id`.

**Recommend: rename `Resource.ID` → `Resource.UID`, wire tag `uid`.** Mechanical, entirely
compiler-caught, zero behaviour change, and it makes the schema reference and the OCSF mapping
honest. Keeping `id` and documenting it as the UID is the cheaper alternative but exports a name
every SIEM integrator maps wrongly once and then works around forever.

> **Refined 2026-09-07 — what makes a value a `uid` is "never reused", not "comes from Kubernetes".**
> Review of the implementation caught observer publishing `uid` = the incident path parameter,
> byte-identical to `name` (`alerts_query.go:193`), against a contract that said "the Kubernetes
> object UID". Checked rather than assumed: the incident id is
> **`uuid.NewString()`** (`internal/observer/store/incidententry/sql.go:138`), and
> `openapi/observer-api.yaml:1769`'s `Incident` has an `incidentId` and **no name field**. So it
> is server-generated and never reused — exactly the property EM-14's delete-then-recreate
> guarantee needs. **The code was right; D2's wording was too narrow.**
>
> `uid` is therefore defined as *a server-generated identifier that is never reused* — a
> Kubernetes object UID for control-plane resources, a generated UUID for observer incidents —
> and `name` as *the identifier the operation addresses the resource by*, **which equals `uid`
> for a resource that has no user-chosen name.** Recorded as a property, not hidden: the
> alternative, omitting `name`, would publish the same value under `name` on a denial (from the
> pre-handler path seed at `middleware.go:154`) and under `uid` on success — forcing a consumer
> to check two fields for one identifier.

**The residual gap this exposes — record it, don't paper over it.** 7 of 79 REST sites pass
`Name` only: `git_secrets.go:98`, `dataplanes.go:200`, `exec.go:86`, `wirelogs.go:139`,
`environments.go:200`, `secrets.go:179`, `projects.go:194` — i.e. **deletes and non-CRUD
mutations, where the service method returns only an error.** So a delete event names the object
but not the generation deleted. EM-14 is still met, but *asymmetrically*: a delete-then-recreate
is distinguishable by comparing the two creates' UIDs, not by reading the delete event.
Closing it means delete service methods returning the deleted object's UID — a service-layer
change, out of P2's scope. **v1 must document `resource.uid` as optional and say exactly which
operation classes omit it**, or P7's read path will assume a stable UID that deletes don't carry.

**D3 — Are `http.method`/`http.path` required?**
They cannot be: MCP events have no request line. **Recommend: an optional `http` group, present
only when `origin: api`,** with the schema reference stating that MCP omits it and that
**`operation_id` is the cross-surface join key** (it already is — P0 ships it on both surfaces).
Marking them required would make every MCP event schema-invalid; marking them core-and-optional
leaves EM-2 partially met, which is the correct outcome and should be recorded as such rather
than quietly satisfied.

> **⚠️ Refined on implementing (2026-09-07) — the presence rule is not keyed on `origin`.**
> `NewUnauthenticatedMiddleware` guards `/mcp` as well as the REST routes, and a token rejection
> there **is** an HTTP-boundary event with a real request line (`POST /mcp`). Keying presence on
> `origin: api` would have forced us to discard a truthful value, so the implemented rule is:
>
> **`http` is present when the event was produced at an HTTP boundary; it is absent for MCP
> `tools/call` operations, which have no request line of their own** — one HTTP request can carry
> several, so the transport's method and path describe none of them in particular.
>
> Consequences: `http` is optional on **both** origins, a consumer must not infer its presence
> from `origin`, and the omission class exit criterion 3 names is *"MCP tool-call operations"*
> rather than *"origin: mcp"*.
>
> **Pinned by the pair that actually demonstrates it** — `TestPublishedRecordShape/mcp_tool_call_omits_the_http_group`
> (origin `mcp`, no `http`) and `/mcp_rejection_at_the_http_boundary_carries_the_http_group`
> (origin `mcp`, `http` present). *An earlier draft of this note cited
> `/unauthenticated_with_no_operation` as the second half; it sets no `Origin` at all, so it
> demonstrated `http`-without-`operation_id` but not the origin-independence that is the rule.*

**D4 — `resource.metadata`: reserved, or dropped from v1?**
Both metadata maps have no writer and are `omitempty`, so **they never appear on the wire today** —
"dropping" them is a no-op in code and the whole decision is about the schema reference.
**Recommend: document as reserved and do not list them as v1 fields.** Publishing a free-form
`map[string]any` in a frozen schema invites a consumer to build an index mapping for it, and P3
then arrives to govern a field that already has an unversioned de-facto shape. P1's non-goal
here survived 79 new call sites (its own record calls this *"the non-goal most at risk"*) —
v1 shouldn't undo that by documentation.

> ### ❌ D5 is retired (2026-09-07) — the rename was wrong, and its own rule is what disproved it
>
> D5 argued `service: "observer"` should become `"observer-api"`, because `6739d1bd9`'s commit
> message, the coverage matrix and the package layout all say *observer-api*. **Implemented, then
> reverted.** The rule D5 itself sends into the schema reference — *match the deployed service
> name* — holds for only one of its two examples:
>
> | | Deployed name | `producer` under D5 |
> |---|---|---|
> | openchoreo-api | `openchoreo-api` (`_helpers.tpl:216`) | `openchoreo-api` ✓ |
> | observer | `observer` (`observer-deployment.yaml:10`, `observer-service.yaml:4`) | `observer-api` ✗ |
>
> The in-repo tooling agrees with the deployment, not the rename: both
> `tools/auditgen/services.go:56` and `tools/auditcoverage/services.go:46` key the service as
> `observer`. What *is* named `observer-api` is the **API surface** — `openapi/observer-api.yaml`,
> `internal/observer/api/`, `docs/audit/observer-coverage-matrix.md` — which is what misled the
> original analysis.
>
> **Resolved: `producer` names the deployed workload, not the API it serves.** P5a routes on this
> value and P7 indexes it, and an operator correlating an audit event with a pod expects one
> string to find both. `emitter.go`'s doc comment now states that rule with the corrected
> examples. **`service: observer` was right all along**, so the one *breaking* change D5 warned
> about never needed making.

**D5 — Settle the canonical producer names.** *(Superseded — see above.)*
`"openchoreo-api"` (`cmd/openchoreo-api/main.go:213`) and `"observer"`
(`cmd/observer/main.go:591`) — while `6739d1bd9`'s own message, the coverage matrix and the
package layout all say **observer-api**. P5a routes on `producer` and P7 indexes it, so this is a
value that becomes load-bearing two phases from now. **Recommend: fix to `observer-api` as part
of task 2.2's rename**, where the field is being touched anyway, and state the naming rule
(match the deployed service name) in the schema reference.

**This is a breaking change filed as a cleanup, and the distinction matters.** `service:
observer` has been emitting since `6739d1bd9` (2026-09-02), so any saved query or k3d-log grep
already written against it breaks. It is cheap *only because* there is no downstream consumer
yet — the same reason §4 gives for the `service` → `producer` rename itself. Read as cosmetic and
deferred, it becomes the one outcome that makes it expensive: a value P7 has indexed.

**D6 — `schema_version` is a string, `"1.0"`.**
The document contradicted itself: §7 item 1 wrote `schema_version: 1`, §10.1 recommended `"1.0"`.
**Resolved to the string.** An integer has no way to express "additive change", so every field in
§8's deferred register would either bump to `2` — breaking consumers for an addition — or bump
nothing, leaving the version unable to describe the shape it stamps. The string carries
CloudTrail's rule, quoted verbatim in the reference: **major** on a field removal or a change to
how a value is represented; **minor** on an addition, which consumers may ignore. Per §10.1 there
is no third component, deliberately.

**This is the field whose *type* is least retrofittable of all** — a consumer that parses
`schema_version` as a number cannot be asked to re-parse it as a string without the major bump the
field exists to announce.

**D7 — `http.path` publishes the request path as sent, without the query string.**
Three candidates, settled in that order over 2026-09-07: the route pattern
(`/namespaces/{namespaceName}/projects/{projectName}`), the raw request URI, and — where it
landed — **`r.URL.Path`: the real path values, query string excluded.**

**Why not the route pattern.** It cannot reconstruct what the client actually sent, and the
pattern is already recoverable from `operation_id`. It would have made the field near-redundant.

**Why not the raw URI.** The query string is the only part of an audit record that is
**free-form client input nothing validates or reads** — and, being unread, it is also the only
part with no counterpart anywhere else in the record. It is what forced both of D7's original
obligations: a length cap, and a new potentially-identifying classification for item 14.

**Why the path is the right cut.** Every identifier in it — namespace, project, component,
resource name — is *already published* in the `resource` group. So the path introduces **no value
the record doesn't already carry, and no exposure the resource fields don't already have.** That
collapses both obligations: nothing new for the PII register, and no cap needed that the
`resource.*` fields don't equally need.

**The value is the *decoded* path** (`r.URL.Path`, not `EscapedPath`), which is what makes that
argument hold: `r.PathValue` returns decoded segments too, so `http.path` and `resource.namespace`
agree on encoding. The cost is that percent-encoding the client chose is not recoverable from the
record — `GET /api/v1/namespaces/ns%2Devil%0Aline/projects` is published with the escapes
resolved. Item 12 must describe the field as **"the decoded request path, query string
excluded"**, not "as sent".

**What it costs, recorded honestly.** A request's query parameters are not in the audit trail —
so `?dryRun=true`, and anything else that changes an operation's meaning through the query, is
invisible to a consumer. EM-2 is therefore met for the request *line* but not the request in
full, and item 12 should say so. The access log, joined by `request_id`, is where the full URI
lives.

> **Item 14 obligation, discharged.** D7 originally required a PII entry for `http.path` to land
> in the same change as the field. With the query excluded, **there is nothing new to classify** —
> the path's identifiers are the same ones `resource.namespace`/`project`/`component`/`name`
> already publish, so item 14 covers them where it covers those. The `absent actor.assurance
> means validated` rule that §8's deferral owes item 12 is still outstanding.

> ### 🔬 A length cap was implemented, then removed (2026-09-07) — and the probe found something else
>
> D7 first required the recorded URI to be length-capped, on `RequestIDFromHeader`'s precedent
> (`adapter.go`, whose doc comment warns that without validation "a client could inflate every
> audit record for its request with an oversized or arbitrary string"). **The cap was built, then
> dropped** — and dropping the query string later removed most of what it was guarding. Three
> things a probe through the *production* middleware chain established, none of which were
> assumptions:
>
> | Claim | Result |
> |---|---|
> | Only registered operations are audited, so an arbitrary URI never reaches a record | **Confirmed.** `gen.HandlerWithOptions` applies the audit middlewares **per registered route**, so an unregistered path 404s in `ServeMux` with no audit event |
> | Proper path-parameter validation prevents an oversized value being recorded | **False, structurally.** Capture happens at middleware entry (`middleware.go:153`), *before* validation, and a rejected request is **still audited** — that is the point. `POST /api/v1/namespaces/<20,000 chars>/projects` produced one audit record |
> | The exposure is confined to `http.path` | **No.** In the same probe `resource.namespace` was recorded at **20,000 characters**, seeded from `r.PathValue` at `middleware.go:154`. That gap **predates P2** — it arrived with P0/P1 — and no cap was ever proposed for it |
>
> **Why the cap was dropped, stated correctly.** *Not* "validation prevents it" — row 2 disproves
> that, and the reference must not claim it. The operative reason is **layer**: input hygiene
> belongs where validation lives, or in `MaxHeaderBytes`, not in the audit adapter, whose job is
> to record faithfully what arrived. Two supporting facts: capping `http.path` while `resource.*`
> stayed unbounded was half a bound; and truncating an identity field would be *worse* than
> recording a long one, since a truncated name is a **wrong** name in an audit trail.
>
> **Excluding the query then removed the asymmetry the cap existed for.** With only the path
> recorded, `http.path`'s exposure is exactly `resource.namespace`'s — a long path segment — and
> that is one pre-existing property to document, not a new one this phase introduced.
>
> **The real ceiling, and it must be documented rather than claimed away.** Neither server sets
> `MaxHeaderBytes` (`internal/server/server.go:48`, `cmd/observer/main.go:364`), so Go's 1 MB
> default bounds the request line — the honest limit for `http.path` and, roughly, for
> `resource.namespace`/`name`. **Item 12's limitations section must state this exposure**,
> alongside §8's other documented holes. Anyone wanting a real bound should set `MaxHeaderBytes`
> or tighten request validation — both outside the audit package.

**D8 — `event_time` is a new capture; `received_time` inherits today's value.**
Verified, and it changes what task 2.10 is: today's `timestamp` is **not** "when the action
occurred" as `types.go:92` claims. `buildEvent` stamps `time.Now()` (`emitter.go:83`) and the REST
adapter emits from a `defer` *after* `next.ServeHTTP` (`middleware.go:183`), so the value is
response-complete time — and for a handler that hijacks the connection (exec, wirelogs) it can be
minutes after the action began. A pure rename to `event_time` would have frozen that inaccuracy
into a field the reference promises is action time.

**So:** `event_time` is captured at each surface adapter's **entry**, before the handler runs, and
carried forward; `received_time` keeps the existing emit-time stamp. Both are pinned in the
reference as OCSF's pair (§10.2): `event_time` ↔ `original_time`, `received_time` ↔ `logged_time`,
with `received_time` defined as **"accepted by the producer's emitter"** — *not* collector-ingest
time, which is what a SIEM consumer will otherwise assume, computing garbage latency from it. On
an ordinary REST request the two differ by microseconds; the pair only carries information for
hijacked connections today, and for a real collector hop after P4.

> ### ⚠️ Revised again (2026-09-07) — `received_time` is deferred; v1 ships one timestamp
>
> **What survives from D8 is the half that matters: the rename and the semantic fix.** Today's
> `timestamp` is response-complete time labelled as action time, and correcting that — capturing
> at adapter entry, publishing it as `event_time` — is a **reshape**, hence a major bump under D6
> and hence v1's business. **Adding a second timestamp later is an addition**, a minor bump; an
> event that carries no pipeline time simply hasn't recorded a pipeline hop.
>
> §4's original argument ("a second timestamp added later leaves every stored event with a null")
> conflated the two. A null in an *optional additive* field is what additive-safe means; the
> ambiguity it warned about lives in the **name** `timestamp`, and the rename cures that
> regardless of whether a sibling ships.
>
> **`received_time` fails the same test D9 applied to `sequence`, for the same reason.** OCSF
> defines `logged_time` as *"when the event reached its final destination"* — a **collector's**
> stamp. A producer-stamped `received_time` is "when my own emitter accepted my own event",
> microseconds after `event_time`, and it approximates a field that properly belongs to a
> component **P4 has not built**. Shipping the producer's guess at it two phases early is exactly
> what D9 declined to do for the sequence counter.
>
> **And two timestamps microseconds apart are an active footgun.** A consumer will compute
> latency from the delta. Given D8's own finding — `event_time` is post-auth, the emit stamp is
> post-handler — that delta is a partial handler duration, not pipeline latency. One well-defined
> timestamp is better than two that invite a wrong subtraction.
>
> **What v1 gives up, recorded.** For a hijacked connection (exec, wirelogs) the gap between
> arrival and emission is the session's rough duration, and it is no longer visible. That is
> worth having eventually — but as an explicit `duration` field, not inferred from two timestamps
> with different definitions.
>
> **One simplification falls out:** `buildEvent` now reads **no clock at all** (`emitter.go`).
> Every time value on an event comes from the single capture at the adapter's entry, so there is
> one clock read per event, in one place.

> **⚠️ Precision added on implementing (2026-09-07) — "entry" means the audit adapter's entry,
> not the process's.** `Middleware` sits *inside* the auth middleware — `NewUnauthenticatedMiddleware`
> must wrap auth rather than nest inside it, since auth short-circuits a rejected request — so on
> an **authenticated** request `event_time` is stamped *after* token validation, which can include
> a cold JWKS fetch (`jwt/middleware.go:124`) and is then not negligible. The unauthenticated
> middleware, being outside auth, does stamp at true entry, so **the two paths measure from
> different points.**
>
> The field is therefore defined as **"when the audit adapter received the request"** — the
> producer's view of when the operation began — and item 12 must publish that wording rather than
> "when the action occurred". The gap to socket-accept time is not knowable in this layer and
> belongs to the access log, which `request_id` already joins to. Moving the capture into the
> outer middleware and sharing it inward was considered and rejected: it would add a second
> context-plumbing mechanism alongside `emittedMarker` to close a difference the reference can
> simply state.

**D9 — `producer_instance` and `sequence` defer to P6, as a pair.**
They are one mechanism: `sequence` alone is ambiguous across replicas (three instances each
counting from 1 make the merged stream look full of duplicates and gaps) and `producer_instance`
alone is a random UUID with nothing to join to. So they move together or not at all; splitting
them is the one option that cannot be right.

**Three reasons they move.** (a) §4 justified `sequence` on marginal cost, and **item 0 removed
that argument** — with one render definition, adding it in P6 costs what it costs now. (b) **The
transport they protect does not exist**: P4 and P5a have not started, the only sink is an
in-process `slog` writer to stdout, so there is no hop at which an event can be lost. (c) The
mechanism is **incomplete without P6's heartbeat or emitted-count**: the pair detects head loss
(a run must start at `1`) and interior gaps, but **not tail loss** — a received `1..80` of an
emitted `1..100` looks complete, as does a crash between the counter increment and the sink
write. §8 already conceded SE-5 is only "partial" for this reason.

**And deferring is strictly safer than shipping half.** Adding `sequence` later is a **minor**
bump under D6; changing what it *means* later — because the detector we build in P6 wanted a
different shape — is a **major** one. Designing the producer half two phases before its only
consumer is how that happens.

**One thing this preserves.** `Emitter`'s doc comment says *"Immutable after construction"*
(`emitter.go:24-25`), and it is load-bearing under concurrent MCP tool calls. A `sequence` counter
would have made that sentence false — safely, via `atomic.Uint64`, but false — and required
rewriting a comment a future reader would otherwise trust.

---

## 7. The recommended minimal v1

**Ordered.** Item zero is a prerequisite, not a parallel task.

| # | Work | Task | Kind |
|---|---|---|---|
| ~~**0**~~ | **Collapse the two render paths into one definition** — **✅ shipped 2026-09-07** | 2.1 pre | prerequisite (§5) |
| 1 | `schema_version: "1.0"` + documented bump policy | 2.1, D6 | new |
| 2 | `service` → `producer`. **`observer` → `observer-api` retired** (D5 was wrong; the value already matched the deployment) and **`producer_instance` dropped** (D9) | 2.2 | rename |
| 3 | `timestamp` → **`event_time`, newly captured at adapter entry**. `received_time` cut — D8 as revised | 2.10 | reshape |
| ~~4~~ | ~~`audit_id`, minted per audited operation~~ — **deferred to P10a 2026-09-08**, to land with the pre-action event that gives it a second record to group | ~~2.11~~ | — |
| 5 | `Resource.ID` → `Resource.UID` (`uid` on the wire); **document which operation classes omit it** | 2.12, D2 | rename |
| ~~6~~ | ~~Make `SetResource` merge rather than replace~~ — **retired 2026-09-07**, not deferred: the corrected design never seeds `Resource` pre-handler. See the D1 supersession note | ~~2.6~~ | — |
| ~~7~~ | Add `environment` to `Hierarchy`, populated in `AuthzChecker.Check` — renders as `resource.environment` — **✅ shipped 2026-09-07** | 2.5 | new + decision |
| 8 | `actor.session_id`, `actor.issuer` — promoted onto `auth.SubjectContext` at `resolver.go:93`, **not** read from a claims map in `ExtractActor` (revision 2, finding 1); **document `actor.id` as the `sub` claim** instead of adding a duplicate `actor.subject` | 2.7 | new |
| ~~9~~ | ~~`actor.assurance`, constant `validated`~~ — **deferred to P9 2026-09-07:** `assurance == "validated"` ⟺ `actor.type != "anonymous"`, so it restates a published field | ~~2.9a~~ | — |
| 10 | Optional `http` group (`method`, `path`), `origin: api` only; `path` is the raw URI, capped — D7 | 2.3, D3 | new, optional |
| ~~11~~ | ~~`sequence`, scoped to `producer` + `producer_instance`~~ — **deferred to P6 2026-09-07**, with `producer_instance`. See D9 | ~~2.18~~ | — |
| 12 | **Generated schema reference** — including D2's, D3's and D4's recorded limits | 2.14 | doc |
| 13 | IdP correlation guide | 2.16 | doc |
| 14 | PII register, **covering `actor.entitlements`** | 2.17 | doc |

> ### ✅ Items 0, 6 and 7 are closed (2026-09-07)
>
> **[`openchoreo-audit-environment-attribute-plan.md`](./openchoreo-audit-environment-attribute-plan.md)**
> owned the render-path collapse (item 0) and the `environment` attribute (item 7); both are
> implemented on `feat/audit-environment-attribute`. Item 6 (`SetResource` merge) is **retired**
> — `environment` lives on `Hierarchy`, which `SetResource` cannot erase.
>
> They were sequenced ahead of P2 rather than deferred, because P2 cannot freeze a schema whose
> tenancy is incomplete, and should not add eight fields through two hand-maintained render paths.
>
> **Items 1–5, 8, 10 and 12–14 remain** — each now a single edit rather than two. Items 9 and 11
> were retired later the same day for a different reason (D9 and §8), not by this work.

**8 items remain** (1–5, 8, 10, 12–14 less the two struck), of which 3 are documentation and 3
are renames of live fields. Only **1 is a genuinely new envelope field** — `schema_version` —
plus the `event_time` capture, the two `actor.*` claims and the optional `http` group.

The phase is far smaller than its 18-row task table implies, and it shrank three times: task
2.12 turned out to be delivered, task 2.5 turned out to be one field rather than four, and
**applying §1's additive-safe test consistently retired items 9 and 11** along with
`producer_instance`. What remains is the set that cannot be added later without a major bump or a
permanently thinner record.

---

## 8. Deferred and blocked register

Recording these is part of the deliverable — an undocumented omission reads as an oversight.

| Item | Task | Status | Reason |
|---|---|---|---|
| `producer_instance` + `sequence` | 2.2, 2.18 | **deferred to P6** (2026-09-07) | Additive-safe under D6. One mechanism, so they move together. The transport they protect doesn't exist yet, and the mechanism is incomplete without P6's heartbeat — which is also what makes P6 the phase that should shape them. D9 |
| `received_time` | 2.10 | **deferred to P4/P5a** (2026-09-07) | Additive. OCSF's `logged_time` is a **collector's** stamp, so the useful version of this field belongs to the collection pipeline P4 builds — a producer-side one is a guess at it, microseconds from `event_time`, that invites a wrong latency calculation. The v1-critical half of task 2.10 (the rename and re-capture) shipped. D8 |
| `audit_id` | 2.11 | **deferred to P10a** (2026-09-08) | Additive, and 1:1 with `event_id` until a second event per operation exists. It ships with the pre-action event that needs it. Consumer rule until then: **an event is its own group.** EM-13a's rejection of `request_id` as a substitute is recorded for whoever implements it |
| `actor.assurance` | 2.9a | **deferred to P9** (2026-09-07) | Additive-safe, and a constant that restates `actor.type` today. Belongs with `on_behalf_of`, the field that gives `asserted` a meaning. **The reference must record: absent `actor.assurance` means `validated`** |
| `actor.on_behalf_of` | 2.9 | **deferred** | Additive-safe; no producer can populate it before P9 |
| `reason` | 2.12 | **deferred** | Additive-safe; a `Could`, with no break-glass surface to carry it |
| `origin: client` | 2.4 | **deferred** | Enum widening is additive; P9 introduces the value |
| `request_id` ← `traceparent` | 2.13 | **⛔ blocked** | P4 not started; no trace exists to join (§2.11). Not a P2 choice |
| OCSF `api_activity` mapping | 2.15 | **⛔ blocked** | Scope §4.9.2 open. Also better written after D2/D5 rename the fields it maps |
| `result: pending`, two-event model | — | out of phase | P10a, as planned — **and it now owns `audit_id` too** (2026-09-08), since the key and the second event it groups belong in one change |
| Attribute registry | — | out of phase | P3, as planned. See D4 |
| `sequence` metrics, drop counter, degraded condition, heartbeat | — | Track 2 | P6 — **and it now owns the counter too** (D9). v1 ships **none** of this, so SE-5 is unmet rather than partial until P6 |

**Known holes v1 should document rather than fix** (all verified in P1's Appendix E):
`source_ip` is unobtainable for direct MCP clients (`RequestExtra` has no `RemoteAddr`); both
`source_ip` and `X-Request-ID` are taken from headers with no trusted-peer check; and on an MCP
session opened with `?filterByAuthz=false` a refusal records `failure`, not `denied`
(`pkg/mcp/mcpaudit/audit_middleware.go:86-98`) — which means a frozen v1 promising a reliable
`result: denied` would be over-promising to P7's read path. Each belongs in the schema
reference's limitations section.

**Three more, found on implementing (2026-09-07):**

- **No completeness guarantee at all, and v1 should say so plainly.** With `sequence` deferred
  (D9), nothing in a v1 record lets a consumer detect a lost event. Even once P6 ships the
  counter, **tail loss stays undetectable** without a heartbeat: a received run of `1..80` from an
  instance that emitted `1..100` is indistinguishable from a complete one, and so is a crash
  between the counter increment and the sink write. The reference should promise gap *detection*
  only for head and interior loss, and only from P6 — never completeness.
- **`actor.session_id` may be empty in every deployment.** `sid` is optional in OIDC, and nothing
  in the tree reads, configures or tests it — the bundled Thunder IdP is unverified. The field
  ships, then gets checked against a live token; if absent, the reference names it a documented
  omission class and item 13's IdP guide says which providers populate it. `iss` is guaranteed by
  OIDC, so `actor.issuer` has no such caveat.
- **`http.path` and `resource.namespace`/`name` carry unbounded client input, by design.** All are
  captured before validation runs and recorded even when the request is then rejected, so the only
  ceiling is `net/http`'s `MaxHeaderBytes` — 1 MB, which neither server overrides. A length cap on
  `http.path` was built and removed: input hygiene belongs in validation or `MaxHeaderBytes`, not
  in the audit adapter, and truncating an identity field would record the wrong object. The
  `resource.*` half **predates P2** (P0/P1). See D7's probe note.
- **Query parameters are not audited.** `http.path` excludes the query string (D7), so a
  parameter that changes an operation's meaning is invisible in the trail. The access log, joined
  by `request_id`, carries the full URI.

---

## 9. Exit criteria

The plan's P2 exit criteria, corrected for what this evaluation found:

1. `schema_version` is the string `"1.0"`, with CloudTrail's major/minor rule documented verbatim
   (D6).
2. ✅ **Met 2026-09-07.** One render path produces the published record, and the agreement test
   is retired as vacuous — replaced by a byte-exact golden test on the rendered line.
3. Every §7 field emits on **both** surfaces, or its absence is documented with the class it
   applies to. The sanctioned cases: `http.*` on **MCP tool-call operations** (D3, as refined —
   not on `origin: mcp` as a whole), `resource.uid` on deletes and non-CRUD mutations (D2),
   `resource.environment` on operations whose authorization is not environment-scoped, and
   **`actor.session_id` where the IdP issues no `sid`** (§8).
3a. ~~A seeded `environment` survives a handler `SetResource` call~~ — **moot.** The merge fix was
   retired with item 6; `environment` is on `Hierarchy`, which `SetResource` cannot reach.
   Replaced by: **a denied create records `resource.environment`** — the property that chose the
   `Hierarchy` design, covered by `TestCheck_RecordsEnvironmentRegardlessOfDecision`.
4. `make audit.schema-docs` regenerates `docs/audit/schema-reference.md`, and a field added
   without a reference entry fails CI — the same gate shape P1 and P1.5b already use.
5. The §6 decisions are recorded **in the repository**, not only here — including D9's reasoning,
   so P6 inherits why the counter is its to design rather than finding a gap, and D5's
   retirement, so nobody re-proposes the `observer-api` rename.
6. ~~"A `traceparent` request joins an existing trace"~~ — **removed. Blocked on P4**, and
   keeping it makes P2 unclosable.
7. ~~"No handler writes tenancy through `AddMetadata`"~~ — **already vacuous**; the plan's own
   2026-08-24 note verified the helper no longer exists.

---

## 10. Prior art — what the versioning buys, and who else does this

Checked against four external audit/event schemas (fetched 2026-09-04, not recalled). Two
results matter: the **bump policy** P2 task 2.1 needs is already written down by CloudTrail, and
**OCSF's `metadata` object independently converged on almost exactly P2's envelope additions** —
which is the strongest available evidence that the §7 minimal set is the right set rather than
merely a defensible one.

### 10.1 The bump policy — take CloudTrail's, it is the axis this document already used

CloudTrail's `eventVersion` is `major.minor` with an explicit rule: **major** increments on a
change that is *not* backward-compatible — removing an existing field, or changing how a field's
contents are represented (their example: a date format) — and **minor** increments when new
fields are added. *"Applications can ignore new fields to stay compatible with new minor
versions."* New event types with an unchanged structure bump nothing.

That rule **is** §1's v1-critical/additive-safe test, arrived at independently: renames and
reshapes are major, additions are minor. So the axis used to cut the minimal set is not a local
invention, and P2 can adopt a policy with a decade of operational history behind it rather than
draft one.

CloudEvents makes the same major/minor choice and adds one refinement worth copying: `specversion`
carries **only major and minor**, deliberately, so patch-level spec changes never force
already-serialized events to be rewritten.

**Recommendation:** `schema_version: "1.0"`, with CloudTrail's major/minor rule documented
verbatim in the schema reference, and the OCSF-style SemVer left alone — a third component we
would never bump adds no information.

### 10.2 OCSF `metadata` — near 1:1 with P2's envelope

EM-17 already targets OCSF `api_activity` (class_uid `6003`). Its required `metadata` object
turns out to contain counterparts for nearly every envelope field P2 proposes:

| P2 / this document | OCSF `metadata` | OCSF requirement |
|---|---|---|
| `schema_version` | `version` (SemVer) | **Required** |
| `event_id` ✅ *(already emitting)* | `uid` | Optional |
| `audit_id` | `correlation_uid` — *"correlate this event with other related events"* | Optional |
| `sequence` | `sequence` — *"establishing exact ordering regardless of time precision"* | Optional |
| `event_time` | `original_time` — *"as reported by the event source"* | Recommended |
| `received_time` | `logged_time` — *"when the event reached its final destination"* | Optional |
| `producer` | `product` | **Required** |
| `producer_instance` | — *(no counterpart; closest are `log_provider`/`log_name`)* | — |

Every v1-critical envelope field except `producer_instance` has an OCSF home. Two consequences:
**EM-17's mapping (task 2.15) gets substantially cheaper**, and the one field with no counterpart
is the one to justify most carefully — it exists for RE-7's restart-vs-gap distinction, which
OCSF leaves to the producer.

> **Revised 2026-09-07.** That last observation aged into a decision. `producer_instance` and
> `sequence` are **deferred to P6** (D9), so v1's envelope now maps **1:1 onto OCSF `metadata`**
> with nothing left over — every field it publishes has a counterpart. The field that stood out
> as needing the most justification turned out to be the field that shouldn't be in v1.

Note also OCSF's `original_time`/`logged_time`/`processed_time` triple: it draws the same
producer-time vs pipeline-time line EM-13 asks for, and adds a third for intermediate hops. A
log-collection pipeline (P4/P5a) is exactly such a hop, so **`received_time`'s semantics should
be pinned now** — "accepted by the producer's emitter" or "accepted by the collector" — because
OCSF distinguishes them and a SIEM consumer will expect the distinction.

> **Note (2026-09-08).** The table above maps what §P2 *proposed*. Four of its rows are now
> deferred — `audit_id` (→ P10a), `sequence` and `producer_instance` (→ P6), `received_time`
> (→ P4/P5a) — so v1's envelope maps onto `version`, `uid`, `original_time` and `product` only.
> Each deferred field's OCSF counterpart is recorded here for whoever adds it.

> **Revised 2026-09-07 — this question answered itself, and the answer removed the field.**
> Forced to choose, the honest reading of OCSF is that `logged_time` means *"accepted by the
> collector"* — its own gloss is "when the event reached its **final destination**". That
> destination is P4's collector, which does not exist, so v1 could only have published the other
> thing under a name consumers would read as the first. **`received_time` is deferred to P4/P5a**
> (D8's second revision), leaving v1's timestamp mapping to `original_time` alone, which is the
> half a producer is actually entitled to stamp.
>
> With `producer_instance` and `audit_id` also gone, **v1's envelope maps onto OCSF `metadata`
> with nothing left over and nothing overreaching**: `version`, `uid`, `original_time`, `product`.

### 10.3 Kubernetes audit — the closest structural sibling, and it validates three decisions

`audit.k8s.io/v1` versions the event with `apiVersion` + `kind` on **every record**, and its field
set independently supports three calls made above:

| Kubernetes field | Bears on |
|---|---|
| `auditID` — shared across the `RequestReceived`/`ResponseComplete` events of one request | **D-none; confirms EM-13a.** The scope doc says `audit_id` "mirrors Kubernetes' auditID" and it does, exactly |
| `stage` ∈ `RequestReceived, ResponseStarted, ResponseComplete, Panic` | **P10a.** K8s uses an explicit `stage`; EM-13b instead overloads `result: pending`. Worth revisiting in P10a — K8s needed four stages, and `pending` only expresses two |
| `objectRef.name` **and** `objectRef.uid` as separate fields, with **no third identifier** | **D2, directly.** K8s models a resource as name + uid, which is precisely the `name` + `uid` pair D2 recommends. It is the reference implementation of the argument that EM-14's third `id` field should not exist |
| `requestReceivedTimestamp` + `stageTimestamp` | **EM-13.** Same two-timestamp split |
| `requestURI` + `verb` | **D3.** K8s records the request line as first-class fields — supporting §4's row that method and path belong in v1 |
| `sourceIPs`, documented as *"all but the last IP can be arbitrarily set by client"* | **§8's known holes.** K8s **documents** the header-trust weakness rather than fixing it. That is the precedent for documenting our `source_ip`/`X-Request-ID` trust assumptions in the schema reference instead of treating them as defects |
| `annotations map[string]string`, with keys required to uniquely identify the setting component (e.g. `podsecuritypolicy.admission.k8s.io/policy`) | **D4 and P3.** Even K8s' free-form map carries a namespacing *convention*. If `resource.metadata` is ever published, P3 should require a similar qualified-key rule |

`userAgent` is also marked *"client-provided, not trusted"* in the reference — the same
document-the-trust-boundary discipline.

### 10.4 What the versioning actually buys

Concretely, four things, in the order they will be needed:

1. **A consumer can pin.** P7's index mapping, P8's table and any external SIEM can declare "I
   read 1.x" and be guaranteed additions won't break them. Without a version they must
   defensively probe for field presence forever.
2. **It makes the additive contract enforceable rather than aspirational.** Once the major/minor
   rule is published, a PR that renames or removes a field is a *visible* major bump in review,
   not a silent wire-shape change. That is the same enforcement logic the audit coverage gate
   already applies to coverage — this document's §3 shows the gates are what kept this feature
   alive.
3. **It is the only safe route to the deferred work.** `on_behalf_of`, `reason`,
   `origin: client`, P3's attribute registry and P10a's second event are all *additions* — every
   one lands as a minor bump under this policy. That is precisely why §8 can defer them without
   risk, and the version is what makes that promise checkable.
4. **It dates the record.** Audit trails are read years later. A stored event that cannot say
   which schema produced it cannot be interpreted after the schema moves — the failure mode
   §4 gives for why `schema_version` is the one field with no retrofit path.

**Sources:** [Kubernetes audit Event v1 reference](https://kubernetes.io/docs/reference/config-api/apiserver-audit.v1/) ·
[CloudTrail record contents](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-event-reference-record-contents.html) ·
[OCSF `metadata` object](https://schema.ocsf.io/objects/metadata) ·
[OCSF `api_activity`](https://schema.ocsf.io/classes/api_activity) ·
[CloudEvents spec](https://github.com/cloudevents/spec/blob/main/cloudevents/spec.md)

---

## Appendix — corrections to the phased plan

Fold these back into `openchoreo-audit-logging-phased-plan.md` §P2 so the next reader isn't
misled.

| Plan says | Verified reality |
|---|---|
| 2.12: `resource.uid` at *"78 REST sites but only 6 MCP sites"*; P2 closes the **surface-parity** gap | **That gap is closed already.** 79 REST, **55** MCP via `mcphandlers/helpers.go:25`, test-enforced, landed in P1's #4571. What remains is a **coverage** gap of a different kind — 7 REST sites carry no UID (D2) |
| 2.5/deliverable 4: promoting tenancy is *"the bulk of the diff"* | With D1 it is **one new field** (`environment`). The other three already emit |
| 2.6: decide whether `SetResource` should merge | **Retired 2026-09-07.** The plan's stated reason ("migrate handlers off metadata") is vacuous, and the replacement reason this document gave was wrong — `environment` goes on `Hierarchy`, which `SetResource` cannot erase by construction |
| EM-5's four tenancy fields | Three exist. `environment` is absent from `Hierarchy` and from `authz.ResourceHierarchy` (`internal/authz/core/types.go:30-35`) — **but it is present on `authz.Context.Resource` (`types.go:52-55`), which `CheckRequest` already carries** (`services/authz.go:20-23`). It is ~4 lines to plumb, not new plumbing |
| 2.12 note: *"delete operations legitimately have no UID"* | **Confirmed and quantified** — 7 of 79 sites, all deletes/non-CRUD mutations: `git_secrets.go:98`, `dataplanes.go:200`, `exec.go:86`, `wirelogs.go:139`, `environments.go:200`, `secrets.go:179`, `projects.go:194`. v1 must document `uid` as optional and name the omitting classes |
| 2.7: add `actor.subject` (`sub`), `session_id`, `issuer` | **`actor.subject` already emits, as `actor.id`** — `ExtractActor` assigns it from the validated `sub` claim (`adapter.go:40-51`). Only `session_id` and `issuer` are new; adding `actor.subject` would publish the same value twice |
| P1.5b will reveal whether the core is service-neutral | **It did: yes.** `6739d1bd9` changed 55 files and 3661 lines, and `internal/server/middleware/audit/` by 3 lines of doc comment (§3) |
| EM-17's OCSF mapping is *"gated on §4.9.2"* and sized as net work | **Much cheaper than assumed.** OCSF's required `metadata` object already has a counterpart for every v1 envelope field except `producer_instance` (§10.2). The gate is a decision, not a research task |
| 2.1's note that the two render paths *could* diverge | **They already had** (found on implementing, 2026-09-07): the marshalled form ordered `origin`/`operation_id`/`resource` before `result` and emitted `"resource":null` where the log stream omitted the group. The single guarding test compared selected keys and caught neither |
| EM-5 treats the four tenancy fields as alike | **`environment` is not.** It publishes the dual-scoped `{namespace}/{name}` form, because it is stored as authorization evaluated it; the other three are bare names. The schema reference must say so, and §10.2's OCSF mapping should not assume a bare value |
| 2.10: `timestamp` → `event_time` is a rename | **It is a redefinition.** `buildEvent` stamps `time.Now()` (`emitter.go:83`) and the REST adapter emits from a `defer` after `next.ServeHTTP` (`middleware.go:183`), so the value is response-complete time, not the "when the action occurred" its own comment claims (`types.go:92`) — minutes late on a hijacked exec/wirelogs connection. `event_time` needs a **new capture at adapter entry**; `received_time` inherits the existing value. D8 |
| 2.7: `session_id`/`issuer` come from "the claims map already in request context" | **No accessor reaches them.** Claims are in context (`jwt/context.go:18`) but only via `GetClaims(r *http.Request)` while `ExtractActor` takes a `context.Context`; and `auth.SubjectContext` carries no claims at all (`auth/interface.go:12-17`), `resolver.go:92` reading `sub` and discarding the rest. Promote `iss`/`sid` onto `SubjectContext` at `resolver.go:93` instead |
| 2.7 assumes `sid` is available | **Unverified anywhere.** Nothing in the tree reads, configures or tests `sid`, and the bundled Thunder IdP is unconfirmed. `actor.session_id` needs a live-token check and, failing that, a documented omission class |
| 2.18: `sequence` is v1 because the render paths get touched anyway | **That argument died with item 0.** One render definition means adding it in P6 costs the same. Plus P4/P5a haven't started, so there is no transport to lose events in, and the pair can't detect tail loss without P6's heartbeat. **Deferred with `producer_instance`** — D9 |
| 2.9a: `actor.assurance` is a constant the trust model turns on | **It restates `actor.type`.** A `SubjectContext` exists only where `jwt.Middleware` already validated the signature, so `validated` ⟺ non-anonymous. Additive under D6, so **deferred to P9** with `on_behalf_of`, the field that gives `asserted` meaning |
| 2.11: `audit_id` is v1 because a correlation key can't group history already written | **True but hollow** — one event per operation today means history has nothing to group. Still v1, on a different basis: so P10a inherits the field across all history rather than a key beginning mid-stream. EM-13a's rejection of `request_id` stands on its own (client-suppliable, and already 1:many on MCP) |
| 2.1/2.10: `schema_version: 1` (§7) vs `"1.0"` (§10.1) | **Contradiction resolved to the string** — an integer cannot express an additive change, so every deferred field would bump either destructively or not at all. D6 |
| 2.3 does not say what `http.path` publishes | **Decided: raw request URI, query included, length-capped** on `RequestIDFromHeader`'s precedent, with item 14 classifying it as potentially-identifying. D7 |
