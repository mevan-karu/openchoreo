# Branch verification test plan — `feat/audit-logger-json-handler`

Manual verification of the current branch against a live k3d OpenChoreo cluster.

## Environment under test

| Item | Value |
|---|---|
| Cluster | k3d `k3d-openchoreo` (single cluster, all four planes) |
| Control plane API | http://api.openchoreo.localhost:8080 |
| UI | http://openchoreo.localhost:8080 |
| Data plane gateway | http://localhost:19080 |
| IdP | Thunder (`http://thunder.openchoreo.localhost:8080`) |
| Branch HEAD | `b92d180f0` |
| Server build reported by `occ version` | `1.2.0` / rev `b92d180f` |
| Images | `ghcr.io/openchoreo/*:latest-dev` |

Two identities were used, so both actor types are covered:

| Identity | Grant | Actor type | Entitlement claim |
|---|---|---|---|
| `service_mcp_client` | client_credentials | `service_account` | `sub` → `service_mcp_client` |
| `admin@openchoreo.dev` | authorization_code + PKCE (`openchoreo-cli` client) | `user` | `groups` → `admins` |
| `customer-portal-client` | client_credentials | `service_account` | used as the unprivileged subject for `denied` |

The user login runs headlessly against Thunder's own flow API — `GET /oauth2/authorize`
→ `POST /flow/execute` with the credentials → `POST /oauth2/auth/callback` with
`{assertion, authId}` → authorization code → token. That is the same sequence the
`/gate/signin` page performs, so it exercises the real PKCE path without a browser
(script: `pkce_login.py` in the run's scratchpad).

## What is being verified

The branch is upstream `main` plus the audit-logging work. The tests below cover
(a) that core platform flows still work on this build, and (b) the audit behaviour
the branch introduces:

- `#4571` audit every state-modifying openchoreo-api operation
- `#4643` audit state-modifying observer-api operations
- `#4664` record `resource.environment` in audit events
- `#4675` freeze the audit event schema as v1
- `#4686` give the audit logger its own JSON handler
- audit **policies** (`audit.defaults` / `audit.policies`, first-match-wins) and the
  startup selector-admissibility gate
- `#4660` Delivery Insights configuration + authorization surface
- `#4685` sre role `resourcereleasebinding:update`

## Status legend

`PENDING` · `PASS` · `FAIL` · `BLOCKED` · `N/A`

---

## Part A — Baseline

| # | Test | Expected | Status | Notes |
|---|---|---|---|---|
| A1 | Server build identity | `occ version` server rev == branch HEAD | **PASS** | server `1.2.0` / `b92d180f` == HEAD `b92d180f0` |
| A2 | All plane pods healthy | every pod Running/Ready | **FAIL** | 45/47 healthy. `opentelemetry-collector` and `prometheus-operator` in CrashLoopBackOff — see finding 1 |
| A3 | Admin token via Thunder client_credentials | 200 + JWT | **PASS** | `service_mcp_client`, `client_secret_basic` |
| A4 | Authenticated `GET /api/v1/namespaces` | 200 | **PASS** | |
| A5 | Unauthenticated `GET /api/v1/namespaces` | 401 | **PASS** | |
| A6 | Live audit config in ConfigMap | `enabled: true`, `defaults.publish: true`, `policies: []` | **PASS** | matches chart defaults |

## Part B — Core platform flows

| # | Test | Expected | Status | Notes |
|---|---|---|---|---|
| B1 | Create project via REST | 201, `Project` CR created | **PASS** | `audit-verify` |
| B2 | Create component via REST | 201, `Component` CR created | **PASS** | `audit-verify-comp` |
| B3 | List/read back through REST | created objects visible | **PASS** | |
| B4 | Deploy `samples/from-image/go-greeter-service` | component + binding reach Ready | **PASS** | ReleaseBinding Ready, DP pod 1/1 in ~80s |
| B5 | Invoke the deployed endpoint through the data plane gateway | HTTP 200 greeting | **PASS** | `Hello, AuditTest!` via `:19080` |
| B6 | Delete component + project via REST | 200/204, CRs gone | **PASS** | 16 deletes → 16× `204`, all CRs gone, 16 audit records (1:1) |

## Part C — Audit emission with default policy

Assertion surface: `AUDIT-LOG` records on stdout of `deploy/openchoreo-api`
(the only sink wired today — `cmd/openchoreo-api/main.go:213`).

| # | Test | Expected | Status | Notes |
|---|---|---|---|---|
| C1 | Every state-modifying REST call emits exactly one audit record | 1:1 | **PASS** | measured on three independent batches: the C-series batch (4 calls → 4 records), B1 project create (1 → 1) and the B6 cleanup (16 deletes → 16). Part B's `kubectl apply` and gateway invoke emit nothing, correctly — they never reach the API |
| C2 | Every record matches the frozen v1 schema | schema v1 | **PASS** | 21 captured records (openchoreo-api + observer) checked against `eventJSON`/`resourceJSON` in `internal/server/middleware/audit/types.go`: `schema_version: "1.0"` on all; zero keys outside the schema (top level, `resource`, `actor`, `http`); zero field-order violations, top level or nested; all 10 non-optional keys present on every record; only `metadata` never observed, which is `omitempty` |
| C3 | Read-only GETs emit no audit record | 0 records | **PASS** | 3 GETs → 0 records |
| C4 | `resource.environment` populated on an environment-scoped operation | field present | **PASS** | `UpdateReleaseBinding` → `"environment": "default/development"` on both success and failure |
| C5 | Failed operation (404 on update) is audited with `result: failure` | record present | **PASS** | `delete_project` on missing project → `result: failure` |
| C6 | Denied operation (unprivileged token) audited with `result: denied` | record present | **PASS** | `customer-portal-client` → 403 + `result: denied` |
| C7 | Unauthenticated state-modifying request audited with `result: unauthenticated` | record present | **PASS** | actor `anonymous/anonymous`, no `action`/`operation_id` (by design — runs before operation resolution) |
| C8 | MCP-origin operation audited with `origin: mcp` | record present | **PASS** | `create_project` MCP tool → `origin: mcp`, no `http` block |
| C9 | Audit records survive application log level `error` (own handler, #4686) | records still emitted | **PASS** | with `logging.level: error`, 3/3 audit records still emitted at INFO while application INFO logs dropped to 0 |
| C10 | Audit records are well-formed newline-delimited JSON, one object per line | parses | **PASS** | every record parsed by `json.loads` per line |

## Part D — Audit policies

Applied by patching `cm/openchoreo-api-config` and restarting the deployment.
The original ConfigMap is saved before the first edit and restored after Part D.

Suppression is asserted as *absence from the API pod's stdout*. That is a valid
proof of non-emission only because `Logger` is the single `Sink` wired today
(`cmd/openchoreo-api/main.go:213`, `cmd/observer/main.go:591`) and `Emitter.Emit`
returns before touching any sink when `Publish` resolves false
(`internal/server/middleware/audit/emitter.go`). If a second sink is ever added
— the `Sink` interface anticipates a webhook or queue — every suppression row
below stops proving what it claims and needs a second assertion at that sink.

### D-positive

| # | Policy | Expected | Status | Notes |
|---|---|---|---|---|
| D1 | `defaults.publish: false` + rule `match: {resources: [project]} set: {publish: true}` | project op audited, component op not | **PASS** | 3 ops → 2 records (both project ops, success + failure); component suppressed |
| D2 | Two matching rules with opposite `publish`, first says `false` | first match wins → suppressed | **PASS** | 3 ops → 1 record (component only); both project rules matched, first won |
| D3 | `match: {results: [success]} set: {publish: false}` | successes suppressed, every other result still audited | **PASS** | 5 ops → 3 records: both successes suppressed; `failure` (404), `denied` (403) and `unauthenticated` (401) all passed through |
| D4 | `defaults.publish: false` + rule `match: {entitlements: [service_mcp_client]} set: {publish: true}` | audited | **PASS** | 3 records for the matching subject; a denied op from `customer-portal-client` correctly stayed suppressed |
| D5 | `match: {origins: [mcp]} set: {publish: false}` | MCP suppressed, REST audited | **PASS** | 3 REST records, 0 for the MCP `create_project` call |

### D-user (selectors that need a real user actor)

Run with a PKCE token for `admin@openchoreo.dev`. Each case fires the *same*
operation twice — once as the user, once as the service account — so the
policy's effect is visible as which of the two got audited, not as a bare count.

| # | Policy | Expected | Status | Notes |
|---|---|---|---|---|
| U1 | none (baseline) | user op audited with `actor.type: user` | **PASS** | `actor.id` = the Thunder `sub` (`01a08565-83dc-…`), `entitlements: {groups: [admins]}` — resolved from the `groups` claim per the `user` subject config, where the service account resolves from `sub` |
| U2 | `match: {actor_types: [user]} set: {publish: false}` | user suppressed, service account audited | **PASS** | 2 ops → 1 record (the service account's) |
| U3 | `defaults.publish: false` + `match: {actors: [<user sub>]} set: {publish: true}` | user audited, service account not | **PASS** | 2 ops → 1 record (the user's) |
| U4 | `defaults.publish: false` + `match: {entitlements: [admins]} set: {publish: true}` | user audited on the group claim | **PASS** | 2 ops → 1 record (the user's); confirms `entitlements` matches on claim *values* regardless of which claim carries them |

### D-negative (startup validation gate)

Each of these must make the API pod fail startup with a specific message.

| # | Invalid policy | Expected message | Status | Notes |
|---|---|---|---|---|
| D6 | `match: {actors: [x]} set: {publish: false}` | "may not be used to suppress publishing" | **PASS** | field `audit.policies[0].set.publish` |
| D7 | `match: {}` + `set: {publish: false}` | "would silence every event" | **PASS** | field `audit.policies[0].match` |
| D8 | `set: {category: management}` | "stamped from the operation, not operator-configurable" | **PASS** | field `audit.policies[0].set.category` |
| D9 | `match: {resources: [nosuchresource]}` | vocabulary `MustBeOneOf` failure | **PASS** | field `audit.policies[0].match.resources[0]`, message enumerates the valid resource vocabulary |
| D10 | `set: {}` (sets nothing) | "policy sets nothing" | **PASS** | field `audit.policies[0].set` |
| D11 | `set: {bogus: true}` | `unknown setting "bogus"` | **PASS** | field `audit.policies[0].set.bogus` |
| D12 | Config restored, pod healthy again | back to baseline | **PASS** | live ConfigMap byte-identical to the pre-test backup; deployment rolled out clean |

In every negative case the container exited 1 at startup with a single
field-scoped `Invalid configuration` error, and the rolling update never
terminated the healthy old pod — the API stayed available on a bad policy
rather than taking the control plane down.

## Part E — Observer audit (#4643)

| # | Test | Expected | Status | Notes |
|---|---|---|---|---|
| E1 | Observer audit config present in its ConfigMap | `audit.enabled: true` | **PASS** | `observer-auth-config`, same shape as the API's (`defaults.publish: true`, `policies: []`) |
| E2 | Observer read-only query emits no audit record | 0 records | **PASS** | `POST /incidents/query` → 0 records |
| E3 | `UpdateIncident` (observer's only state-modifying op) audited | record with `producer: observer` | **PASS** | `update_incident`, `producer: observer`, `result: failure` on a missing incident id |

## Part F — Other branch changes touched in passing

| # | Test | Expected | Status | Notes |
|---|---|---|---|---|
| F1 | Delivery Insights config surface present in control-plane config/authz (#4660) | present | **PASS** | `DELIVERY_INSIGHTS_AGGREGATION_ENABLED=false`, `DELIVERY_INSIGHTS_EVENTS_SOURCE_ENABLED=false` in `observer-config` (defaults leave it off); `deliveryinsights:view` in the live `sre` ClusterAuthzRole |
| F2 | `sre` role has `resourcereleasebinding:update` (#4685) | present in ClusterAuthzRole | **PASS** | present in the live cluster-scoped role |

---

## Findings

### 1. `opentelemetry-collector` and `prometheus-operator` are in a permanent CrashLoopBackOff

Not caused by this branch — both pods come from separate observability charts
(`observability-events-otel-collector-0.1.1` and
`observability-metrics-prometheus-0.7.0`), not from `install/helm` in this
checkout. Recording it because it is the one thing broken in the cluster.

Both die on the liveness probe rather than on their own error:

| Pod | Limits | Liveness probe | Start → SIGTERM | Start → exit | Exit |
|---|---|---|---|---|---|
| `opentelemetry-collector` | 100m CPU / 200Mi | `:13133/`, delay **0s**, timeout 1s, period 10s, failures 3 | ~29s | ~29s | 2 |
| `prometheus-operator` | 40m CPU / 60Mi | `/healthz`, delay **0s**, timeout 1s, period 10s, failures 3 | ~34s | ~60s | 137 (SIGKILL after the termination grace period) |

`opentelemetry-collector` writes nothing to stdout across 33 restarts, and
`prometheus-operator`'s own log shows a normal startup right up to
`received SIGTERM, exiting gracefully` — so neither is failing on config. Each
survives roughly its probe budget (3 × 10s plus scheduling slack), then gets
killed from outside.

The node is at **135% CPU limit** and **101% memory limit** overcommit on 5 CPU
/ 8 GB, so both containers are starved enough to miss a 1-second probe with no
initial delay. Neither probe has an `initialDelaySeconds` or a `startupProbe`,
so a slow start is indistinguishable from a dead process and the pod can never
recover.

Impact: no trace or metric ingestion through the collector, and no
`Prometheus`/`Alertmanager` CR reconciliation. The already-created Prometheus
and Alertmanager pods are running, so existing metric queries still work, and
OpenSearch/fluent-bit logging is unaffected.

Suggested fix, in preference order: add a `startupProbe` (or
`initialDelaySeconds: 30`) to both in the upstream charts; and/or raise the CPU
limits in the k3d values — 40m for `prometheus-operator` is below what it needs
to answer a probe during informer sync.

### 2. No bugs found in the audit work on this branch

All 10 emission cases, all 5 policy behaviours and all 6 startup-validation
cases behaved exactly as the code documents. Specifically worth noting:

- Policy validation errors are field-scoped and quote the offending path
  (`audit.policies[0].match.resources[0]`), and the vocabulary error enumerates
  the valid resource names — an operator can fix a typo from the message alone.
- A bad policy is contained: the container exits 1 at startup, the rolling
  update never terminates the healthy pod, and the API keeps serving. Testing
  six invalid configs in a row never once took the control plane offline.
- `audit.defaults.publish: false` allowlist mode works and is correctly exempt
  from the blanket-suppress guard that rejects an equivalent *rule*.
- The published record is stable in both content and *order*: across 21 records
  from two producers, no key outside the frozen schema appeared and no field
  was ever emitted out of `eventJSON`'s declared order.
- Actor resolution is per-subject-type and correct in both directions: a user
  resolves `actor.id` from `sub` and entitlements from `groups`, a service
  account resolves entitlements from `sub`, and all three actor-keyed selectors
  (`actor_types`, `actors`, `entitlements`) discriminate between the two.
- `logging.level: error` silenced every application INFO log while all audit
  records still came through at INFO — the separate handler (#4686) does what
  it claims.

### 3. Minor: commit message for #4660 names the action `insights:view`

The implementation and chart both use `deliveryinsights:view`
(`internal/authz/core/actions.go:292`,
`internal/observer/authz/constants.go:16`, control-plane `values.yaml`). Only
the commit message text is out of date; nothing to fix in code.

## Coverage gaps

Stated rather than papered over:

- **`#4616` (record the commit a workload was built from)** was not covered —
  it needs a build-from-source run through the workflow plane, not a
  from-image deploy.
- **`#4614` (delivery lifecycle events from the releasebinding controller)**
  was not verified end-to-end; the events sink depends on the collector, which
  is down (finding 1).
- **Observer audit coverage is one operation wide** because `UpdateIncident` is
  the only state-modifying operation the observer defines. It was exercised on
  the failure path only (no live incident existed to resolve).

## Run log

| Step | Action |
|---|---|
| 1 | Confirmed running images are built from this branch (`occ version` server rev `b92d180f`) |
| 2 | Acquired an admin token from Thunder via client_credentials (`service_mcp_client`) |
| 3 | Backed up `cm/openchoreo-api-config` before any policy edit |
| 4 | Part B: project/component CRUD through REST; deployed `samples/from-image/go-greeter-service`; invoked it through the data plane gateway |
| 5 | Part C: 10 audit-emission cases, asserted against `AUDIT-LOG` records on `deploy/openchoreo-api` stdout (the only sink wired — `cmd/openchoreo-api/main.go:213`) |
| 6 | Part D: 5 policy behaviours + 6 invalid policies, each applied by patching the ConfigMap and restarting |
| 7 | Restored the ConfigMap and verified it is byte-identical to the backup |
| 8 | Part E: observer audit via `PUT /api/v1alpha1/incidents/{id}` |
| 9 | Part F: checked the Delivery Insights and `sre` role surfaces in the live cluster |
| 10 | Deleted every project/component created by the run through the REST API |
| 11 | Re-ran D3 with `denied`/`unauthenticated` arms, then restored and re-verified the ConfigMap byte-for-byte |
| 12 | Logged in as `admin@openchoreo.dev` over PKCE and ran the four D-user cases; restored and re-verified the ConfigMap again |

The `go-greeter-service` sample was left deployed in the `default` project as a
working reference; everything else created by this run was removed.
