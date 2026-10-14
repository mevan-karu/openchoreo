# Helm upgrades not rolling pods on ConfigMap changes — findings & effort estimate

## TL;DR

The fluent-bit case is a real, *provable* bug and it is **not** a missing annotation — the
annotation is present but permanently inert. Two other genuine gaps exist in the main repo and
are one-line fixes. Everything else that looks like a gap is a cert-manager mount and must be
left alone.

Recommended scope: **fluent-bit (2 community modules) + backstage + observer + 2 more community
adapters.** Estimated **2–2.5 days** including k3d verification and PR review cycles — of which the
fluent-bit work is ~1.5 days and *everything else is about 2 hours of one-line edits*.

Surveyed both repos (`install/helm` and all ~38 community modules), matching every reference form —
volume mounts (`configMap:`/`secretName:`), `envFrom` (`configMapRef`/`secretRef`) and individual
env vars (`configMapKeyRef`/`secretKeyRef`). **Six real gaps across 6 files, one of them hard.**

---

## 1. The fluent-bit bug (root cause, proven)

Affects `observability-logs-opensearch` and `observability-logs-openobserve` — identical structure
in both (`helm/templates/fluent-bit/config.yaml`, `existingConfigMap: "fluent-bit"`).

Both parent charts render the fluent-bit ConfigMap themselves and point the upstream subchart at it:

```yaml
# helm/values.yaml
fluent-bit:
  existingConfigMap: "fluent-bit"
```

The upstream fluent-bit chart (v0.56.0) *does* have a checksum annotation:

```
charts/fluent-bit/templates/daemonset.yaml:37
  checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

…but `charts/fluent-bit/templates/configmap.yaml:1` is guarded by
`{{- if not .Values.existingConfigMap -}}`. Because we set `existingConfigMap`, that template
renders **empty**, so the annotation is the SHA-256 of the empty string — a constant, forever.

Verified by rendering the chart twice with materially different configs. Rendering with
`grep -E "^kind:|^  name:|checksum/config"` to attribute each hash to its resource:

```
$ helm template t . --set fluent-bit.enabled=true --set openSearchSetup.enabled=false \
    --set fluentBitCustomizations.clusterInstance=aaa
kind: DaemonSet
  name: fluent-bit
        checksum/config: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855   <-- inert
kind: Deployment
  name: logs-adapter-opensearch
        checksum/config: 8df95a76e6a3660abdcd9ce68684f648d169d993ae37f7763f292eff377a3be5

# run 2: different cluster instance AND auditLogs.enabled=true
$ helm template t . --set fluent-bit.enabled=true --set openSearchSetup.enabled=false \
    --set fluentBitCustomizations.clusterInstance=zzz-TOTALLY-DIFFERENT --set auditLogs.enabled=true
        checksum/config: e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855   <-- unchanged
        checksum/config: 8df95a76e6a3660abdcd9ce68684f648d169d993ae37f7763f292eff377a3be5
```

`e3b0c442…b855` is the well-known SHA-256 of `""`. The DaemonSet annotation never changes, so
`helm upgrade` never rolls fluent-bit pods no matter what happens to the config.

(The adapter's `8df95a76…` also holds steady across both runs, but that one is correct, not a second
inert hash: `templates/adapter/configmap.yaml` depends only on `.Values.adapter.observerUrl`, which
neither run changed. It is not audit-aware.)

### Why the obvious fixes don't work

| Approach | Verdict |
|---|---|
| Set `fluent-bit.podAnnotations` with a checksum | **Impossible.** `values.yaml` is static YAML; Helm cannot compute a hash there. Upstream `toYaml`s `podAnnotations` (daemonset.yaml:42) rather than `tpl`-ing it, so a template expression would be emitted literally. |
| Content-hash the ConfigMap name (`fluent-bit-<hash>`) | **Impossible.** `existingConfigMap` is consumed raw at `_pod.tpl:145`, not `tpl`-ed, so it must be a literal. |
| Move config into upstream's `config.service/inputs/filters/outputs` | **Correct but expensive.** Upstream `tpl`s those with `$` = the *subchart* context, where `.Values` is only the `fluent-bit:` subtree. Our template depends on `.Values.auditLogs.enabled`, `.Values.auditLogs.producers`, `.Values.fluentBitCustomizations.clusterInstance` and `include "…auditIndexPrefix" .` — none resolve there, and they fail **silently** (audit pipeline just vanishes). Requires re-parameterizing the whole audit block under `global`, which collides head-on with the in-flight audit-logging epic. |
| `fluent-bit.hotReload.enabled: true` | Changes semantics. Per daemonset.yaml:34, enabling it *removes* the checksum annotation by design, and it needs `Hot_Reload On` in `[SERVICE]`. Moves us away from "helm upgrade rolls pods" rather than fixing it. |

### Recommended fix — post-upgrade hook Job that patches the hash in

A parent-side hook Job, rendered in the **parent** context (so it has full access to
`auditLogs`, `fluentBitCustomizations`, and the chart's helpers), which stamps the real hash onto
the DaemonSet's pod template:

```
kubectl patch daemonset fluent-bit --type=strategic -p \
  '{"spec":{"template":{"metadata":{"annotations":{
     "checksum/openchoreo-config":"{{ include (print $.Template.BasePath "/fluent-bit/config.yaml") . | sha256sum }}"}}}}}'
```

Why this one:

- **Content-aware** — the hash is templated at render time, so the patch is a no-op when the
  config is unchanged. No restart-on-every-upgrade.
- **Zero coupling** to upstream subchart internals. Survives fluent-bit chart bumps.
- **Doesn't touch the audit config**, so it does not collide with the audit epic.
- Helm's three-way merge preserves the annotation across upgrades — it is present only in live
  state, never in the old or new manifest, and Helm only prunes fields that were in the *old*
  manifest and dropped from the new one.
- There is a **direct in-repo precedent to copy**: `openchoreo-control-plane/templates/authz/
  bootstrap-job.yaml` + `bootstrap-rbac.yaml` (distroless kubectl image, `post-install,post-upgrade`
  hook, `hook-weight` ordering, `before-hook-creation,hook-succeeded` delete policy, SA + Role +
  RoleBinding).

Needs a namespaced Role with `daemonsets` `get`/`patch` — narrower than the authz precedent's
ClusterRole.

**Two implementation details to settle before coding:**

1. The Job template needs the same `{{- if (index .Values "fluent-bit").enabled }}` guard as
   `config.yaml`, or it will try to patch a DaemonSet that was never installed.
2. **Decide the failure mode explicitly.** Today a stale config is a silent no-op. With a hook, a
   failed patch (RBAC wrong, kubectl image unreachable in an airgapped install, DaemonSet not yet
   created) **fails the whole `helm upgrade`**, and under `--atomic` rolls the module back. Options:
   modest `backoffLimit` and let it fail loudly, or tolerate-and-warn (`|| true` in the container
   command) so a patch failure degrades to today's behaviour rather than breaking upgrades.
   Recommend tolerate-and-warn for a log collector; this should be a decision in review, not a
   surprise.

**Scope limitation to state up front:** this hook hashes `/fluent-bit/config.yaml` only. fluent-bit
also takes `OPENSEARCH_USERNAME`/`OPENSEARCH_PASSWORD` via `secretKeyRef` on
`opensearch-admin-credentials` (`values.yaml`, `fluent-bit.env`), and that Secret is *not* rendered
by this chart — the chart only renders `opensearch-security-config`. So a credential rotation still
will not roll the DaemonSet. "fluent-bit now rolls on config change" must not be read as
"fluent-bit now rolls." If credential rotation needs to roll pods too, that is a separate item.

---

## 2. Main-repo gaps (`install/helm/`) — both real, both trivial

**Backstage** — `openchoreo-control-plane/templates/backstage/deployment.yaml` mounts three
chart-rendered ConfigMaps and carries **zero** checksum annotations:

- `…-ci-config` (`configmap-ci.yaml`) — deployment.yaml:303
- `…-csp-config` (`configmap-csp.yaml`) — deployment.yaml:307
- `…-extra-config` (`extra-config-configmap.yaml`) — deployment.yaml:312

This is almost certainly the symptom that prompted this investigation: commit `558d242b3`
("allow the logo CDN in ThunderID CSP") changes `configmap-csp.yaml`, and backstage would not
have restarted to pick it up.

**Observer** — `openchoreo-observability-plane/templates/observer/observer-deployment.yaml:24`
checksums `observer-config.yaml` but **not** `observer-auth-configmap.yaml`, which it also mounts
(`observer-auth-config`, deployment line 124). Partial coverage.

---

## 2b. Community-module gaps beyond fluent-bit — two, both trivial

Surveyed all ~38 modules for workloads that mount chart-rendered config without a checksum. Two
real hits, both of the same class as backstage, both `envFrom` (worse than a volume mount: env vars
are read once at process start, so the pod can *never* pick the change up without a restart):

- **`observability-tracing-opensearch`** — `templates/adapter/deployment.yaml:37` consumes
  `envFrom.configMapRef: tracing-adapter-opensearch`, which the same chart renders at
  `templates/adapter/configmap.yaml:8`. Zero checksum annotations.
- **`observability-tracing-aws-xray`** — `templates/adapter/deployment.yaml:42` consumes
  `envFrom.secretRef: {{ .Values.awsCredentials.name }}`, rendered by the same chart at
  `templates/aws-credentials/secret.yaml` when `awsCredentials.create` is set. Zero checksums, so
  rotating AWS credentials through `helm upgrade` would not restart the adapter.

The other ~14 adapter deployments already checksum their own ConfigMaps correctly.

## 3. What is NOT a gap — do not "fix" these

A naive grep for "workload with config mounts but no checksum" flags four more deployments. All
four are false positives: every one of their mounts is a **cert-manager-issued Secret or a
controller-populated ConfigMap**, not chart-rendered config.

| Template | Mounts | Why to leave it |
|---|---|---|
| `control-plane/cluster-gateway/deployment.yaml` | `clusterGateway.tls.secretName`, `internalMtls.caSecretName` | cert-manager Secrets. Chart doesn't render them; nothing to hash. |
| `control-plane/controller-manager/deployment.yaml` | webhook-server-cert, gateway CA, client TLS | Same — all cert-manager. |
| `{data,observability,workflow}-plane/cluster-agent/deployment.yaml` | `tls.clientSecretName`, `tls.serverCAConfigMap` | Externally supplied. |
| `cluster-gateway/ca-configmap-placeholder.yaml` | — | Chart renders an **empty placeholder** that the CA-extractor hook job overwrites at runtime. Hashing the template would hash the placeholder and force a restart on every upgrade while never tracking the real content. |

Cert rotation is a genuinely different problem with a different solution (cert-manager
`reloader`-style controllers, or relying on the app to re-read from disk). Explicitly out of scope.

### Chart-rendered Secrets: one deliberate exclusion

`openchoreo-api` mounts `remote-connect-signing-key`, which **this chart does render**
(`templates/openchoreo-api/remote-connect-signing-key.yaml`) — so it is not a cert-manager case and
deserves its own reasoning. It should still be left without a checksum:

- The key is `genPrivateKey "ed25519"`, but guarded by a `lookup` that reuses the stored value so
  upgrades don't invalidate live capabilities. It is therefore stable *in a live cluster* but
  **not stable under `helm template`/`--dry-run`**, where `lookup` returns empty and a fresh key is
  generated every render. A checksum over it would churn on every dry-run and diff.
- It carries `helm.sh/resource-policy: keep` and never changes after first install, so there is
  nothing for a restart to pick up.

Generated-secret templates are the one category where adding a checksum is actively harmful — an
ungated `genPrivateKey`/`randAlphaNum` would restart the workload on *every* upgrade. Worth stating
as a rule for reviewers rather than leaving to be rediscovered.

### Vendored subcharts: audited only where we drive them

`survey.sh` excludes `charts/`, so upstream subchart templates were not audited — which is right
(we don't own that code) but leaves one thing worth naming, since the fluent-bit bug lived exactly
at that boundary. The `helm template` output above shows `StatefulSet/opensearch-master` carrying
**no** `checksum/config` while the same render produces `ConfigMap/opensearch-master-config`. Same
class of gap, different subchart.

Deliberately out of scope: rolling an OpenSearch StatefulSet on a config change is a data-bearing
operation that needs considered handling, not an automatic restart. Recording it here so a reviewer
doesn't read its absence as a survey miss.

### Externally-supplied Secrets consumed via `secretKeyRef`

Backstage sources 11 env vars via `secretKeyRef` from `.Values.backstage.secretName`
(deployment.yaml:89–244). That Secret is **not** rendered by this chart — `templates/backstage/`
contains no `kind: Secret` — so it is out of scope for the same reason as the cert-manager mounts.
Backstage's fix is exactly the three ConfigMaps in §2, nothing more.

Already-correct, no action: `openchoreo-api`, `portal-assistant`, `event-forwarder`,
`finops-agent`, `sre-agent`, and 14 of the 16 community-module adapter deployments.

---

## 3b. GitOps / FluxCD interaction — no flag needed

Checked because a `post-upgrade` hook Job is imperative and could plausibly conflict with GitOps
reconciliation. It does not, on the shipped path. Three independent confirmations:

1. **The `gitops-flux-cd` module runs Kustomization only, never HelmRelease.** Its README states
   the design decision outright — *"Only source-controller and kustomize-controller are required.
   Helm and notification controllers are not needed since OpenChoreo manages deployments through
   its own CRDs"* — and the documented install sets `--set helmController.create=false`.
2. **What it syncs is OpenChoreo CRs**, not charts: Components, ReleaseBindings, Environments,
   DeploymentPipelines, ComponentTypes, Traits (`platform-kustomization.yaml`,
   `projects-kustomization.yaml`).
3. **Modules are installed by Helm directly** — `helm upgrade --install observability-logs-opensearch …`
   in both the module README and `make/e2e.mk:632`. No `HelmRelease` exists anywhere in either repo
   outside the e2e Kustomization test helper.

So in GitOps mode there is no Helm release for Flux to manage, the fluent-bit DaemonSet is never in
a Flux inventory, and Flux's drift correction — which only acts on resources it applied — has no
reason to touch the patched annotation. **Nothing to disable, no flag required.**

### If platform charts are ever moved under a Flux `HelmRelease`

Not the documented path, but worth recording since it's the one configuration where this interacts:

- **Hooks still run.** `spec.install.disableHooks` and `spec.upgrade.disableHooks` both default to
  `false` (helm-controller v2 API). The patch Job works unchanged.
- **Drift detection is off by default.** `spec.driftDetection.mode` defaults to Disabled, so there
  is no interference out of the box.
- If drift detection *is* enabled, it compares via server-side dry-run apply. SSA should preserve a
  field owned by a different field manager (our `kubectl patch`), so the annotation ought to
  survive — but this is **unverified and should be tested empirically** before relying on it. The
  mitigation is one line: a `driftDetection.ignore` rule targeting the annotation path.

### Known tradeoffs of the hook approach

Both are inherent to doing this imperatively, and neither is severe enough to change the
recommendation — but reviewers should know:

- **The restart is invisible to `helm template` / `helm diff`.** The annotation is applied by the
  Job at runtime, not declared in the chart, so anyone reviewing rendered manifests will not see
  the rollout coming.
- **`--no-hooks` (or `disableHooks: true`) degrades silently** — the patch simply never runs and
  behaviour reverts to today's stale config, with no error or warning. Same failure shape as the
  bug being fixed.

Rejected on these grounds and still rejected: moving `auditLogs`/`fluentBitCustomizations` under
`global` to make upstream's checksum work (§1) would be declarative and GitOps-pure, but it is a
breaking values-API change across two modules that risks silently collapsing the audit pipeline,
and it collides with the in-flight audit epic. GitOps hygiene does not outweigh that.

## 4. Effort estimate

| Work item | Estimate |
|---|---|
| fluent-bit hook Job + RBAC in `observability-logs-opensearch` (incl. k3d verification) | 0.5–1 day |
| Same, replicated to `observability-logs-openobserve` (copy-paste) | 2 hours |
| Backstage: 3 checksum lines | 30 min |
| Observer: 1 checksum line | 15 min |
| `observability-tracing-opensearch` + `observability-tracing-aws-xray`: 1 line each | 30 min |
| Verification pass — `helm template` diffs + live k3d upgrade proving pods roll on change and hold steady on no-change | 0.5 day |
| **Total** | **~2–2.5 days** incl. PR review cycles |

Three PRs, since the repos are separate and the risk profiles differ:

1. **`openchoreo`** — backstage + observer. ~45 min, near-zero risk. **Land this first and
   independently**; it is the one that matches the CSP commit that prompted the investigation.
2. **`community-modules`** — the two tracing adapters. ~30 min, near-zero risk. Independent.
3. **`community-modules`** — fluent-bit hook Job in both logs modules. The only one needing design
   review and k3d verification; keep it separate so the trivial fixes aren't held up behind it.

## 5. Verification method

1. `helm template` the chart, change one config value, `helm template` again, diff the annotations —
   the checksum must move.
2. `helm template` twice with *identical* values — the checksum must be byte-identical (guards
   against a `randAlphaNum`-style unstable hash forcing restarts on every upgrade).
3. In k3d: `helm upgrade` with a config change, confirm pods actually roll; `helm upgrade` again
   with no change, confirm they do **not**.
