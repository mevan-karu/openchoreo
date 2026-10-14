# Audit trail on OpenObserve — install + verification, 2026-10-02

Single-cluster k3d `openchoreo`, control plane **1.3.1**, observability plane **1.3.1**,
logs module **observability-logs-openobserve 0.7.1** (chart `0.7.1` from
`oci://ghcr.io/openchoreo/helm-charts`, matching the 0.7.x row of the module's compatibility
table for OpenChoreo v1.3.0+).

**42 audit records, zero `unknown` actors, all three result types.** The trail is served from
OpenObserve end to end: emitted by `openchoreo-api` → Fluent Bit → OpenObserve `audit_logs`
stream → `logs-adapter-openobserve` → observer → CLI / portal.

## What was installed

1. Namespace `openchoreo-observability-plane` + `cluster-gateway-ca` ConfigMap.
2. ExternalSecrets `observer-secret` and `openobserve-admin-credentials`. The module expects one
   secret with `ZO_ROOT_USER_EMAIL` / `ZO_ROOT_USER_PASSWORD`; OpenBao already held
   `openobserve-admin-email` and `openobserve-admin-password`, so the ExternalSecret maps the two
   entries into those keys.
3. `openchoreo-observability-plane` 1.3.1 with `install/k3d/single-cluster/values-op.yaml`.
4. `observability-logs-openobserve` 0.7.1 with `fluent-bit.enabled=true`,
   `fluentBitCustomizations.clusterInstance=openchoreo`, **`auditLogs.enabled=true`**.
5. `ClusterObservabilityPlane/default` registered, pointing at `http://observer.openchoreo.localhost:11080`.
6. Control plane upgraded to turn audit emission on:
   `openchoreoApi.config.audit.enabled=true`,
   `openchoreoApi.config.audit.observabilityPlaneRef={kind: ClusterObservabilityPlane, name: default}`.
   The api pod rolled automatically on the config change.

The `audit_logs` stream was created by the setup job with 365-day retention, and a direct
OpenObserve search (`SELECT ... FROM "audit_logs"`) returns the same 42 records the observer
serves. Note the stream's `doc_num` stat still reads 0 — OpenObserve updates those stats lazily,
so the stat is not a way to check ingestion.

## Result

| surface | actor.type | actor.id | user_agent | records |
|---|---|---|---|---|
| mcp | service_account | service_mcp_client | `Python-urllib/3.14` | 14 |
| rest | service_account | service_mcp_client | `Python-urllib/3.14` | 2 |
| mcp | user | admin@openchoreo.dev | `Python-urllib/3.14` | 7 |
| rest | user | admin@openchoreo.dev | `Go-http-client/1.1` (occ) | 8 |
| rest | user | admin@openchoreo.dev | `node` (Backstage) | 8 |
| rest | user | developer@openchoreo.dev | `Go-http-client/1.1` (occ) | 3 |

37 success, 3 denied, 2 failure. 11 distinct actions.

### Filters, served by the OpenObserve adapter

| filter | records |
|---|---|
| `actor.id: [service_mcp_client]` | 16 |
| `actor.id: [admin@openchoreo.dev]` | 23 |
| `actor.id: [developer@openchoreo.dev]` | 3 |
| `actor.type: [service_account]` / `[user]` | 16 / 26 |
| `result: [denied]` / `[failure]` | 3 / 2 |
| `surface: [mcp]` / `[rest]` | 21 / 21 |
| `action: [create_project]` | 6 |
| `category: [management]` | 42 |
| `producer: [openchoreo-api]` | 42 |
| `resource.namespace: [default]` | 41 |

Every count cross-checks against the surface/actor breakdown. The one record without a namespace
is the developer's denied `create_namespace`, which targets a cluster-scoped resource.

The portal's Audit Logs page shows all 42 with actor IDs, surfaces and `failure` badges.

## Operations exercised, then cleaned up

- **Backstage UI** (admin): create project + service component via scaffolder; delete both via UI.
- **CLI `occ`** (admin): apply project + release binding, scaffold/apply component, create and
  update environment, delete component / environment / project.
- **CLI `occ`** (developer): create namespace, delete admin's project, update environment — all
  **denied**; `occ auditlogs` refused for lacking `auditlogs:view`.
- **MCP** (service account ×2 batches, admin user ×1): 7-op create/update/delete batch each.
- **REST** (service account): create + delete project.

No `audit3-*` resources remain.

## Issues hit during the install

- **Fluent Bit cannot start on a k3d node without `/etc/machine-id`** — the pod sits in
  `Init:0/1` with `MountVolume.SetUp failed for volume "etcmachineid": hostPath type check failed`.
  The repo's own install path covers this (`install/k3d/k3d-install.sh:438`, single-cluster README
  step 1), but this cluster had been created without it. Fixed with:
  `docker exec k3d-openchoreo-server-0 sh -c "[ -s /etc/machine-id ] || cat /proc/sys/kernel/random/uuid | tr -d '-' > /etc/machine-id"`.
  Until this was fixed nothing shipped, and the audit trail read back empty — worth a line in the
  module README's limitations, since the failure looks like an audit problem rather than a node one.
- **`cluster-gateway-ca` in the control plane still holds the chart's placeholder text**
  ("# Placeholder certificate - will be replaced by actual CA certificate"), so the README's
  copy-the-ConfigMap step hands the observability plane an invalid CA and
  `cluster-agent-observabilityplane` crash-loops with
  `failed to parse server CA certificate /ca-certs/ca.crt: no valid certificates found`.
  The issued CA is in the `cluster-gateway-tls` Secret (`ca.crt`); copying that instead fixed it.
  This does not affect the audit path — the observer reads the adapter directly — but it does
  break the agent link.
- **The cluster was control-plane only**: no ClusterComponentTypes, pipelines or environments, so
  component operations failed at first (these produced the 2 `failure` records, which turned out
  useful for coverage). Applying `samples/getting-started/all.yaml` **from the working tree
  failed** against the installed 1.3.1 controller — the webhook rejects
  `${workload.toContainerPorts()}` as an undeclared CEL reference, because the local checkout is
  ahead of the release. The v1.3.1 tag's copy of the same file applies cleanly.
- No data plane is installed, so environments and components exist but do not reconcile. That does
  not affect audit records, which are emitted by the API at write time.

## Carried over from the previous run (not OpenObserve-specific)

- `occ auditlogs` still applies the context's project scope, so a bare query returns
  "No audit records matched the query" with the default context. PR #4831 fixes this; the released
  1.3.0 CLI used here does not have it. Naming `--project` works around it.
- `occ` still sends `Go-http-client/1.1`; Backstage is distinguishable only as `node`.
- A denied audit-log read is still not itself audited.

Local occ contexts: `default` now points at credential `oo-admin`; `oo-dev` holds the developer login.
