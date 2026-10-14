# Audit Log Delivery — Publishing, Storage and Retrieval

**Companion to:** [`openchoreo-audit-logging-phased-plan.md`](./openchoreo-audit-logging-phased-plan.md) §P4/§P5a/§P7 (the phases this plans) and [`openchoreo-audit-logging-scope.md`](./openchoreo-audit-logging-scope.md) §2.5/§2.7/§2.8 (what each `TR-x`/`ST-x`/`AX-x` means).
**Predecessor:** [`openchoreo-audit-schema-evaluation.md`](./openchoreo-audit-schema-evaluation.md) — P2 froze the record this plan delivers.
**Phase:** P4-slice · P5a · P7 · **Status:** Design, not started · **Date:** 2026-09-08
**Repos:** `openchoreo`, `community-modules`, `openchoreo.github.io`

> **Why this document exists.** P2 froze the audit event. Nothing yet collects it, stores it or
> reads it back. This document connects the two remaining chains — **producer → collector →
> destination** and **destination → adapter → observer → caller** — and says what an operator
> configures at each tier, for each supported backend. Everything marked "verified" was read out of
> the tree or fetched from a vendor's own docs, with a `file:line` or a URL. Where a companion plan
> and the code disagree, the code wins and the disagreement is called out in §0.1.

> ### 🔑 The finding that reshapes all three phases
>
> **Audit is not a new pipeline. It is a fourth log class on a pipeline someone else is already
> building.**
>
> Epic [#4089](https://github.com/openchoreo/openchoreo/issues/4089) — *Observability (logs,
> metrics, events) for OpenChoreo system components*, milestone v1.3.0 — is delivering exactly the
> control-plane collection path P4 says does not exist. Two of its three layers are already real:
>
> | #4089 layer | State | What audit inherits |
> |---|---|---|
> | **1 — chart labels** | **Merged**, PR [#4636](https://github.com/openchoreo/openchoreo/pull/4636) | `openchoreo.dev/plane` and `openchoreo.dev/plane-id` on every pod template in all four plane charts, plus `spec.infrastructure.labels` for kgateway-rendered proxies |
> | **2 — collection & routing** (`community-modules`) | Not started | The platform-pod fluent-bit input, `Exclude_Path` on `*_dp-*_*.log` / `*_workflow-*_*.log`, and the required `clusterInstance` value |
> | **3 — observer API + adapter** | **In review**, PR [#4658](https://github.com/openchoreo/openchoreo/pull/4658) | `GET /api/v1alpha1/platform-logs`, `POST /api/v1alpha1/platform-logs/query` on the adapter contract, a `ClusterLogsQuerier` + authz decorator, and `clusterlogs:view` — the exact file-for-file template the audit read path clones |
>
> **Three consequences.** P4's *"control-plane telemetry foundation"* stops being audit's critical
> path and becomes a dependency owned by another epic. P5a shrinks to *one routing rule and one
> destination* on a collector that will already be tailing the right pods. And P7's read path
> stops being a design problem and becomes a clone of a PR that is in review right now.
>
> **The risk this introduces, stated plainly:** audit's Phase B cannot start until #4089 layer 2
> lands, and layer 2 has not started. That is the same single-point dependency the phased plan's §6
> flagged — *"there is no fallback path if the module work slips"* — now with a named owner.

---

## 0. The short answer

| Question | Answer |
|---|---|
| **How do audit logs reach a backend?** | The producer emits JSON on stdout with `msg: "AUDIT-LOG"`; #4089's fluent-bit `rewrite_tag`s that line onto its own tag — **gated on the kubelet-derived source tag, not the payload** — and a second `[OUTPUT]` ships it to a dedicated index/stream. §1 |
| **What stops a workload forging an audit record?** | The `Match` is an allowlist of producer `(namespace, container)` pairs taken from the container log filename, which a pod cannot influence. Content matching alone was the original design and is a forgery vector. §1.3.1 |
| **Where does an operator configure it?** | Four places, three of which already exist. The only genuinely new operator surface is the module's `auditLogs` block and one observer adapter URL. §1.5 |
| **How is it read back?** | `POST /api/v1alpha1/audit-logs/query` on the Observer → the same path on the adapter contract → the module. A clone of #4658's structure behind a dedicated `auditlog:view`, but a POST like the observer's eight other reads. §2 |
| **What is documented where?** | Official docs own everything backend-independent, including the ST-7 conformance matrix; module READMEs own destination names, retention values and credentials. §3 |

**Two things this plan deliberately does *not* do.** It does not extend #4658's platform-logs
endpoint to serve audit — §2.1 gives four reasons. And it does not put a telemetry target on the
`ObservabilityPlane` CRD — D-A.

### 0.1 Corrections to the phased plan, verified against the tree

Fold these back into `openchoreo-audit-logging-phased-plan.md` so the next reader isn't misled.

| Plan says | Verified reality |
|---|---|
| §2.6: *"The Observer's generated server is dead. Routes are hand-registered in `cmd/observer/main.go`"* | **No longer true.** P1.5a landed as `3fe559c9a`. Both surfaces are served through the generated strict server — `cmd/observer/main.go:343-352` (`gen`) and `:383-404` (`internalgen`). **P7 tasks 7.11 ("hand-register the routes") and 7.12 ("test asserting every spec path is registered") are written against a model that no longer exists** — a missing method now breaks the build |
| §2.6: `AdaptersConfig` has *"three hardcoded named pairs"* | **Four** — FinOps was added (`internal/observer/config/config.go:52-64`, defaults `:377-386`). Audit is the fifth |
| P5a deliverable 1: *"either a dedicated writer/stream, or a stable field the collector can match on"* | Both, and they are not alternatives. The **field** is the routing key; the **format guarantee** is what makes the field reliably present. §1.2 |
| P4 is *"the critical path… nothing after P5a can ship without it"* | Still true as a dependency, but **it is not audit's to build.** #4089 owns it, with layer 1 merged |
| §2.5: audit emission is *"hardwired"* with no sink abstraction | **A `Sink` interface and multi-sink fan-out already exist** — `emitter.go:17-19`, `:42`, `:65-67`. The hardwiring is one layer down: `NewLogger` inherits the *application* logger's handler (`logger.go:55-57`) |

---

## 1. Publishing — the operator experience

### 1.1 The chain, end to end

```
openchoreo-api / observer pod
  │  stdout: one JSON line per event, msg="AUDIT-LOG"              ← CONTROL PLANE
  ▼
fluent-bit DaemonSet  (platform-log input, from #4089 layer 2)
  │  [FILTER] rewrite_tag                                          ← AGENT
  │      Match  the kubelet-derived tag: <pod>_<plane-ns>_<container>
  │      Rule   $log ~ "msg":"AUDIT-LOG"   →  audit.<producer>
  │      Keep=false: the record LEAVES the platform-logs stream
  │  [FILTER] parser       decode the JSON into fields
  ▼
[OUTPUT] Match audit.*                                              ← BACKEND
  ├─ OpenSearch    →  index  audit-logs-YYYY-MM-DD   (ISM: 365d)
  └─ OpenObserve   →  stream audit_logs              (data_retention: 365)
```

**The `Match` is the security control and the `Rule` is only the discriminator.** See §1.3.1 —
routing on payload content alone is a forgery vector, raised in
[discussion #4396](https://github.com/openchoreo/openchoreo/discussions/4396#discussioncomment-17975277).

Three properties make this the right cut:

- **The workload/platform boundary is already drawn.** #4089 layer 2 must add an input that
  excludes `*_dp-*_*.log` and `*_workflow-*_*.log` at the *path*, so that anyone with `kubectl` who
  labels a workload pod cannot route it into a platform-scoped view. Audit inherits that boundary
  for free.
- **Audit cannot be split off by file path.** Audit records and application logs from
  `openchoreo-api` are interleaved in the *same* container log file, so the discriminator has to be
  record content.
- **`rewrite_tag` with `Keep false` removes the record from the platform-logs stream** as it
  re-emits it under `audit.*`. That is what stops audit being readable through a view gated only by
  `platformlogs:view`. Verified against the Fluent Bit docs: the rule form is
  `Rule $KEY REGEX NEW_TAG KEEP`, and a re-emitted record re-enters the pipeline from the start.

### 1.2 The control plane — the producer

**The operator surface already exists and does not change.** P0/P1 shipped it on both producers:

```yaml
# install/helm/openchoreo-control-plane/values.yaml:1877      (openchoreoApi.config.audit)
# install/helm/openchoreo-observability-plane/values.yaml:474 (observer.audit)
audit:
  enabled: true
  defaults:
    publish: true
  policies: []      # the ordered selector/settings list from P0/P1
```

**One code change is required, and it is the phase's only producer-side work.**

`NewLogger` currently wraps the *application* logger's handler (`logger.go:55-57`), so audit
inherits whatever format the app configured. `cmd/observer/main.go:501-506` selects
`slog.NewTextHandler` when `LOG_LEVEL=debug` — under which **audit records ship as text and the
collector rule in §1.3 silently matches nothing.** The fix is to construct
`slog.NewJSONHandler(os.Stdout, …)` inside `NewLogger`, keeping `forceLevelHandler` for the level
override that P0 added.

This is what P5a deliverable 1 means by *"its own format guarantee, not just its own destination"*.
It is strictly cheaper than the alternative it was weighed against — a dedicated file plus a
sidecar — which needs a volume and abandons the container-log model that every collector in the
ecosystem is built on.

> **The routing contract.** State it once, in the schema reference, and treat it as published:
>
> *An audit record is a single-line JSON object on the producer's stdout whose `msg` field is
> exactly `AUDIT-LOG`. Collectors route on that field. It is stable across schema minor versions.*

`msg` is a slog envelope field, not part of event schema v1, so freezing it costs no version bump
under D6's rule. Files: `internal/server/middleware/audit/logger.go` and the golden `want` strings
in `record_shape_test.go`, which pin exact bytes.

### 1.3 The agent — collection and routing

Lands in `community-modules`, in each logs module's `helm/templates/fluent-bit/config.yaml`, on top
of #4089 layer 2's platform-log input.

```
    # One rule per producer. Match is tag-based, and the tag comes from the
    # container log filename <pod>_<namespace>_<container> written by the
    # kubelet — it cannot be set from the payload. See §1.3.1.
    [FILTER]
        Name          rewrite_tag
        Match         kube.var.log.containers.*_openchoreo-control-plane_api-server-*
        Rule          $log "msg":"AUDIT-LOG"   audit.openchoreo-api   false
        Emitter_Name  audit_emitter_api

    [FILTER]
        Name          rewrite_tag
        Match         kube.var.log.containers.*_openchoreo-observability-plane_observer-*
        Rule          $log "msg":"AUDIT-LOG"   audit.observer   false
        Emitter_Name  audit_emitter_observer

    [FILTER]
        Name          parser
        Match         audit.*
        Key_Name      log
        Parser        audit_json
        Reserve_Data  On
        Preserve_Key  On
```

Namespaces and container names must be **templated from chart values, not hardcoded** as shown.
Verified in the tree: the producers are container `api-server` in namespace
`openchoreo-control-plane` (`openchoreo-api/deployment.yaml:60`) and container `observer` in
`openchoreo-observability-plane` (`observer-deployment.yaml:60`) — note the container is
`api-server`, **not** `openchoreo-api`, which is exactly the sort of value that gets guessed wrong.
Both namespaces are Helm release namespaces (`install/k3d/k3d-install.sh:24-27`) and are
overridable.

Three mechanics the implementer must get right, in this order of risk:

> **⚠️ The `Match` patterns above assume layer 2's tag scheme, and if that assumption is wrong,
> audit collection fails closed and silently.** They only resolve if the input carrying platform
> pods uses `Tag kube.*` over `Path /var/log/containers/*.log`. #4089 layer 2's `Exclude_Path`
> denylist implies it may **split the input**, and a separate input can carry its own tag prefix —
> in which case every `Match` here matches nothing, no records arrive, and **no error is raised
> anywhere**. This is the same silent-stop failure the `producers` values block warns operators
> about, one layer down. Treat layer 2's tag scheme as a hard dependency, pin it when layer 2
> lands, and rely on §6's *positive* Phase B assertion — that a real record actually arrives — to
> catch it, since every negative test passes when nothing is collected at all.

**The `kubernetes` filter must be ordered before `rewrite_tag`.** It matches `kube.*` and so will
not run on a re-emitted `audit.*` record; the enrichment has to already be attached when the record
is re-emitted. That enrichment is what §1.3.2's provenance cross-check rests on, so this is not a
tidiness constraint. Confirm it empirically — §6's Phase B check asserts a stored audit record
carries `kubernetes.container_name`.

**No emitter cycle, and Fluent Bit guards it anyway.** `audit.openchoreo-api` does not match
`kube.var.log.containers.*`, so the rules cannot re-trigger on their own output. Per the Fluent Bit
docs, a detected cycle is rejected with a logged error rather than looping, and the distinct
`Emitter_Name` per rule keeps the two emitters separately visible in metrics — which is how you see
a rule that has stopped firing.

**`Preserve_Key On` is required, not a tuning choice** — see D-F and §1.3.2. It retains the raw
pre-parse line, which is the only artifact a later producer-side signature could verify over; the
parsed-and-merged document is by construction not what the producer emitted. Two consequences to
accept openly: every audit record is stored twice, once raw and once parsed, and **the raw `log`
field must be declared in the index template** (§1.4) or `"dynamic": "false"` drops it and the cost
buys nothing.

> ### ⚠️ The rule's second field is a regex over the raw line, and the obvious pattern is wrong
>
> `AUDIT-LOG` on its own is unanchored and matches a substring anywhere in the line — including
> **`AUDIT-LOG-RENDER-FAILED`**, the diagnostic `logRenderFailure` emits at `logger.go:96-104`.
> With `Keep false` that diagnostic would be *removed from the platform-logs stream* and filed in
> the audit index instead. A render failure silently vanishing from the operator's log view is the
> worst failure mode available here.
>
> Match the JSON key/value pair, not the token. **Two things the implementer must confirm against
> Fluent Bit rather than assume:** the exact quote escaping its regex engine needs, and whether
> `$log` is still the right record accessor once #4089 layer 2's input lands — a different parser
> on that input may name the field differently.
>
> §6's Phase B check includes the negative case, because the positive one passes either way.

#### 1.3.1 Why the `Match` is source-gated — the forgery vector

> **Raised by @Pinguladora in [discussion #4396](https://github.com/openchoreo/openchoreo/discussions/4396#discussioncomment-17975277)
> (2026-08-11), and it changes the design rather than adding to it.**

Fluent Bit tails `/var/log/containers/*.log` with no namespace or label filter. **If the collector
routes on payload content alone, any workload that can write to stdout can mint an audit record.**
An application pod in a data plane writing this line would land a forged event in the audit index,
and the read path would have no reliable way to tell it from a real one:

```json
{"schema_version":"1.0","producer":"openchoreo-api","origin":"api",
 "category":"authorization","action":"update_role_mapping",
 "actor":{"type":"user","id":"someone@acme.corp"},"result":"success"}
```

Every field an audit consumer trusts — `producer`, `actor.id`, `action`, `result` — is
attacker-chosen in that line. **`msg: "AUDIT-LOG"` is not a credential.**

**The fix, and why it holds.** The `Match` pattern keys on the **tag**, and with `Tag kube.*` on the
tail input the tag is derived from the container log *filename*:
`kube.var.log.containers.<pod>_<namespace>_<container>-<id>.log`. That filename is written by the
**kubelet**, not by the container, so a pod cannot influence its own namespace or container name in
it. A line from an application pod therefore never reaches the `rewrite_tag` rule at all: it stays
in the application log stream as an ordinary log line.

So routing becomes **two-factor: an unforgeable source gate, plus a content discriminator within
that source.** The content marker's only remaining job is separating audit records from the
producer's *own* application logs — which is a correctness problem, not a security one.

**This composes with #4089 layer 2 rather than duplicating it.** That epic excludes `*_dp-*_*.log`
and `*_workflow-*_*.log` at the input for the same reason — because pod *labels* are writable by
anyone with `kubectl` and log *paths* are not. Audit applies the same principle one layer further
in, as an allowlist of two producers rather than a denylist of workload prefixes. **Prefer the
allowlist here:** a new plane or a renamed workload silently widens a denylist, and cannot widen
this one.

> **Two corrections to the discussion's example recipe**, neither affecting its argument:
>
> - It matches `$audit_id ^.+$`. **`audit_id` is not in schema v1** — it was deferred to P10a, to
>   land with the pre-action event that gives it a second record to group. And with `Merge_Log Off`
>   the payload is still an opaque string at that point, so `$audit_id` would not resolve at all;
>   the raw line is in `$log`. The marker stays `$log` against `"msg":"AUDIT-LOG"`.
> - It uses one rule. **There are two producers** in two namespaces, so there are two rules — and
>   that is an improvement, because it makes the destination tag (`audit.openchoreo-api` /
>   `audit.observer`) itself pipeline-derived.

#### 1.3.2 Should the collector overwrite `producer`? — recommend no, assert instead

The discussion proposes a `[FILTER] Name modify / Set producer openchoreo-api` on the audit tag, so
the field comes from the pipeline rather than from the payload. The instinct is right; **the
mutation is the wrong way to act on it**, for two reasons:

- **It breaks record fidelity, and the fidelity is load-bearing later.** The same comment argues for
  producer-side signing in a later phase. A collector that rewrites a field inside the signed
  payload guarantees every signature fails verification. Any field the collector mutates is a field
  signing can never cover.
- **The tag gate already closes the hole.** `producer` is stamped in `buildEvent` from the emitter's
  own `serviceName` (`emitter.go`), constructed once at boot. Once the `Match` guarantees the record
  came from the `api-server` container, `producer: openchoreo-api` is not attacker-chosen — so
  overwriting it corrects nothing an attacker could have caused.

**Recommend instead: keep the record verbatim and index the pipeline's own provenance beside it.**
`Reserve_Data On` already retains the `kubernetes` filter's enrichment — `namespace_name`,
`pod_name`, `container_name` — all kubelet-derived and unforgeable. Map those in the audit index
template (§1.4) and treat them as authoritative. The read path then has a **cross-check it can act
on**: a record whose `producer` disagrees with its `kubernetes.container_name` is evidence of a
misconfigured collector, and can be surfaced rather than silently normalised away.

If a deployment prefers the simpler option, `modify Set` remains available — but it should be a
recorded choice, because it forecloses signing.

#### 1.3.3 What this does not protect, and must be documented

The same comment makes a second point worth carrying verbatim into the docs rather than resolving:

> **The collector configuration becomes the trust anchor.** Whoever can write that ConfigMap can
> drop the audit output, redirect it, or add a filter rewriting `actor.id` in flight, and nothing
> downstream would detect it. That places the integrity of the audit trail on RBAC over a
> ConfigMap rather than on the records themselves — and that ConfigMap edit could itself be a
> `kubectl` or GitOps change, i.e. inside the accepted-risk gap the epic already records for
> GitOps.

This is a privileged / post-compromise scenario, so it is not a reason to change phase 1. It **is**
the strongest argument for producer-side signing in a later phase, and it belongs in the schema
reference's limitations section next to the existing documented trust boundaries (`source_ip` and
`X-Request-ID` are taken from headers with no trusted-peer check). Kubernetes audit sets the
precedent: it *documents* that `sourceIPs` can be client-set rather than pretending otherwise.

Two follow-ups this opens, both out of scope here and worth filing:

- **Tamper evidence via producer signing** (SPIFFE/SVID), over the whole event rather than a chosen
  subset — the comment's own refinement, since signing `event_id`/`sequence`/`producer_instance`
  alone would leave `actor.id`, `action`, `resource` and `actor.assurance` rewritable.
- **A threat-model document** recording the accepted risks: GitOps/`kubectl` edits outside the API,
  and collector-config RBAC.

Four more things this has to get right, all module-side:

1. **A second `[OUTPUT]` with `Match audit.*`.** On OpenSearch the existing output's `Match kube.*`
   already excludes the new tag, so nothing double-ships. **On OpenObserve the existing output is
   `Match *` and must be narrowed to `kube.*`**, or audit lands in the `default` stream as well —
   a silent authorization hole, not merely a duplicate.
2. **`Reserve_Data On`** keeps the `kubernetes.*` enrichment — pod, namespace, and #4636's
   `openchoreo.dev/plane` labels — alongside the decoded audit fields.
3. **A dedicated tail DB** if layer 2 introduces a second `[INPUT]`. Both inputs would otherwise
   share `DB /var/lib/fluent-bit/db/tail-container-logs.db` and collide on cursors.
4. **`Merge_Log` stays `Off` on `kube.*`.** Turning it on globally would restructure every
   `container-logs-*` document and break the adapter's regex log-level extraction
   (`internal/opensearch/types.go:296-348`). Decoding happens only on the audit tag.

New module values, one block per module:

```yaml
auditLogs:
  enabled: false                 # publishing is opt-in per backend
  indexPrefix: "audit-logs-"     # opensearch
  stream: "audit_logs"           # openobserve

  # The source allowlist. Each entry becomes one rewrite_tag rule; the tag is
  # kubelet-derived, so this is the security boundary, not a convenience.
  # Adding an entry here is what grants a workload the right to write audit
  # records — review it as such.
  producers:
    - producer: openchoreo-api
      namespace: openchoreo-control-plane
      container: api-server
    - producer: observer
      namespace: openchoreo-observability-plane
      container: observer
```

Shipping the two known producers as the default keeps a fresh install working with no manual
configuration, while leaving the list explicit enough that an operator running a non-default
namespace can see what to change — and that adding a third producer is a deliberate act.

### 1.4 The backend — destination and retention

**The destination name is the coupling contract** — the only thing shared between the collection
side and the query side (the phased plan's §2.9 calls this the transplantable pattern from the
events split). Name it once, and word it backend-neutrally so a SQL or table-shaped backend later
has somewhere to sit:

| Backend | Destination | Set by |
|---|---|---|
| OpenSearch | index `audit-logs-YYYY-MM-DD`, daily — matching `container-logs-*` and `k8s-events-*` | `Logstash_Prefix audit-logs` |
| OpenObserve | stream `audit_logs` in the configured org | output `URI /api/{org}/audit_logs/_json` |

#### OpenSearch — three additions to an explicitly extensible script

`init/setup-opensearch.sh` already says *"Define more templates above and add them to this array"*,
and drives both templates and policies from parallel bash arrays.

- **`auditLogsIndexTemplate`** — `"index_patterns": ["audit-logs-*"]`, `"dynamic": "false"` (the
  house convention), plus an entry in the `indexTemplates` array. **Derive the field list from the
  code, not from this document:** `eventJSON` (`types.go:144-160`) and `resourceJSON`
  (`types.go:165-175`) are the authoritative published shape — `schema_version`, `event_id`,
  `event_time`, `actor`, `action`, `category`, `result`, `request_id`, `source_ip`, `producer`,
  `origin`, `operation_id`, `http`, `resource`, `metadata`, with `resource` carrying
  `type`/`namespace`/`environment`/`project`/`component`/`resource`/`uid`/`name`.

  Two mapping decisions to take **during** this work rather than now:

  - **`resource.environment` publishes the dual-scoped `{namespace}/{name}` form** while its
    sibling tenancy fields are bare names — it is stored exactly as authorization evaluated it.
    `keyword` is the right type, but the schema reference must say why it differs, or a consumer
    filters it wrongly once and works around it forever.
  - **`actor.entitlements` is `map[string][]string`** (`types.go:18`) with no declared shape. Under
    `"dynamic": "false"` it is stored but **unindexed** unless mapped deliberately. This is the
    field the PII register singles out as having shipped with no plan, so decide it explicitly:
    index it, or record that it is retrievable but not filterable.

  **Two more groups the template must declare, because `"dynamic": "false"` silently drops
  anything it does not:**

  - **The collector's provenance fields**, which §1.3.2 makes load-bearing rather than incidental:
    `kubernetes.namespace_name`, `kubernetes.pod_name` and `kubernetes.container_name` as
    `keyword`. These are kubelet-derived and unforgeable, and they are what lets the read path
    cross-check a record's claimed `producer` against where it actually came from.
  - **The preserved raw line**, `log`, as `wildcard` — the same type the existing `container-logs`
    template gives it. Without this declaration `Preserve_Key On` costs collector throughput and
    index storage and delivers nothing, which would quietly remove D-F's foundation.

  Note the OpenSearch fluent-bit output sets `Replace_Dots On`, so label keys arrive with `.`
  rewritten to `_` — the existing `container-logs` template already mirrors that convention and the
  audit one must too.

- **`auditLogsIsmPolicy`** on `audit-logs-*`, plus an entry in `ismPolicies` — the same
  `active → delete` two-state machine the other two classes use.
- **`AUDIT_LOGS_MIN_INDEX_AGE`** in `helm/templates/opensearch-setup-logs/job.yaml`, fed from
  `openSearchSetup.dataRetentionTime.auditLogs`.

**Retention defaults longer than application logs — that is the entire point of TR-7.** Recommend
`auditLogs: "365d"` against the existing `containerLogs: "30d"` and `k8sEvents: "30d"`.

#### OpenObserve is not at parity, and this plan says so rather than implying it

That module has **no stream creation, no schema and no retention mechanism at all**. Streams appear
on first ingest, and the adapter already swallows error `20002` (*stream not found*) as zero hits.
So this is net-new work there, not a mirrored change.

Verified against the vendor docs (`openobserve.ai/docs/reference/api/stream/setting`, fetched
2026-09-08): `POST /api/{org_id}/streams/{stream_name}/settings` accepts **`data_retention` as an
integer number of days**, alongside `defined_schema_fields`, `index_fields`,
`full_text_search_keys` and `partition_keys`. Two additions:

- that call in `init/setup-openobserve.sh`, which today creates only an alert template and a
  webhook destination;
- a second `PathPrefix` rule in `templates/openobserve/httproute.yaml` — today one hardcoded rule
  reaches only the single stream, so a second stream is unreachable through the gateway in
  multi-cluster mode.

**The shape mismatch is unavoidable and belongs in the docs:** OpenSearch ISM takes a duration
string (`"365d"`), OpenObserve takes days as an integer (`365`). Do not invent a shared value that
one of them then has to parse.

### 1.5 The complete operator surface

| Tier | Key | Where | Status |
|---|---|---|---|
| Producer — control plane | `openchoreoApi.config.audit.{enabled,defaults,policies}` | `openchoreo-control-plane/values.yaml:1877` | **exists** |
| Producer — observability plane | `observer.audit.{enabled,defaults,policies}` | `openchoreo-observability-plane/values.yaml:474` | **exists** |
| Pod identity | `openchoreo.dev/plane`, `openchoreo.dev/plane-id` | all four plane charts | **merged — #4636** |
| Agent — collection | platform-log input, `clusterInstance` | module `values.yaml` | **#4089 layer 2** |
| Agent — audit routing | `auditLogs.enabled`, `auditLogs.indexPrefix` / `.stream` | module `values.yaml` | new |
| Backend — retention | `openSearchSetup.dataRetentionTime.auditLogs` · OpenObserve `data_retention` | module `values.yaml` | new |
| Read | `observer.auditAdapter.{url,timeout}` | `openchoreo-observability-plane/values.yaml` | new |
| Authorization | `auditlog:view` on bootstrap roles | `openchoreo-control-plane/values.yaml` | new |

**Two of eight rows are genuinely new operator configuration.** That is the measure of how much
riding #4089's pipeline buys.

### 1.6 Decisions taken, each with the constraint that decides it

**D-A — Telemetry targeting is chart-level configuration, not a CRD field.**
This closes the phased plan's open decision gate *"control-plane telemetry targeting"*. Nothing in
the tree reads `ObservabilityPlane` to resolve any telemetry or adapter target — its three URLs
(`observerURL`, `rcaAgentURL`, `finOpsAgentURL`, `api/v1alpha1/observabilityplane_types.go:14-45`)
are all *read* endpoints consumed by the CLI and portal, and the Observer's own adapter URLs are
flat koanf/env (`config.go:52-64`). Adding a CRD reader is net-new machinery for a value the
collector consumes at Helm-install time regardless. **#4089 makes the same call**, and matching it
matters more than the abstract consistency argument.

**D-B — Audit is control-plane-scoped, so there is exactly one audit destination per installation.**
Not one per environment, not one per data plane. The producers are `openchoreo-api` and `observer`,
each a singleton in its plane. This is the answer to *"the configured observability plane for audit
logs"*: there is one, and it is a chart value.

**D-C — Same observability plane as workload logs, a separate destination within it.** The audit
adapter URL is nonetheless **its own koanf key**, defaulting to the same `http://logs-adapter:9098`
Service. So an operator who later wants audit in a hardened, separately-credentialed backend
changes configuration, not code. The read-side half of pluggability falls out for free.

**D-D — Route on an unforgeable source gate *and* a content marker, never content alone.** The
`Match` is an allowlist of producer `(namespace, container)` pairs, derived by the kubelet from the
container log filename; the `Rule` only separates audit records from that producer's own
application logs. **Content-only routing was the original design and it is a forgery vector** —
§1.3.1. Rejected alternatives for the marker itself: a dedicated file plus sidecar (needs a volume,
abandons the container-log model); stderr as the audit channel (shared with Go panics and any
third-party library that writes there).

**D-F — The collector records provenance; it does not rewrite the event.** `producer` stays as the
emitter stamped it, and the kubelet-derived `kubernetes.*` fields are indexed beside it as the
authority. Overwriting `producer` in-flight would foreclose the producer-side signing that §1.3.3
argues for. §1.3.2.

**D-F depends on `Preserve_Key On` and its index mapping, so the two travel together.** The parser
filter already means the stored document is not byte-identical to the emitted line; the retained
raw `log` field is the only thing a signature could ever verify over. Drop either the flag or its
`wildcard` mapping and D-F's argument no longer holds — at which point the simpler `modify Set
producer` becomes the better choice. Reviewers should treat all three as one decision.

**D-E — Publishing is opt-in per module (`auditLogs.enabled: false`); production is always on.**
OP-6: a control plane with no observability plane configured runs unchanged, and events still reach
stdout where `kubectl logs` finds them.

**D-G — Delivery stays log-based in this phase; a push sink is deferred.** Pushing from the emitter
to a collector — an OTLP exporter to an OTel Collector, which would then own backend and index
routing — was considered and **deferred to a later phase**, to land as a second `Sink`
implementation alongside event-based auditing. It removes the forgery vector at the root rather
than gating it (§1.3.1 becomes moot with no shared stdout to forge into), but it pulls forward two
things this phase does not want: P5b's transport work, and part of P6's reliability work, since
`Emit` runs synchronously in the request goroutine (`middleware.go:183`) and TR-5/PR-9 forbid
blocking the request path on a network hop. **Nothing here forecloses it** — the `Sink` interface
and multi-sink fan-out already exist (`emitter.go:17-19`, `:42`, `:65-67`), and §1.2's owned handler
makes the stdout sink a clean peer of a future OTLP one rather than a special case.

The implementation breakdown for everything in this section is
[`openchoreo-audit-publishing-plan.md`](./openchoreo-audit-publishing-plan.md).

---

## 2. Retrieval and presentation through the Observer

> ### 🔧 Revised 2026-09-09 — #4658 merged, and five of this section's claims move
>
> The implementation breakdown is now
> [`openchoreo-audit-observer-api-plan.md`](./openchoreo-audit-observer-api-plan.md), which supersedes
> this section wherever they differ. What changed, all verified in the tree:
>
> | This section says | Now |
> |---|---|
> | Naming unsettled between `platform-logs` and `cluster-logs` | **Settled: `platform-logs`**, action `platformlogs:view` (`actions.go:267`). §2.1's warning is closed, and open question 1 with it |
> | §2.3: add *"a fifth adapter pair"*, `audit.adapter.url` | **D-C is revised.** `GetPlatformLogs` is a **method on the existing `*LogsAdapter`** reusing `logs.adapter.url`; audit follows. A separate key is inert config today and additive whenever someone needs the split |
> | §2.3: *"Generalise `mapAdapterHTTPError`"* | **Moot.** Platform logs checks `501` inline and returns a per-signal sentinel, bypassing that function. The hard-code survives in `alerts_adapter.go:118` as an unrelated cleanup |
> | §2.2: `501` appears *"exactly once"* in the adapter spec | **Twice** — the precedent for declaring it on a new operation is established, not argued |
> | §2.4: grant to *"admin, platform-engineer and security-admin"* | **`security-admin` does not exist** as a bootstrap role. Phase 1 grants admin + platform-engineer and records AX-4 as partially met |
>
> **And the verb flipped: the audit read is `POST /api/v1alpha1/audit-logs/query`, not a GET.**
> Open question 2 is closed. Eight of the observer's nine read endpoints are already `POST …/query`;
> `GET /platform-logs` is the outlier. Decisively, `auditgen` skips every `GET` **before** its
> `Overrides` check (`generator.go:67`), so a GET audit-read could never satisfy AX-5 without a
> generator change — whereas a POST is a coverage-gate candidate by default, which turns leaving it
> unaudited into a visible line in `exemptions.go`. The verb table in §2.1 and the "follow the
> newest precedent" paragraph below are superseded on this point.

### 2.1 Clone the platform-logs pattern; do not extend the platform-logs endpoint

**Audit must not be served by #4658's endpoint.** Four reasons, all pointing the same way:

| | Platform logs | Audit |
|---|---|---|
| Permission | `platformlogs:view` / `clusterlogs:view` — reaches SRE-shaped roles | AX-4: admin, platform engineer, security admin **only** |
| Retention | operational, days | compliance, months to years |
| Destination | platform-logs index | `audit-logs-*`, separately access-controlled |
| Shape | free-text lines with a `level` | a frozen structured schema with no `level` at all |

What audit *does* reuse is #4658's **shape, file for file**. That PR is the template:

| #4658 (platform logs) | Audit equivalent |
|---|---|
| `openapi/observer-api.yaml` → `GET /api/v1alpha1/platform-logs` | `POST /api/v1alpha1/audit-logs/query` — **verb differs deliberately**, see the revision note |
| `openapi/observability-logs-adapter-api.yaml` → `POST /api/v1alpha1/platform-logs/query` | `POST /api/v1alpha1/audit-logs/query` **with `501` declared** |
| `pkg/observability/logs.go` → `PlatformLogsAdapter` | a new `AuditAdapter` |
| `internal/observer/service/platform_logs{,_adapter,_authz}.go` | `audit_logs{,_adapter,_authz}.go` |
| `internal/observer/api/handlers/platformlogs.go` | `auditlogs.go` |
| `internal/observer/types/platformlogs.go` | `types/auditlogs.go` |
| `internal/authz/core/actions.go` → `clusterlogs:view` | `auditlog:view` |
| `.mockery.yaml` → `ClusterLogsQuerier` | `AuditLogsQuerier` |

> **Naming is unsettled upstream, and audit must not add a third vocabulary.** #4658 uses
> `platform-logs` in its paths and handler files, `clusterlogs:view` for the action, and "cluster
> logs" in its title. Whichever wins, audit mirrors it. Resolve before the audit spec is written.

> ~~**Follow the newest precedent on verb and shape:** `GET` with flat query parameters, as #4658
> chose deliberately.~~ **Superseded 2026-09-09 — the read is a `POST …/query`.** #4658 is one
> endpoint against the observer's eight existing read-as-POST endpoints, and the GET would have put
> AX-5 out of reach. Reasoning in
> [`openchoreo-audit-observer-api-plan.md`](./openchoreo-audit-observer-api-plan.md) §C-2.

What audit does inherit from that precedent is the discipline, not the verb: every array filter
needs a declared cap, enforced server-side with a `400` rather than truncated.

### 2.2 Contract work — `openchoreo`, and it must land first

1. **`POST /api/v1alpha1/audit-logs/query` on the adapter contract, declaring `501` from day one.**
   **This is the single highest-consequence detail in the phase.** `501` appears exactly once in
   that spec today (`openapi/observability-logs-adapter-api.yaml:181`, on `/events/query`), which is
   why a module *cannot legally 501 out of* `/logs/query` and the ones that try resort to a 500
   carrying a message. Omit it here and every existing `observability-logs-*` module becomes
   contractually obliged to implement audit on its next, unrelated regeneration.
2. **`AuditSearchScope`.** Neither existing scope fits: `ComponentSearchScope` requires `namespace`
   and is UID-based, `WorkflowSearchScope` is workflow-shaped, and an audit event is frequently
   neither — a role change is org-scoped and an `access` rejection resolves to no resource at all.
3. **Filters** for time range, actor, action, category, resource, result, producer and tenancy
   (AX-2). **Declared attributes (AT-9) and collapse-on-`audit_id` (AX-2a) are out of this phase** —
   P3 has not declared attributes, and `audit_id` was deferred to P10a. Leave room; do not build.
4. **Cursor pagination — opaque and module-encoded.** `limit` plus a `total` capped at 1000 does not
   satisfy AX-2's stable ordering, and neither module has pagination today (`from` is hardcoded `0`
   on OpenObserve; OpenSearch has no `from`, `search_after` or scroll). Specify the cursor as an
   opaque token the module mints. **Specifying a cursor *shape* would pick a backend by accident.**

   > **The OpenSearch half is more than `search_after`, and the module task should say so.** The
   > reference adapter resolves indices per request with `GenerateIndices(start, end)` — a
   > day-by-day enumeration with `IgnoreUnavailable: true`. `search_after` alone gives stable
   > ordering over a *fixed* index set; here a new daily index can appear mid-scroll and an ISM
   > delete can remove the oldest one. **A point-in-time opened against the resolved index set is
   > what makes AX-2's "stable ordering" true**, so the task is *"open a PIT, encode its id in the
   > cursor, handle expiry as a distinct error"* — not *"add `search_after`"*. The contract must
   > therefore define an error for an **expired or retention-aged-out cursor**: a `4xx` telling the
   > caller to restart the query, never a silent empty page.
5. **Add the codegen line to `make/golang.mk`'s `openapi-codegen` target** — a new spec is not
   picked up automatically.

### 2.3 Observer

- **A fifth adapter pair** in `AdaptersConfig` (`config.go:52-64`): `audit.adapter.url` and
  `audit.adapter.timeout`, an `AUDIT_ADAPTER_URL` env mapping (`config.go:214-221`), defaults
  alongside `config.go:377-386`, and the `observer-config.yaml` template.

  > *Worth fixing in passing:* `METRICS_ADAPTER_URL` is **absent** from
  > `templates/observer/observer-config.yaml` even though the config requires it — it silently
  > falls back to the `http://metrics-adapter:9099` default. Pre-existing, and the audit key
  > touches the same file.
- **Service plus authz decorator wired once in `main.go`**, which is what gives HTTP *and* MCP
  coverage from one change — the pattern `NewLogsServiceWithAuthz` and its siblings established.
- **Generalise `mapAdapterHTTPError`**, which hard-codes `501 → ErrEventsNotImplemented`.
- **AX-7 degradation.** No audit adapter configured, or a `501` from the module, must produce a
  clear *"no audit backend configured"* error — never an empty success presented as complete. This
  is also how scope §4.2's ST-6-vs-AX-1 contradiction resolves: **forward-only is an explicitly
  read-path-disabled configuration**, recorded rather than discovered.

### 2.4 Authorization

`auditlog:view`, cluster-scoped, registered exactly the way #4658 registers `clusterlogs:view`.
Five edit sites — and **an action absent from `systemActions` expands to nothing and grants
nothing, silently.** That trap is already sprung in the tree by `portal-assistant:invoke` at
`openchoreo-control-plane/values.yaml:1409`, which is dead config no CI check catches.

`internal/authz/core/actions.go` (const **and** the `systemActions` table) ·
`internal/observer/authz/constants.go` · a `*_authz.go` service decorator · the Helm bootstrap
roles · the `.mockery.yaml` entry.

**Grant to `admin` (automatic via `*`), `platform-engineer` and `security-admin`. Deliberately not
`sre` or `rca-agent`**, despite their existing broad observability reads. AX-4 makes audit read a
sensitive privilege in its own right — audit records disclose activity across every team — and
this is a call to state, not to default into.

**AX-5 — audit the audit read.** There is **no precedent for auditing a read** anywhere in
`definitions.go`, so this is a new pattern rather than one more entry. The read event is itself
published to the audit index; that is intended, and P0's policy vocabulary already allows
suppressing it if the volume becomes a problem.

### 2.5 The module side

**`observability-logs-opensearch` is the reference implementation** (ST-4 — it is the default
deployment): a query handler translating the contract into Query DSL over `audit-logs-*` through
the existing `GenerateIndices` day-walk, plus §1.4's index template and ISM policy.

**`observability-logs-openobserve` follows as the second implementation**, and it is what actually
proves ST-3's pluggability claim — one contract over a DSL engine and a SQL engine, with
`_timestamp` in microseconds against `@timestamp` in RFC3339 and a second `count(*)` round-trip for
`total`.

**Every other `observability-logs-*` module gets a trivial `501` PR opened proactively**, rather
than a landmine waiting on its next unrelated regeneration.

---

## 3. Documentation

The site is `openchoreo.github.io`. Write to `docs/`, which serves at `/docs/next/` and becomes
`/docs/` at the next version cut (`lastVersion = 'v1.2.x'`). `sidebars.ts` is hand-authored and is
the **only** navigation mechanism — `_category_.json` is vestigial, one file, driving nothing.
`onBrokenLinks` and `onBrokenAnchors` are both `throw`, so a bad anchor fails the build.

### 3.1 Official docs — everything backend-independent

| Page | Type | Content |
|---|---|---|
| `docs/platform-engineer-guide/audit-logging.mdx` | **Hub — the main deliverable** | Follow `observability-alerting.mdx`'s house shape: Overview → Architecture (Mermaid, single- and multi-cluster) → Prerequisites → Enabling audit → Configuring publishing → Retention → Viewing the trail → Configuration Reference (Key Helm Values) → Troubleshooting → Related Documentation. Add under the existing `Platform Engineer Guide → Observability` sidebar category, which today holds exactly one page |
| `docs/reference/audit-event-schema.mdx` | **Reference, generated** | The v1 schema field by field, CloudTrail's major/minor bump rule verbatim, and the documented omission classes: `http.*` on MCP tool-call operations, `resource.uid` on deletes and non-CRUD mutations, `resource.environment`'s dual-scoped form, `actor.session_id` where the IdP issues no `sid`, the unbounded-path/`MaxHeaderBytes` note, and the plain statement that **v1 offers no completeness guarantee** |
| `docs/reference/audit-pii-register.mdx` | Reference | SE-6. Must cover `actor.entitlements`, which P1 shipped as an emitted field with no plan |
| Conformance matrix (ST-7) | Reference table | Which modules implement audit query and which return `501`. **This is the seam** — §3.3 |
| `docs/platform-engineer-guide/modules/observability-logging-adapter-api.mdx` | Reference | **No work.** It already `SwaggerUI`-embeds the adapter spec from `main`, so the new operation appears the moment §2.2 merges |
| `docs/platform-engineer-guide/modules/building-a-module.md` | How-to | One section: what a module must implement for audit, and that `501` is the sanctioned answer |
| `docs/concepts/platform-abstractions.md:73` | Concept | Already claims *"all platform activity is auditable"* with nothing behind it. Link it to the hub |

**The generated schema reference — pick the precedent deliberately.** `make audit.schema-docs`
produces `docs/audit/schema-reference.md` in this repo (P2 item 12), joining the two generated
coverage matrices already under `docs/audit/`. Getting it onto the site means a sibling of
`openchoreo.github.io/scripts/generate-helm-docs.js`, which already reads
`../openchoreo/install/helm` from a sibling checkout and writes MDX with injected frontmatter.
**Two caveats to record rather than discover:** that script is run **by hand — no CI workflow
invokes it** — and its output carries no "do not edit" banner. Both apply to the new one, and both
are worth fixing while adding a second consumer.

### 3.2 Module docs — everything backend-specific

Each module's `README.md` gets one new section, following the established spine (`## Prerequisites`
→ `## Installation` → `## Enable log collection` → `## Dependencies` → `## Compatibility`):

```
## Enable audit log collection
### Trusted producers                      the (namespace, container) allowlist
### Configuring the audit destination      index name  |  stream name
### Retention                              "365d" ISM  |  data_retention: 365
### Credentials
```

**"Trusted producers" is the section that must not read like ordinary configuration.** It has to
say, in the operator's own terms, that each entry grants a workload the right to write records into
the audit trail, that the list defaults to OpenChoreo's own two producers, and that a non-default
release namespace requires editing it or audit collection silently stops. §1.3.1's forgery example
is the right thing to show — an operator who has seen it will not widen the list casually.

OpenSearch additionally gets a Troubleshooting entry in the same class as its existing one: an
`audit-logs-*` index created *before* the template is applied misses the `"dynamic": "false"`
mappings and returns nothing.

> **One limitation to state, not paper over.** Module READMEs reach the site only through a
> **runtime `fetch()`** of `raw.githubusercontent.com` in `src/components/EcosystemItem` — there is
> no build-time sync and no vendored copy. They are unversioned, absent from DocSearch, and missing
> from the llms.txt and markdown-export outputs. So **anything an operator must find by searching
> belongs in official docs**; the module README carries only what they read *after* choosing a
> backend.

### 3.3 The seam between the two

The **ST-7 conformance matrix in official docs, linking out to each module README**, is the join.
ST-7 exists precisely so an operator can tell, *before* committing to a backend, whether it
satisfies the read path.

Because **no module descriptor declares a signal type or an adapter API version** — `module.yaml`
is an images manifest, `Chart.yaml` keywords are prose, and the ecosystem catalog's `category`/
`tags` are free text — the matrix is **hand-maintained**. So this plan names who updates it: the PR
that adds audit support to a module updates the matrix row in the same change, and
`building-a-module.md` says so.

---

## 4. Sequencing

```
#4089 layer 2  (platform-log collection, cm)  ──┐
                                                ├──►  B. collection + destination (cm)
A. producer format guarantee (oc, small)      ──┘             │
                                                              ▼
#4658 merges  ──►  C. contract: adapter + observer specs (oc)
                              │
                              ├──►  D. observer read path + authz (oc)
                              └──►  E. module query handlers (cm)
                                          │
                                          ▼
                                  F. documentation
```

| Phase | Repo | Blocked by | Notes |
|---|---|---|---|
| **A** — audit logger's own JSON handler; publish the routing contract | oc | — | Independent and small. Do it first; it unblocks any collector work and is worth landing even if B slips |
| **B** — `rewrite_tag` + parser, second `[OUTPUT]`, index template, ISM policy, OpenObserve stream settings + HTTPRoute | cm | #4089 layer 2 | **The single point of failure.** In the log-based delivery the module owns collection *and* query, and there is no fallback path |
| **C** — `/audit-logs/query` on the adapter spec **with `501`**, `GET /audit-logs` on observer-api, `AuditSearchScope`, opaque cursor | oc | #4658 naming | Must merge before any module work starts, so modules implement against a fixed contract |
| **D** — observer service, adapter client, authz decorator, `auditlog:view`, `mapAdapterHTTPError`, AX-5 | oc | C | A clone of #4658's files |
| **E** — OpenSearch handler, then OpenObserve; proactive `501` PRs on the rest | cm | C, B | OpenSearch is the reference (ST-4) |
| **F** — docs | site, cm | A–E | Conformance matrix last, when the answers are real |

**Out of scope, recorded so the omissions read as decisions:** the Backstage portal viewer (P8)
consumes this API and is planned separately; webhook and multi-sink publishing (P5b); durable
queueing, sequence numbers and the degraded condition (P6); declared attributes (P3) and
collapse-on-`audit_id` (P10a) — the read path leaves room for both and implements neither.

**Two follow-ups this plan opens rather than closes**, both from §1.3.3 and both worth filing now so
they are not rediscovered: **producer-side signing** (SPIFFE/SVID over the whole event), and a
**threat-model document** recording the accepted risks — GitOps and `kubectl` edits that bypass the
API, and RBAC over the collector ConfigMap being the trail's integrity anchor.

---

## 5. Open questions needing a decision before implementation

1. ~~**`platform-logs` vs `cluster-logs`.**~~ **Closed 2026-09-09: `platform-logs`**, action
   `platformlogs:view`.
2. ~~**Is `GET` with query parameters right for audit specifically?**~~ **Closed 2026-09-09: no —
   `POST /api/v1alpha1/audit-logs/query`.** The fallback this question named became the answer.
3. **OpenObserve stream schema.** `defined_schema_fields` would pin the v1 field set the way the
   OpenSearch index template does; leaving it inferred is less work and less brittle across schema
   minor bumps. **Recommend inferred for phase 1**, recorded here so it is a choice.
4. **Does `sre` get `auditlog:view`?** **Recommend no** — reversible, and better stated than
   defaulted.
5. **Do audit-read events go to the audit index?** **Recommend yes** (AX-5), with the policy
   vocabulary available to suppress them if the volume becomes a problem.
6. **Stamp `producer` from the pipeline, or keep the record verbatim and assert against
   `kubernetes.*`?** **Recommend assert** (§1.3.2) — mutation forecloses the producer-side signing
   that §1.3.3 argues for. Worth confirming with @Pinguladora, since it is a deliberate departure
   from the recipe in the discussion and rests on a constraint that comment itself introduced.
7. **What should the read path do with a `producer` / `kubernetes.container_name` mismatch?**
   Surfacing it is the point of indexing both. Options: reject the record at query time, return it
   flagged, or expose it only through a health/consistency check. **Recommend flag and return** —
   dropping evidence of a misconfigured collector is the wrong instinct for an audit trail. Not
   blocking for phase 1, but decide before the read path treats the fields as decorative.

---

## 6. Verification

**Phase A** — `make test` on the audit package; the golden `want` strings in `record_shape_test.go`
change only in the slog envelope. Then run `observer` with `LOG_LEVEL=debug` and confirm the audit
line is still JSON while application logs are text. That single check is the whole point of the
phase.

**Phase B** — on k3d (`make k3d`, then the observability plane and the logs module):

```sh
curl -X POST .../api/v1/... -H "Authorization: Bearer $TOKEN"    # any audited operation
#  → present in audit-logs-YYYY-MM-DD                                ← the Match patterns resolve
#  → absent from container-logs-* AND from the platform-logs index   ← the Keep=false check
curl -s localhost:9200/_plugins/_ism/policies/audit-logs | jq    # retention differs from container logs
```

**The positive check is doing double duty and should be read as two assertions.** "A record
arrived" is the *only* evidence that the `Match` patterns resolve against #4089 layer 2's actual
tag scheme — the §1.3 dependency whose failure mode is silence. Run it against both producers,
`openchoreo-api` and `observer`, since they are two separate rules and a typo in one is invisible
in the other's result.

> **Two negative cases, and only they catch the bugs that matter. The positive check above passes
> with either defect present.**
>
> 1. **Over-broad marker.** Emit a line containing `AUDIT-LOG-RENDER-FAILED` from a producer pod and
>    assert it lands in the **platform-logs** index, not the audit one (§1.3).
> 2. **Forgery.** From an ordinary workload pod in a data plane, write a well-formed audit record to
>    stdout claiming `"producer":"openchoreo-api"` and `"result":"success"` — the §1.3.1 payload.
>    Assert it appears **only** in `container-logs-*` and **never** in `audit-logs-*`. This is the
>    check that proves the source gate is doing its job, and it is the one to write first, because
>    a content-only rule passes every other test in this section.

Two more assertions on a real stored record, each pinning a mechanic that would otherwise fail
quietly:

- **Provenance survived the re-emit.** The document carries `kubernetes.container_name: api-server`
  alongside `producer: openchoreo-api`, and both are queryable. This is what proves the
  `kubernetes` filter is ordered before `rewrite_tag` (§1.3) and gives §1.3.2's cross-check
  something to work with.
- **The raw line survived and is indexed.** `log` is present and searchable, not merely stored —
  the pairing of `Preserve_Key On` with its `wildcard` mapping that D-F rests on.

**Phases C–E** —

```sh
curl -s -H "Authorization: Bearer $ADMIN_TOKEN" \
  -X POST http://localhost:11080/api/v1alpha1/audit-logs/query \
  -d '{"startTime":"…","endTime":"…","category":["authorization"]}' | jq
curl -s -H "Authorization: Bearer $DEV_TOKEN" '…/audit-logs'     # → 403, not a filtered empty list
# stop the adapter                            → a clear error, not an empty success
# point observer at a module returning 501    → "no audit backend configured"   (AX-7)
# page past the cursor's expiry               → 4xx telling the caller to restart
# the read above appears in the audit index   → AX-5
```

Then, before any PR: `make code.gen && make lint-fix && make test && make code.gen-check`. Note
`helm-generate` is part of `code.gen`, so hand-editing `values.yaml` without regenerating
`values.schema.json` fails the check.

**Phase F** — `npm run build` in `openchoreo.github.io` (broken links and anchors are build
failures) and `npm run format`.
