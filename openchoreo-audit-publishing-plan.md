# Audit Log Publishing — Implementation Plan

**Parent:** [`openchoreo-audit-delivery-plan.md`](./openchoreo-audit-delivery-plan.md) §1. This is the
per-repository breakdown of the **publishing** half only — producer → collector → destination.
**Not covered here:** the read path (parent §2) and its documentation, which is a separate plan.
**Phase:** P4-slice · P5a · **Status:** Ready to implement · **Date:** 2026-09-08

> **Delivery model, confirmed.** Publishing is **log-based**: the producer writes JSON to stdout and
> the collecting agent routes it. A push sink — an OTLP exporter from the emitter to an OTel
> Collector — is **deferred to a later phase**, where it lands as a second `Sink` implementation
> alongside event-based auditing. The `Sink` interface and multi-sink fan-out already exist
> (`emitter.go:17-19`, `:42`, `:65-67`), so nothing in this plan forecloses it; §OC-1 is in fact
> what makes the stdout sink a clean peer of a future OTLP one.

---

## 0. Scope, and the two findings that size it

**In scope:** one Go file in `openchoreo`; the fluent-bit routing, destination and retention in two
community modules; the publishing half of the operator docs.

| Finding | Consequence |
|---|---|
| **The `openchoreo` side of publishing is one file.** The audit config surface already exists on both producers, and #4636 already stamps `openchoreo.dev/plane` on every control-plane and observability-plane pod. All that is missing is the format guarantee | §OC is small and can land first, independently |
| **Publishing does not hard-block on [#4555](https://github.com/openchoreo/openchoreo/issues/4555).** The parent plan called that a single point of failure. Verified against the module README: the chart already supports a **fluent-bit-only install** on a cluster that hosts no backend, which is exactly what a control-plane cluster needs. In single-cluster (k3d) the DaemonSet already tails the producers today | #4555 improves the topology story; it does not gate audit. §CM assumes only the **existing** `kube.*` input |

**The dependency that remains** is a coordination one: if #4555 later splits the tail input or
changes its tag prefix, §CM-1's `Match` patterns must move with it. Tracked as R-1 in §5.

---

## 1. `openchoreo` — the producer

One task, one file, plus a documentation obligation inherited from P2.

### OC-1 — Give the audit logger its own JSON handler

**Problem.** `NewLogger` reuses the *application* logger's handler (`logger.go:55-57`), so audit
inherits its format. `cmd/observer/main.go:501-506` selects `slog.NewTextHandler` when
`LOG_LEVEL=debug`, under which audit records ship as text and §CM-1's collector rule matches
nothing — silently.

**Change.** `internal/server/middleware/audit/logger.go`:

```go
// NewLogger creates an audit logger writing newline-delimited JSON to w.
// It owns its handler rather than borrowing the application logger's: audit
// records are routed by a collector that parses them, so their format is a
// published contract and must not follow the app's logging.format or level.
func NewLogger(w io.Writer) *Logger {
    return &Logger{slogger: slog.New(slog.NewJSONHandler(w, &slog.HandlerOptions{
        Level: slog.LevelInfo,
    }))}
}
```

Call sites become `audit.NewLogger(os.Stdout)` — `cmd/openchoreo-api/main.go:213` and
`cmd/observer/main.go:591`.

**Three consequences, each a decision rather than a detail.**

1. **`forceLevelHandler` becomes dead code and should be deleted** (`logger.go:22-49`). It exists
   solely to stop the borrowed handler's level filter suppressing audit; owning the handler makes
   that structural. Leaving it in place leaves ~25 lines whose doc comments describe a problem the
   code no longer has. Delete `TestForceLevelHandler_WithAttrsAndWithGroupPreserveForce`
   (`logger_test.go:294`) with it.

   `TestLogEvent_NotGatedByAppLogLevel` (`logger_test.go:332`) becomes vacuous for the same reason
   — the app logger is no longer involved — so **replace it rather than delete it**, with a test
   that pins the new and stronger guarantee: *an audit record is JSON even when the application
   logger is a text handler at `LevelError`.* That is the regression this whole task exists to
   prevent, and nothing currently covers it.

2. **The `component` attr disappears from the record, and that is the right outcome.**
   `cmd/openchoreo-api/main.go:96` builds its logger with
   `logging.NewWithComponent(…, version.Get().Name)`, and `slog`'s `.With` bakes those attrs into
   the handler — so `slogger.Handler()` carries them and **today's openchoreo-api audit lines carry
   `"component":"openchoreo-api"`**. Observer's `initLogger` (`cmd/observer/main.go:59`) attaches
   none, so **its audit lines do not.** The field is therefore already inconsistent between the two
   producers, and it duplicates `producer`, which is inside the event and consistent by
   construction.

   Removing it is a change to the published log line but **not** a schema bump: `component`, like
   `msg` and `level`, is a slog envelope field and not part of `eventJSON` (`types.go:144-160`).
   Record it in the schema reference's envelope section anyway.

3. **The golden tests lose their `ReplaceAttr` hook.** `newRecordLogger`
   (`record_shape_test.go:24-33`) injects a `ReplaceAttr` that strips slog's `time` attr so the
   7 byte-exact `want` strings compare cleanly. With an owned handler the test can no longer
   inject it. **Do not add a production option for a test's benefit** — strip `time` from the
   emitted line in the test harness before comparing. The `logLinePrefix` constant
   (`{"level":"INFO","msg":"AUDIT-LOG",`) and all 7 `want` strings then stay unchanged, which is
   the point: this task must be provably shape-neutral for the event itself.

**Acceptance — and be honest about what a unit test can prove here.**

The guarantee this task delivers is **structural**: once `NewLogger` takes an `io.Writer`, there is
no application logger in scope to inherit a format from. A unit test that builds a text-mode app
logger and asserts the audit output is JSON would construct two unrelated objects and assert
nothing about the coupling — it would pass trivially and forever, including if handler borrowing
were reintroduced by another route. **Do not write that test.**

What actually guards the regression is a **call-site** assertion, because the call site is the only
place the two loggers still meet:

- `make test GO_RUN_ARGS="./internal/server/middleware/audit/..."` green with all 7 golden `want`
  strings in `record_shape_test.go` **unmodified** — the proof that this task is shape-neutral for
  the event.
- A wiring test that `observer`'s audit emitter emits parseable JSON when the process log format is
  text. `internal/observer/api/handlers/audit_wiring_test.go` is the existing precedent for
  asserting on audit wiring in that binary and is the natural home.
- Manual, and recorded as the verification of last resort rather than a substitute for the above:
  run `observer` with `LOG_LEVEL=debug`; application lines are text, audit lines are JSON.

**Files.** `internal/server/middleware/audit/logger.go`, `logger_test.go`,
`record_shape_test.go`, `cmd/openchoreo-api/main.go`, `cmd/observer/main.go`.

### OC-2 — Publish the routing contract

The collector matches on a field, so that field is a published interface and must be documented
where a consumer will look. Add to P2's generated schema reference
(`docs/audit/schema-reference.md`, item 12) an **Envelope and transport** section stating:

> An audit record is a single-line JSON object on the producer's stdout. The slog envelope carries
> `time`, `level: "INFO"` and `msg: "AUDIT-LOG"`; the event's own fields follow. **Collectors route
> on `msg`.** The marker is stable across schema minor versions. A record whose body cannot be
> rendered is published instead as `msg: "AUDIT-LOG-RENDER-FAILED"` carrying only the event's
> identity — it is a diagnostic, **not** an audit record, and must not be routed to the audit
> destination.

That last sentence is what §CM-1's rule has to honour, and stating it here is what makes the
requirement discoverable from the module side.

### OC-3 — Nothing in the Helm charts

Recorded so it is not re-investigated. The producer configuration
(`openchoreoApi.config.audit`, `observer.audit` — `enabled` / `defaults.publish` / `policies[]`)
already exists and is unchanged by this work, and pod identity labels landed in #4636
(`openchoreo.dev/plane: controlplane`, `_helpers.tpl:141-143`). **No `values.yaml` change, so no
`values.schema.json` regeneration and no `code.gen` impact on the openchoreo side.**

---

## 2. `community-modules` — collection, destination and retention

Two modules. OpenSearch is the reference implementation (ST-4) and should land first and complete;
OpenObserve follows and is what proves the destination naming is genuinely backend-neutral.

### 2.1 `observability-logs-opensearch`

#### CM-OS-1 — Source-gated audit routing in the fluent-bit config

**File.** `helm/templates/fluent-bit/config.yaml`.

Insert between the existing `[FILTER] Name kubernetes` and `[OUTPUT] Name opensearch` blocks — the
ordering is load-bearing, see the note below. One `rewrite_tag` per allowlisted producer, rendered
from values:

```
{{- range $p := .Values.auditLogs.producers }}
    [FILTER]
        Name          rewrite_tag
        Match         kube.var.log.containers.*_{{ $p.namespace }}_{{ $p.container }}-*
        Rule          $log "msg":"AUDIT-LOG"   audit.{{ $p.producer }}   false
        Emitter_Name  audit_emitter_{{ $p.producer }}
{{- end }}

    [FILTER]
        Name          parser
        Match         audit.*
        Key_Name      log
        Parser        audit_json
        Reserve_Data  On
        Preserve_Key  On
```

and a parser in the same ConfigMap's `parsers.conf`:

```
    [PARSER]
        Name   audit_json
        Format json
```

> **Deliberately no `Time_Key`/`Time_Keep`.** The obvious move is to drive the record timestamp
> from `event_time`, and it is a trap. The tail input already applies `multiline.parser docker, cri`,
> which sets the record time from the CRI line; re-setting it here means the OpenSearch output's
> `Logstash_Format On` derives the **index name** from a re-parsed value. Go marshals `time.Time`
> as RFC3339 with a `Z` suffix for UTC, which `%z` does **not** match — so a format mismatch would
> not error, it would silently stamp the wrong time and file records into the wrong daily index.
> Let the CRI-derived time stand as `@timestamp` and carry `event_time` as an ordinary indexed
> `date` field. The record then holds both, each honestly labelled: `@timestamp` is when the
> collector saw the line, `event_time` is when the audit adapter received the request.

Six things that must be right, in order of how expensive they are to get wrong:

1. **The `Match` is the security control; the `Rule` is only a discriminator.** The tag derives
   from the container log *filename*, which the kubelet writes — a pod cannot influence its own
   namespace or container name in it. Routing on payload content alone lets any workload forge an
   audit record. Full argument: parent plan §1.3.1.
2. **The `Rule` regex must be `"msg":"AUDIT-LOG"`, not `AUDIT-LOG`.** The bare token also matches
   `AUDIT-LOG-RENDER-FAILED` (OC-2), and `Keep false` would then *delete that diagnostic from the
   container-log stream* while filing it in the audit index. Confirm the exact quote escaping
   Fluent Bit's regex engine needs — this is the one line worth testing in isolation first.
3. **Order after the `kubernetes` filter.** That filter matches `kube.*` and so will not run on a
   re-emitted `audit.*` record; the enrichment must already be attached when the record is
   re-emitted. Those `kubernetes.*` fields are the provenance CM-OS-2 indexes and the read path
   cross-checks against `producer`.
4. **No emitter cycle.** `audit.openchoreo-api` does not match `kube.var.log.containers.*`, so the
   rules cannot re-trigger on their own output. Per the Fluent Bit docs a detected cycle is
   rejected with a logged error rather than looping; the distinct `Emitter_Name` per producer also
   keeps the two emitters separately visible in the collector's metrics, which is how an operator
   sees a rule that has stopped firing.
5. **`Merge_Log` stays `Off` on `kube.*`.** Turning it on globally would restructure every
   `container-logs-*` document and break the adapter's regex log-level extraction
   (`internal/opensearch/types.go:296-348`). Decoding happens only on the audit tag.
6. **No second `[OUTPUT]` guard is needed here** — the existing output's `Match kube.*` already
   excludes `audit.*`, so nothing double-ships. (This is *not* true on OpenObserve; see CM-OO-1.)

**`Preserve_Key On` is required, not a tuning choice.** It retains the raw pre-parse line, which is
the only artifact a later producer-side signature could verify over. It must be paired with the
`log` mapping in CM-OS-2 or `"dynamic": "false"` drops it and the storage cost buys nothing.

> **⚠️ The `Reserve_Data On` + `Preserve_Key On` combination is to-verify, not settled** (R-4).
> Three things must hold at once on the audit tag: parsed audit fields at top level, `kubernetes.*`
> enrichment retained, and the raw `log` string retained. `Reserve_Data` covers the second and
> `Preserve_Key` the third, but their interaction — and whether parsed keys land at top level or
> nested — is worth confirming empirically rather than from the flag names. §6 assertion 6 is the
> check; if it fails, this combination is the first place to look, not a mystery. Note also that
> when the parse succeeds and `log` survives, the payload is stored twice in the same document and
> CM-OS-2b's `wildcard` mapping indexes it. That is the accepted cost of the signing path.

#### CM-OS-2 — The audit output, index template and retention

**Files.** `helm/templates/fluent-bit/config.yaml` (output), `init/setup-opensearch.sh`,
`helm/templates/opensearch-setup-logs/job.yaml`, `helm/values.yaml`.

**a. Second `[OUTPUT]`**, mirroring the existing one and differing only in `Match` and prefix:

```
    [OUTPUT]
        Name opensearch
        Match audit.*
        Logstash_Format On
        Logstash_DateFormat %Y-%m-%d
        Logstash_Prefix {{ .Values.auditLogs.indexPrefix | trimSuffix "-" }}
        ... Host/Port/auth/tls identical to the container-logs output
```

**b. `auditLogsIndexTemplate`** in `setup-opensearch.sh`, next to `containerLogsIndexTemplate`
(`:60`) and added to the `indexTemplates` array (`:248`), which the script's own comment already
invites (*"Define more templates above and add them to this array"*). `"index_patterns":
["audit-logs-*"]`, `number_of_shards: 1`, `number_of_replicas: 1`, `"dynamic": "false"` — the
house convention.

**Derive the field list from the code, not from this plan.** `eventJSON`
(`internal/server/middleware/audit/types.go:144-160`) and `resourceJSON` (`:165-175`) are
authoritative. Three groups must be declared or `"dynamic": "false"` silently drops them:

| Group | Fields | Notes |
|---|---|---|
| Event envelope | `schema_version`, `event_id`, `event_time` (**`date`**), `action`, `category`, `result`, `request_id`, `source_ip`, `producer`, `origin`, `operation_id` — all `keyword` except `event_time` | `event_time` is the record's authoritative time; `@timestamp` is the collector's |
| Nested groups | `actor.{type,id,issuer,session_id}`, `http.{method,path}`, `resource.{type,namespace,environment,project,component,resource,uid,name}` | all `keyword` |
| Collector-added | `kubernetes.{namespace_name,pod_name,container_name}` as `keyword`; `log` as `wildcard` | provenance (CM-OS-1 note 3) and the preserved raw line |

Two mapping decisions to take here rather than defer:

- **`resource.environment` carries the dual-scoped `{namespace}/{name}` form** while its sibling
  tenancy fields are bare names — stored exactly as authorization evaluated it. `keyword` is right,
  but the schema reference must say why it differs or a consumer filters it wrongly once and works
  around it forever.
- **`actor.entitlements` is `map[string][]string`** (`types.go:18`) with no declared shape. Under
  `"dynamic": "false"` it is stored but **unindexed** unless mapped. This is the field the PII
  register singles out as having shipped with no plan. Decide explicitly: index it, or record that
  it is retrievable but not filterable.

**c. `auditLogsIsmPolicy`**, modelled on `containerLogsIsmPolicy` (`:361`) — the same
`active → delete` two-state machine, `ism_template.index_patterns: ["audit-logs-*"]`,
`priority: 100` — plus an entry in the `ismPolicies` array (`:436`) and a retention variable
alongside `:357-358`:

```bash
auditLogsRetention="${AUDIT_LOGS_MIN_INDEX_AGE:-365d}"
```

**d. Job env**, in `job.yaml` next to the two existing `*_MIN_INDEX_AGE` vars (`:41-44`):

```yaml
        - name: AUDIT_LOGS_MIN_INDEX_AGE
          value: "{{ .Values.openSearchSetup.dataRetentionTime.auditLogs }}"
```

**e. Values.** `dataRetentionTime.auditLogs: "365d"` beside the existing `containerLogs: "30d"` and
`k8sEvents: "30d"`. **Audit defaults far longer on purpose — a different retention policy is the
entire reason TR-7 asks for a separate stream.**

#### CM-OS-3 — The `auditLogs` values block

**File.** `helm/values.yaml`.

```yaml
auditLogs:
  # Route OpenChoreo audit records to their own index. Requires fluent-bit
  # to be running in the cluster where openchoreo-api / observer run.
  enabled: false
  indexPrefix: "audit-logs-"

  # The source allowlist. Each entry becomes one rewrite_tag rule matched on
  # the kubelet-written container log filename, so it cannot be satisfied by
  # anything a pod puts in its own log line. Adding an entry grants a workload
  # the right to write into the audit trail — review it as such.
  producers:
    - producer: openchoreo-api
      namespace: openchoreo-control-plane
      container: api-server
    - producer: observer
      namespace: openchoreo-observability-plane
      container: observer
```

**The defaults are verified, not guessed** — container `api-server` in namespace
`openchoreo-control-plane` (`openchoreo-api/deployment.yaml:60`) and container `observer` in
`openchoreo-observability-plane` (`observer-deployment.yaml:60`); both namespaces are Helm release
namespaces (`install/k3d/k3d-install.sh:24-27`) and overridable. Note the container is
`api-server`, **not** `openchoreo-api`.

Shipping both producers as defaults keeps a fresh install working with no manual configuration,
while an operator on non-default namespaces can see exactly what to change.

**Gating: routing is conditional, destination shaping is not.** Gate CM-OS-1 and CM-OS-2a on
`auditLogs.enabled`, but leave the index template and ISM policy **ungated**, so the destination is
correctly shaped before the first record could arrive — the same ordering trap the README's
existing troubleshooting entry describes for `container-logs-*`. **This is deliberate, not an
oversight:** the setup job will create an `audit-logs` template and ISM policy on every install,
including ones that never publish audit. Both are metadata against an index pattern that matches
nothing, so the cost is zero and the alternative — shaping the destination only once records are
already flowing — is the failure the troubleshooting entry exists for. Say so in the values comment
so a reviewer does not "fix" it.

#### CM-OS-4 — README

New `## Enable audit log collection` section, following the module's established spine, with four
subsections: **Trusted producers**, **Configuring the audit destination**, **Retention**,
**Credentials**.

*Trusted producers* must not read like ordinary configuration: say that each entry grants a
workload the right to write into the audit trail, that the list defaults to OpenChoreo's own two
producers, and that a non-default release namespace requires editing it **or audit collection
silently stops**. Show the parent plan's forgery example — an operator who has seen it will not
widen the list casually.

Add a Troubleshooting entry in the same class as the existing one: an `audit-logs-*` index created
before the template is applied misses the mappings and returns nothing.

### 2.2 `observability-logs-openobserve`

Same shape, three genuine differences. **This module is not at parity and the plan should not imply
it is** — it has no stream creation, no schema and no retention mechanism at all.

#### CM-OO-1 — Routing, and one mandatory fix to existing config

**File.** `helm/templates/fluent-bit/config.yaml`.

CM-OS-1's filters transfer unchanged. Then:

> **The existing output is `Match *` and must be narrowed to `Match kube.*`.** Without this the
> audit records also land in the `default` stream — which is readable by anyone with container-log
> access. That is a silent authorization hole, not a duplicate-storage annoyance, and it is the
> single highest-risk line in this module's work.

Second output:

```
    [OUTPUT]
        Name http
        Match audit.*
        URI /api/{{ .Values.common.openObserveOrg }}/{{ .Values.auditLogs.stream }}/_json
        ... Host/Port/auth/tls identical to the container-logs output
```

#### CM-OO-2 — Stream settings and retention (net-new for this module)

**File.** `init/setup-openobserve.sh` — today it creates only an alert template and a webhook
destination.

Add a phase, idempotent like the others, calling the stream settings API. **Verified against the
vendor docs** (`openobserve.ai/docs/reference/api/stream/setting`, fetched 2026-09-08):
`POST /api/{org_id}/streams/{stream_name}/settings` accepts **`data_retention` as an integer number
of days**, alongside `defined_schema_fields`, `index_fields`, `full_text_search_keys` and
`partition_keys`. `PUT` on the same path partially updates.

```bash
AUDIT_LOGS_RETENTION_DAYS="${AUDIT_LOGS_RETENTION_DAYS:-365}"
```

wired from a new `openObserveSetup.dataRetentionDays.auditLogs: 365` value and an env var on the
setup job, mirroring how the OpenSearch module does it.

**Recommend leaving the schema inferred** — do not set `defined_schema_fields` in phase 1. Pinning
the v1 field set here duplicates the OpenSearch index template with a second thing to keep in step,
and it is brittle across schema *minor* bumps, which are additive by policy. Record it as a choice.

> **The retention value shape differs and must not be unified.** OpenSearch ISM takes a duration
> string (`"365d"`); OpenObserve takes an integer count of days (`365`). Document per module rather
> than inventing a shared value one of them has to parse.

#### CM-OO-3 — HTTPRoute

**File.** `helm/templates/openobserve/httproute.yaml`. It has one hardcoded `PathPrefix` rule for
the single stream, so **a second stream is unreachable through the gateway in a multi-cluster
setup** — remote fluent-bit would fail to ship audit while succeeding on container logs. Add a rule
for `/api/{{ org }}/{{ auditLogs.stream }}/_json`.

#### CM-OO-4 — Values and README

`auditLogs` block as CM-OS-3 but with `stream: "audit_logs"` in place of `indexPrefix`, the same
`producers` allowlist, and `openObserveSetup.dataRetentionDays.auditLogs`. README section as
CM-OS-4, minus the index-template troubleshooting entry (which has no OpenObserve counterpart) and
noting that the stream is created on first ingest.

---

## 3. `openchoreo.github.io` — the publishing half of the docs

Write to `docs/`, which serves at `/docs/next/`. `sidebars.ts` is hand-authored and is the only
navigation mechanism. `onBrokenLinks` and `onBrokenAnchors` are `throw`.

### DOC-1 — `docs/platform-engineer-guide/audit-logging.mdx`, publishing sections

Create the page and write the publishing half now; the read-path sections land with that plan.
Follow `observability-alerting.mdx`'s house shape and register it under the existing
`Platform Engineer Guide → Observability` sidebar category, which today holds one page.

Sections in scope here: **Overview** · **Architecture** (Mermaid, single- and multi-cluster) ·
**Prerequisites** · **Enabling audit** (the producer config that already exists) · **Configuring
publishing** (the module `auditLogs` block, including the trusted-producer allowlist and why it is
a security boundary) · **Retention** · **Troubleshooting**.

Two things this page owns that no module README can, because a reader must be able to *search* for
them:

- **Why audit is on its own destination** — different retention, different access, TR-7.
- **The documented trust boundary.** The collector configuration is the trail's integrity anchor:
  whoever can write that ConfigMap can drop the audit output, redirect it, or rewrite `actor.id` in
  flight, and nothing downstream detects it. Record it beside the existing documented boundaries
  (`source_ip` and `X-Request-ID` are header-derived with no trusted-peer check). Kubernetes audit
  sets the precedent of documenting `sourceIPs` rather than pretending.

### DOC-2 — Note the module-README limitation

Module READMEs reach the site only through a **runtime `fetch()`** of `raw.githubusercontent.com`
in `src/components/EcosystemItem` — no build-time sync, no vendored copy. They are unversioned,
absent from DocSearch, and missing from the llms.txt and markdown-export outputs. So anything an
operator must find by searching belongs in DOC-1; the READMEs carry only what they read *after*
choosing a backend. No code change — this is a constraint on how DOC-1 and CM-OS-4/CM-OO-4 divide
content.

---

## 4. Sequencing

```
OC-1 ──► OC-2
  │
  └──► CM-OS-1 ─► CM-OS-2 ─► CM-OS-3 ─► CM-OS-4 ──┐
                                                   ├──► DOC-1 ─► DOC-2
       CM-OO-1 ─► CM-OO-2 ─► CM-OO-3 ─► CM-OO-4 ──┘
```

| Order | Task | Repo | Depends on | Why here |
|---|---|---|---|---|
| 1 | OC-1 | oc | — | Small, independent, and worth landing on its own merits. Nothing downstream is testable until audit is reliably JSON |
| 2 | OC-2 | oc | OC-1 | The contract the module implements against |
| 3 | CM-OS-1 → 3 | cm | OC-2 | Reference implementation. Land the destination shaping (CM-OS-2b/c) *before* the routing goes live |
| 4 | CM-OS-4 | cm | CM-OS-3 | |
| 5 | CM-OO-1 → 4 | cm | OC-2, CM-OS-* | Second implementation; do it after OpenSearch so the destination convention is settled, not negotiated twice |
| 6 | DOC-1, DOC-2 | site | all | Written when the values keys are real |

CM-OO can run in parallel with CM-OS-4 if two people are available; the only true ordering
constraint between the modules is that the **destination naming convention** is agreed once.

---

## 5. Risks and open decisions

| # | Item | Disposition |
|---|---|---|
| **R-1** | **#4555 may change the tail input's tag scheme.** CM-OS-1's `Match` patterns assume `Tag kube.*` over `Path /var/log/containers/*.log`, which is what the modules ship today. If #4555 splits the input and its platform-log input carries a different tag prefix, every `Match` matches nothing — **no records, no error** | Not blocking: audit ships on the existing input. Re-verify when #4555 lands, and rely on §6's *positive* check, since every negative test passes when nothing is collected |
| **R-2** | **The `Rule` regex escaping is unverified.** `"msg":"AUDIT-LOG"` contains quotes, and Fluent Bit's regex engine may need them escaped | Test the rule in isolation before wiring the rest. Cheap to check, expensive to miss |
| **R-3** | Removing the `component` attr changes the openchoreo-api log line | Deliberate (OC-1 note 2). Not a schema bump; record in the reference's envelope section |
| **R-4** | **`Reserve_Data On` + `Preserve_Key On` is an unverified combination.** The audit tag needs parsed fields at top level, `kubernetes.*` retained, and raw `log` retained, all at once | Confirm empirically before wiring the rest of CM-OS-1. §6 assertion 6 is the check; treat a failure there as this, not as a mapping problem |
| **R-5** | **Timestamp semantics.** `@timestamp` (CRI, collector-observed) and `event_time` (adapter entry) will differ, and the OpenSearch daily index is chosen by the former | Intended — see the parser note in CM-OS-1. Document both in the schema reference so the read path and any consumer know which to filter on. Do **not** "fix" it by re-parsing `event_time` into the record time |
| **D-1** | **`actor.entitlements`: index or store-only?** | Decide in CM-OS-2b. Recommend store-only for phase 1 and record it, since the PII register has not yet classified the field |
| **D-2** | **OpenObserve `defined_schema_fields`: pin or infer?** | Recommend **infer** (CM-OO-2). Pinning duplicates the OpenSearch template and is brittle across additive minor bumps |
| **D-3** | Delete `forceLevelHandler`, or keep it as defence in depth? | Recommend **delete** (OC-1 note 1). Kept, it is dead code whose comments describe a problem that no longer exists |
| **D-4** | Audit retention default of `365d` | A product decision, not a technical one. Flagged because it is the number an operator will most often override, and because it drives OpenSearch storage sizing (ST-8's guidance is a separate deliverable) |

---

## 6. Verification — the definition of done

**OC-1**, in `openchoreo`:

```sh
make test GO_RUN_ARGS="./internal/server/middleware/audit/..."
# → green, with all 7 golden `want` strings in record_shape_test.go UNMODIFIED
make code.gen && make lint-fix && make test && make code.gen-check
```

Plus the new regression test: application logger as a text handler at `LevelError`; the audit line
still parses as JSON and carries `msg: "AUDIT-LOG"`.

**CM + end to end**, on k3d (`make k3d`, then the observability plane and the logs module with
`auditLogs.enabled=true`):

```sh
curl -X POST .../api/v1/... -H "Authorization: Bearer $TOKEN"     # any audited operation
```

Six assertions, and **the order matters — the two negative ones are the tests worth writing first,
because a content-only rule passes every positive check:**

| # | Assertion | Catches |
|---|---|---|
| 1 | The record is in `audit-logs-YYYY-MM-DD` | The `Match` patterns resolve against the real tag (R-1) |
| 2 | Repeat for an `observer` operation (e.g. `UpdateIncident`) | A typo in one of the two rules, invisible in the other's result |
| 3 | It is **absent** from `container-logs-*` | `Keep false` |
| 4 | **Negative — over-broad marker.** A producer line containing `AUDIT-LOG-RENDER-FAILED` lands in `container-logs-*`, not `audit-logs-*` | The §CM-OS-1 note 2 bug |
| 5 | **Negative — forgery.** An ordinary workload pod in a data plane writes a well-formed audit record to stdout claiming `"producer":"openchoreo-api"`, `"result":"success"`. It appears **only** in `container-logs-*` | The whole point of the source gate |
| 6 | A stored record carries `kubernetes.container_name: api-server` beside `producer: openchoreo-api`, and `log` is present and searchable | Filter ordering (note 3), and that `Preserve_Key On` is paired with its mapping |

And retention:

```sh
curl -s localhost:9200/_plugins/_ism/policies/audit-logs | jq       # 365d, distinct from container-logs
curl -s "$OO_URL/api/default/streams/audit_logs/settings"           # data_retention: 365
```

**DOC.** `npm run build` in `openchoreo.github.io` (broken links and anchors fail the build) and
`npm run format`.
