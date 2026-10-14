# Live cluster verification — WorkflowRun update BOLA

Cluster: k3d `openchoreo`. API: `api.openchoreo.localhost:8080`. IdP: Thunder.

## Identities used

| role | client | authz |
|---|---|---|
| attacker | `mcp-e2e-subject-client` | bound via temp `AuthzRoleBinding` to `workflowrun:view` + `workflowrun:update`, scoped to project `default` / component `bola-attacker-comp` |
| admin | `service_mcp_client` | `ClusterAuthzRole` admin (pre-existing) |

## Fixtures (namespace `default`, all prefixed `bola-`)

- `bola-victim-run` — labels `project=default`, `component=greeter-service` (attacker has NO access)
- `bola-attacker-run` — labels `project=default`, `component=bola-attacker-comp` (attacker owns)
- `bola-test-wfrun-editor` (AuthzRole), `bola-test-attacker-binding` (AuthzRoleBinding)

Both runs reference a deliberately nonexistent `ClusterWorkflow`
(`bola-test-nonexistent-workflow`) so the controller marks them
`ComponentValidationFailed` and never submits an Argo build.

---

## Run 1 — PRE-FIX baseline (exploit confirmed)

Image: `ghcr.io/openchoreo/openchoreo-api:latest-dev`,
digest `sha256:a9b081e9fcd52445b212a99b0eadef62961de396d9416b885eb48f1f384258cb`,
built **2026-08-25**. This is the upstream registry image and predates the fix, which
is uncommitted local code. So this run measures the vulnerability, not the fix.

| test | expected after fix | actual PRE-fix |
|---|---|---|
| C0 control: attacker updates its OWN run | 200 | **200** (binding works) |
| T1 attacker PUTs VICTIM run, body carries attacker's labels | 403 | **200 — THEFT SUCCEEDED** |
| T2 victim run unchanged | labels `greeter-service` | **labels `bola-attacker-comp`, annotation `t1=pwned`** |
| T3 admin relabels victim run (planting) | 4xx immutable | **200 — relabel succeeded** |
| T4 admin omits labels entirely | ownership preserved | **200, labels `null` — OWNERSHIP STRIPPED** |
| T5 positive control: annotation only | 200 | 200 |
| T6 attacker GETs victim run | 403 | 403 |

T6 returning 403 while T1 returned 200 is the crux: the same subject is correctly
denied *read* on the victim run, yet successfully *writes* it. That is the
authorization asymmetry the report described, reproduced end to end against a
running system.

T1 also confirms the escalation chain from §3 of the review: after the theft the run
carries the attacker's labels, so the attacker's subsequent `GET` / logs / events /
delete would all be legitimately authorized.

## Run 2 — POST-FIX (all pass)

Image rebuilt from the local working tree via `make k3d.update.openchoreo-api`,
digest `sha256:16ba2c4e7338754e9501640c549a7013e648dff376c6bbb27f3053ef187408ec`,
pod reports `buildTime 2026-08-31T14:10:34+0530`, `goVersion go1.26.3`.

Requests went directly to `svc/openchoreo-api` via port-forward on 18080, because
kgateway held a stale upstream after the rollout and returned 503. The gateway is not
part of the authorization path, so this does not affect the result.

| test | expected | actual | |
|---|---|---|---|
| C0 attacker updates its OWN run | 200 | 200 | PASS — binding genuinely grants update, so the 403s below are about ownership, not a broken fixture |
| T1 attacker PUTs VICTIM run with its own labels | 403 | 403 | **PASS — theft blocked (was 200)** |
| T2 victim run read back | unchanged | labels `greeter-service`, no `t1` annotation | PASS — no partial write |
| T3 admin relabels victim run | 4xx immutable | 400 `label openchoreo.dev/component is immutable` | **PASS — planting blocked** |
| T4 admin omits labels entirely | ownership preserved | 200, labels `greeter-service` carried over | PASS — no strip (was `null`) |
| T4b admin sends explicit empty label | 4xx immutable | 400 `label openchoreo.dev/component is immutable` | PASS — cannot clear via empty string |
| T5 admin updates annotation only | 200 | 200, `t5` applied | PASS — legitimate updates unaffected |
| T6 attacker GETs victim run | 403 | 403 | PASS |

Both defects are confirmed fixed end to end against a running system, and the
legitimate update path still works.

### Confirmed incidentally

The status-code nit from the review is real: the immutability rejection returns
**400 BAD_REQUEST**, whereas an equivalent CRD-level CEL rejection surfaces as 422.
Same class of error, two codes. Mirrors `component/service.go`, so it is pre-existing
precedent rather than something this change introduced.

## Cleanup

All `bola-` fixtures deleted after the run; `AuthzRole`, `AuthzRoleBinding`, and both
`WorkflowRun`s removed. No lasting change to the cluster.

