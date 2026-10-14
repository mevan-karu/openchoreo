# Audit logging — documentation and default-on plan

## Status (2026-09-17, after syncing upstream)

| Item | State |
|---|---|
| #866 (docs base) | Merged as `a667a9e`. Docs branch `docs/audit-logging` created from `upstream/main` |
| C1 k3d single-cluster + quick-start + e2e | **Merged in openchoreo #4763** — via conditional `--set` flags in `k3d-install.sh` / quick-start `.helpers.sh`, not values files. #4764 deleted `k3d-observability-plane.sh` and `k3d-prerequisites.sh` |
| C2 k3d multi-cluster README | **Open** — `install/k3d/multi-cluster/README.md` on `upstream/main` still has no audit / CP-cluster Fluent Bit steps |
| B3/B4 module READMEs | **Done upstream** (community-modules `c71518a`, #616, #618): multi-cluster audit collection, trusted producers, retention, `auditLogs.output.*` separate destination |
| A1–A4, B1, B2, C3 (docs site) | **Written, uncommitted** on `docs/audit-logging`; `npm run format:check` and `npm run build` pass |
| Generated `docs/reference/helm/*.mdx` | Not regenerated — no audit keys yet; regenerate with `scripts/generate-helm-docs.js` against an `upstream/main` checkout in a separate change |

Status: draft plan, 2026-09-17. Builds on §3 of `openchoreo-audit-delivery-plan.md` and replaces it
where the two differ (placement changed from the *Observability* sidebar category to *Platform
Setup*; MCP, CLI and k3d defaults added).

Deferred, still tracked in delivery plan §3.1: `docs/reference/audit-event-schema.mdx`,
`docs/reference/audit-pii-register.mdx`, and the `make audit.schema-docs` → site pipeline. This plan
covers configuration only.

## Base revision (`openchoreo.github.io`)

All docs-site work stacks on
[openchoreo.github.io#866](https://github.com/openchoreo/openchoreo.github.io/pull/866)
("update observability module versions and refactor installation commands", branch
`default-modules-versioning`, open against `main`; head `bc034e9`). Branch from `pr-866`
(`git fetch upstream pull/866/head:pr-866`), or from `upstream/main` once it merges. It supersedes
the earlier `341f4d3`. What it changes that this plan depends on:

- `docs/_constants.mdx` adds `versions.logsOpensearchModule`, `tracingOpensearchModule`,
  `metricsPrometheusModule`, `eventsOtelCollectorModule`. **Every module snippet in this plan uses
  `${versions.logsOpensearchModule}`**, never a literal version.
- **Snippet form (final in #866).** Module commands are a named MDX export rendered by
  `<CodeBlock>`, not an inline template literal:

  ```mdx
  export const logsCollectionEnable = `helm upgrade observability-logs-opensearch \\
    oci://ghcr.io/openchoreo/helm-charts/observability-logs-opensearch \\
    --namespace openchoreo-observability-plane \\
    --version ${versions.logsOpensearchModule} \\
    --reuse-values \\
    --set fluent-bit.enabled=true \\
    --set fluentBitCustomizations.clusterInstance=openchoreo`;

  <CodeBlock language="bash">{logsCollectionEnable}</CodeBlock>
  ```

  Continuations are `\\`; literal shell `${…}` (e.g. OTel `\${env:OPENSEARCH_USERNAME}`) is escaped
  as `\${…}`. Export names are camelCase per command (`logsModuleInstall`, `tracesModuleInstall`,
  `metricsModuleInstall`, `logsCollectionEnable`, `eventsCollectorInstall`,
  `observabilityModulesInstall`) and must be unique within a page. New audit snippets follow this
  form, e.g. `auditControlPlaneEnable`, `auditObserverEnable`.
- **Existing export names to extend, not duplicate**, for the audit flag:
  `logsCollectionEnable` exists on `on-k3d-locally.mdx` (~line 797), `on-your-environment.mdx`
  (~1140) and `air-gapped-installation.mdx` (~523). Adding `--set auditLogs.enabled=true` means
  editing that constant's last line (move the closing `` `; ``), not adding a new block.
- The Fluent Bit step sets `fluentBitCustomizations.clusterInstance=openchoreo`;
  `on-your-environment.mdx` and `air-gapped-installation.mdx` explain it must be unique per cluster.
  Audit snippets that touch Fluent Bit keep that flag.
- #866 needed a follow-up `chore: fix lint failure` because prettier reflowed a template literal in
  `air-gapped-installation.mdx` (it stripped indentation from continuation lines). Run
  `npm run format:check` and eyeball the rendered snippets after `npm run format`.
- #866's checklist leaves `npm run build` unticked; run it on the base before stacking so a
  pre-existing break isn't attributed to the audit PR.
- The site copies `on-k3d-locally/k3d-observability-plane.sh` and `k3d-prerequisites.sh` are
  **deleted**. No mirroring of repo scripts into the docs repo is needed any more; the page
  references repo files via `versions.githubRef` (`install/k3d/k3d-install.sh`,
  `single-cluster/values-cp.yaml`, `values-op.yaml`), so C1 value-file changes reach the page
  without a docs edit.

## 0. Scope

Three workstreams, each its own PR set:

| # | Workstream | Repo | Kind |
|---|---|---|---|
| A | Platform Setup guide + reference pages (MCP, CLI, concepts, authz) | `openchoreo.github.io` | Docs |
| B | Modules docs (marked **[MODULES]** below) | `openchoreo.github.io` + `community-modules` | Docs |
| C | Audit logs **on by default** in k3d single-cluster and multi-cluster setups, and the matching getting-started docs | `openchoreo` + `openchoreo.github.io` | Code + docs |

Interpretations to confirm:
- "Mark them" = tag every modules-docs item explicitly (**[MODULES]**), covering both the docs-site
  `modules/*` pages and the module READMEs.
- "Enable by default for k3d/multi-cluster" = change the k3d install scripts, values files and
  READMEs so a fresh install emits, collects and serves audit logs with no extra flags. Helm chart
  defaults (`audit.enabled: false`) stay unchanged for production installs.

## 1. The chain every page must describe

Audit is four stages. A page that covers only stage 1 leaves the reader with an empty index.

| Stage | Where it is configured | Source |
|---|---|---|
| **Emit** | CP chart `openchoreoApi.config.audit.*`; OP chart `observer.audit.*` | `install/helm/openchoreo-control-plane/values.yaml:1886`, `install/helm/openchoreo-observability-plane/values.yaml:525` |
| **Collect + store** | Logs module `auditLogs.enabled / indexPrefix / producers` (Fluent Bit must run where `openchoreo-api` and `observer` run) | `community-modules/observability-logs-opensearch/helm/values.yaml:103`, `observability-logs-openobserve/helm/values.yaml:20` |
| **Read** | Observer `POST /api/v1alpha1/audit-logs/query`, `occ auditlogs`, MCP `query_audit_logs`, Backstage UI; needs cluster-scoped `auditlogs:view` | `openapi/observability-logs-adapter-api.yaml:242`, `internal/occ/cmd/auditlogs/cmd.go`, `internal/observer/mcp/auditlogs.go` |
| **Discover** | CP `audit.observabilityPlaneRef` → metadata endpoint advertises the observer URL, so clients name no plane | `internal/openchoreo-api/services/metadata/service.go` |

Asymmetry to call out: only the CP side has `observabilityPlaneRef`; observer emits but does no
discovery.

### Sharp edges to put in the guide (all verified in source)

1. `audit.enabled: true` without `observabilityPlaneRef.kind` and `.name` fails API startup
   (`internal/openchoreo-api/config/audit.go:50-63`).
2. A wrong/missing plane does **not** fail — metadata just omits the observer URL and logs a warning,
   so `occ auditlogs` cannot find the trail.
3. `policies[]` is config-file-only; the env-var loader cannot address list elements
   (`internal/auditconfig/audit.go:24-27`). Examples must use `-f values.yaml` or `--set-json`, never
   `--set`.
4. Policies are ordered, first-match-wins. Selector values (`resources`, `operations`, `actions`,
   `actor_types`, `categories`, `surfaces`, `results`) are validated at startup — a typo fails fast.
   `actors` and `entitlements` are not validated.
5. `set:` accepts only `publish`. `category` is rejected (stamped from the operation).
6. `defaults.publish: false` is allowlist mode and fails **closed**: a mis-scoped selector silently
   drops events.
7. `actor.idClaim` (default `sub`) decides what `actor.id` holds and what `policies[].match.actors`
   and `occ auditlogs --actor` match against. Must be set identically on CP and OP.
8. `auditLogs.producers` is a security boundary: each entry lets a workload write into the trail.
   Defaults hardcode the `openchoreo-control-plane` / `openchoreo-observability-plane` namespaces
   and the `api-server` / `observer` container names; non-default namespaces silently stop
   collection.
9. OpenSearch index template and 365d retention apply on every module install, even with
   `auditLogs.enabled=false` — deliberate, so nobody "fixes" it.

## 2. Workstream A — Platform Setup guide and reference pages (`openchoreo.github.io`)

Write to `docs/` only (serves `/docs/next/`). Precedent: `952858f` added
`local-development-setup.mdx` to `docs/` + `sidebars.ts` without touching `versioned_docs/`.
Branch per *Base revision* above (local `main` has diverged from `origin/main`).

### A1. New page `docs/platform-engineer-guide/audit-logging.mdx`

Sidebar: add `"platform-engineer-guide/audit-logging"` to *Platform Setup* items
(`sidebars.ts:85-100`), after `secret-management`.

Outline (follows `observability-alerting.mdx` house shape):

1. **Overview** — what is recorded (state-modifying REST + MCP operations on openchoreo-api and
   observer; management vs authorization categories; success / failure / denied / unauthenticated),
   and that the trail is a separate signal from container logs.
2. **Architecture** — Mermaid diagram of the four stages, single-cluster and multi-cluster
   variants (who needs Fluent Bit where).
3. **Prerequisites** — observability plane installed and registered as a
   `ClusterObservabilityPlane`/`ObservabilityPlane`; a logs module with audit support
   (OpenSearch, OpenObserve) — link to A-matrix in B2.
4. **Enable audit logging** — three steps, each a `helm upgrade --reuse-values` command as an
   `export const` + `<CodeBlock>{…}</CodeBlock>` pair, using `versions.helmSource`/`helmChart` for
   the planes and `versions.logsOpensearchModule` for the module (house form from #866):
   1. Logs module: `--set auditLogs.enabled=true` (+ Fluent Bit on the CP cluster when separate).
   2. Observability plane: `observer.audit.enabled=true`.
   3. Control plane: `openchoreoApi.config.audit.enabled=true` + `observabilityPlaneRef`.
   Order matters: collection first, so the first records are not written before routing exists.
5. **Multi-cluster** — Fluent Bit on CP cluster pointed at the OP OpenSearch; `producers` stays the
   same because it matches on namespace/container, not cluster.
6. **Configuring what is published** — `defaults.publish`, `policies[]` with `match` / `set`, the
   selector field table, 3 worked examples in values-file form:
   suppress a noisy service account; publish only `authorization` category; allowlist mode.
   Include edges 3–6.
7. **Actor identity** — `actor.idClaim`, multi-issuer note (`--issuer` pairing), must match CP/OP.
8. **Access** — `auditlogs:view` is cluster-scoped and in the default `platform-engineer` role
   (`values.yaml:1514`); granting to an auditor via a custom `ClusterAuthzRole` — link
   `authorization/custom-roles`. Namespace/project filters narrow, never widen.
9. **Viewing the trail** — Backstage UI (verify page/route), `occ auditlogs` (link A3), MCP (link A2).
10. **Retention and storage** — per-backend, link out to module READMEs (B3/B4).
11. **Configuration reference** — table of CP and OP `audit.*` keys; link to generated
    `reference/helm/*`.
12. **Verify** — `kubectl logs deploy/openchoreo-api -c api-server | grep AUDIT-LOG`; index
    `audit-logs-*` exists; `occ auditlogs --since 1h` returns rows.
13. **Troubleshooting** — startup `observability_plane_ref.kind: required`; empty trail with records
    in stdout (producers/namespace mismatch, Fluent Bit not on CP cluster); `occ` cannot find observer
    (wrong ref); 403 (`auditlogs:view` missing); OpenSearch index created before template.
14. **Related documentation**.

### A2. `docs/reference/mcp-servers.mdx`

- Observability Plane tool list (line ~292): add `query_audit_logs` with filters and the
  `auditlogs:view` requirement. Update the intro sentence ("covering logs, traces, …") to include
  audit logs.
- Verify whether observer MCP has toolsets that gate it; if yes, document.
- Note that MCP tool calls on both servers are themselves audited (`surface: mcp`) and can be
  targeted by `policies[].match.surfaces`.

### A3. `docs/reference/cli-reference.md`

Hand-maintained (precedent `952858f`). Add `### auditlogs` under the appropriate section (new
"Observability" heading or next to `component logs`): usage, aliases (`audit-logs`, `auditlog`),
full flag table from `internal/occ/cmd/auditlogs/cmd.go` `addQueryFlags`, and the examples block
verbatim. Mention 24h default window, 366-day max, pagination hint on stderr.

### A4. Small cross-links

- `docs/concepts/platform-abstractions.md:73` ("all platform activity is auditable") → link A1.
- `docs/platform-engineer-guide/authorization/overview.md` — mention `auditlogs:view` in the
  auditor example (line 32) and link A1.
- `docs/platform-engineer-guide/observability-alerting.mdx` → "Related" link to A1.
- `docs/reference/helm/control-plane.mdx` / `observability-plane.mdx` are generated by
  `scripts/generate-helm-docs.js` — regenerate, do not hand-edit.

## 3. Workstream B — Modules docs **[MODULES]**

Rule from delivery plan §3.2: module READMEs are fetched at runtime, unversioned and not searchable,
so anything an operator must find by search lives in official docs; READMEs carry backend-specific
detail only.

| ID | File | Work |
|---|---|---|
| **B1 [MODULES]** | `openchoreo.github.io/docs/platform-engineer-guide/modules/observability-logging-adapter-api.mdx` | No content change — SwaggerUI embeds `openapi/observability-logs-adapter-api.yaml` from `main`, which already has `/api/v1alpha1/audit-logs/query`. Add one sentence noting the audit operation is optional. |
| **B2 [MODULES]** | `openchoreo.github.io/docs/platform-engineer-guide/modules/building-a-module.md` | New section "Supporting audit logs": the query endpoint contract, `501` as the sanctioned "not supported" answer, trusted-producer collection requirement (match on kubelet log filename, not log content), separate index/stream + long retention, and a **conformance matrix** (module → audit query supported). The PR adding audit to a module updates its row. |
| **B3 [MODULES]** | `community-modules/observability-logs-opensearch/README.md` | Already has *Enable audit log collection*, *Trusted producers*, *Configuring the audit destination*. Add: multi-cluster CP-cluster Fluent Bit example with exact flags; Retention (`365d` ISM value key); Troubleshooting entry (index created before template). |
| **B4 [MODULES]** | `community-modules/observability-logs-openobserve/README.md` | Has *Enable audit log collection* (line 171). Check parity with B3: trusted producers, stream name, `data_retention`, credentials, multi-cluster. |
| **B5 [MODULES]** | Other logs modules (`aws-cloudwatch`, `azure-loganalytics`, `gcp-cloudlogging`, `moesif`) | No audit support today — record as "not supported" rows in B2 matrix; no README change. |

## 4. Workstream C — Audit on by default in k3d setups

### C1. Single-cluster (`openchoreo` repo)

| File | Change |
|---|---|
| `install/k3d/single-cluster/values-cp.yaml` | `openchoreoApi.config.audit: { enabled: true, observabilityPlaneRef: { kind: ClusterObservabilityPlane, name: default } }` — the plane `k3d-install.sh:517` creates. Required, else startup fails. |
| `install/k3d/single-cluster/values-op.yaml` | `observer.audit.enabled: true` |
| `install/k3d/k3d-install.sh:471` | Add `--set auditLogs.enabled=true` to the Fluent Bit `--reuse-values` upgrade |
| `install/k3d/k3d-observability-plane.sh:86` | Same |
| `install/k3d/single-cluster/README.md` | Mirror the flags in the manual steps; add a "Verify audit logs" step |
| `install/quick-start/` | Check whether it installs observability; if so, same flag |

Not changed: `install/k3d/preload-images.sh:580` renders charts only to collect images, and
`auditLogs.enabled` adds Fluent Bit config, not images. There is no `make k3d` install target
(`make/k3d.mk` has only build/load/update/uninstall/status), despite CLAUDE.md. The e2e k3d setups
in `make/e2e.mk` are out of scope unless e2e should cover audit.

Installs without the observability plane (`k3d-install.sh` without `--with-observability`, or
skipping Step 7 of `on-k3d-locally.mdx`): putting the audit block in `values-cp.yaml` is safe.
`AuditObservabilityPlaneRef.validate` only checks `kind`/`name` are non-empty; the plane is
resolved lazily in the metadata service, which logs a warning on `NotFound` and omits the observer
URL. Cost: `AUDIT-LOG` lines in `api-server` stdout that nothing collects. Verify during C1 that no
other startup path resolves the ref eagerly, and state this behaviour in A1 Troubleshooting.

### C2. Multi-cluster (`openchoreo` repo)

Verified gap: `install/k3d/multi-cluster/README.md` enables Fluent Bit only on **dp** (~line 662)
and **wp** (~line 682). Neither **cp** (where `openchoreo-api` runs) nor **op** (where `observer`
runs) has Fluent Bit, so flipping `auditLogs.enabled` alone collects nothing.

| File | Change |
|---|---|
| `install/k3d/multi-cluster/values-cp.yaml` | audit enabled + `ClusterObservabilityPlane/default` ref (same C1 caveat) |
| `install/k3d/multi-cluster/values-op.yaml` | `observer.audit.enabled: true` |
| `install/k3d/multi-cluster/README.md` — OP logs module step | Add `--set fluent-bit.enabled=true --set fluentBitCustomizations.clusterInstance=openchoreo-op --set auditLogs.enabled=true` (in-cluster OpenSearch, default host) |
| `install/k3d/multi-cluster/README.md` — new CP step | Create `openchoreo-observability-plane` namespace + `opensearch-admin-credentials` secret on cp (as done for wp), then install the module with Fluent Bit only, copying the dp block's flags exactly (`host.k3d.internal`, port `11085`, vhost `opensearch.observability.openchoreo.localhost`) + `clusterInstance=openchoreo-cp` + `auditLogs.enabled=true` |
| dp / wp blocks | No `auditLogs.enabled` — no producers run there; enabling adds harmless rewrite rules but misleads |
| `/etc/machine-id` workaround | Confirm the cp node needs the same fix `k3d-install.sh` applies before Fluent Bit |

### C3. Getting-started docs (`openchoreo.github.io`)

| File | Change |
|---|---|
| `docs/getting-started/try-it-out/on-k3d-locally.mdx` — Step 7 "Enable logs collection in the configured logs module" | Extend the `logsCollectionEnable` constant (~line 797) with `--set auditLogs.enabled=true` after `clusterInstance=openchoreo`. CP/OP audit values come from the repo `values-cp.yaml` / `values-op.yaml` referenced at ~274 / ~757. Add a short "Audit logs" check under *Try it Out* linking A1 |
| `docs/getting-started/try-it-out/on-your-environment.mdx` — "Enable Fluent Bit for log collection" (~1140) | Do **not** change `logsCollectionEnable`; add an optional "Enable audit logging" admonition after its `<CodeBlock>` linking A1 |
| `docs/platform-engineer-guide/air-gapped-installation.mdx` — `logsCollectionEnable` (~523) | No image change (audit needs no extra pulls). Optional admonition linking A1; if the guide shows the flag, add it to the existing constant rather than a new block |
| `docs/platform-engineer-guide/multi-cluster-connectivity.mdx` / `deployment-topology.mdx` | Note that audit collection needs Fluent Bit on the CP cluster; link A1 multi-cluster section |
| `docs/platform-engineer-guide/local-development-setup.mdx` | Check for k3d install steps; mirror if present |

## 5. Sequencing

1. **C1 + C2** (`openchoreo`) — needs a released logs module with `auditLogs` (verify
   `0.0.0-latest-dev` includes it; OpenObserve `cfe4751` is recent).
2. **B3, B4** (`community-modules`) — independent; can go in parallel.
3. **A1, A2, A3, A4, B1, B2, C3** (`openchoreo.github.io`) — stacked on #866; one PR or split
   guide vs reference. C3 lands after C1/C2 merge, since the k3d page pulls values files from the
   repo via `versions.githubRef`.

## 6. Verification per PR

- Docs: `npm run build` (both `onBrokenLinks` and `onBrokenAnchors` throw), `npm run format` /
  prettier check.
- k3d: fresh `k3d-install.sh --with-observability` → `AUDIT-LOG` in `api-server` logs →
  `audit-logs-*` index populated → `occ auditlogs --since 1h` returns a `create_project` record
  after a test action. Same for multi-cluster via the README, including an observer-side record.
- Without `--with-observability`: CP starts cleanly (C1 open question).

## 7. Items to verify before writing

- Backstage audit logs UI page exists and its route/permission (for A1 §9).
- Observer MCP toolset gating for `query_audit_logs` (A2).
- Whether `helm-docs` output needs regenerating for the `audit.actor.idClaim` addition (A4).
- Exact OpenObserve README parity gaps (B4).
- Whether `install/quick-start` installs the observability plane (C1).
