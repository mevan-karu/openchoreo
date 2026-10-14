# Docs changes: Thunder → ThunderID 1.0.1 (+ random credentials)

Repo: `openchoreo/openchoreo.github.io`. **Edit `docs/` only** (next/unreleased). Do not touch
`versioned_docs/version-*` — v1.2.x and older correctly document Thunder 0.28. Line numbers were read at
commit `f00b7ad`. Chart facts verified with `helm show values oci://ghcr.io/thunder-id/helm-charts/thunderid
--version 1.0.1`: `registry: ghcr.io/thunder-id`, `repository: thunderid`, chart `name: thunderid`, setup job
still a pre-install Helm hook.

## 1. Install commands: chart ref + pinned version

Old: `oci://ghcr.io/asgardeo/helm-charts/thunder` `--version 0.28.0` → new: `oci://ghcr.io/thunder-id/helm-charts/thunderid` `--version 1.0.1`.

- [ ] `docs/getting-started/try-it-out/on-k3d-locally.mdx` L219 (chart ref), L222 (version)
- [ ] `docs/getting-started/try-it-out/on-your-environment.mdx` L335 (install ref), L338 (version)
- [ ] `docs/getting-started/try-it-out/on-your-environment.mdx` L449 (HTTPRoute upgrade ref), L451 (version)
- [ ] `docs/platform-engineer-guide/air-gapped-installation.mdx` L138–139 (`helm template`), L232 (`helm pull`)

Unchanged, do not edit: namespace `thunder`, release name `thunder`, `values-thunder.yaml` URL
(air-gapped L259, on-your-environment L329, on-k3d L223 — path is still `install/k3d/common/values-thunder.yaml`).

## 2. Pod label selector

Chart name drives `app.kubernetes.io/name`; our `fullnameOverride: thunder` does **not** affect it, so the
selector becomes `thunderid` even though every other name stays `thunder`. Say this in the PR so reviewers
don't "fix" it back.

- [ ] `docs/getting-started/try-it-out/on-k3d-locally.mdx` L230 — `kubectl wait ... -l app.kubernetes.io/name=thunderid`
- [ ] `docs/getting-started/try-it-out/on-your-environment.mdx` L346 — same
- [ ] `docs/platform-engineer-guide/backstage-configuration.mdx` L740 — `kubectl get pods -n thunder -l app.kubernetes.io/name=thunderid`

## 3. Bootstrap prose (shell scripts → declarative YAML resources)

The values key is still `bootstrap.scripts`, but its contents are now YAML resource documents
(`resource_type: user|group|application|server_config`), not shell.

- [ ] `on-k3d-locally.mdx` L216 — "bootstrap scripts that run on first startup" is wrong twice over. Say:
      declarative resource documents under the chart's `bootstrap.scripts` key, applied by the setup job
      (a pre-install Helm hook) on first install.
- [ ] `on-your-environment.mdx` L326 — "pre-install helm hook" is still correct; only reword the
      users/groups/apps half to say declarative resource documents.

## 4. Service-account claim: `sub` → `client_id`

In ThunderID 1.0 a client-credentials token's `sub` is the application UUID; the client id arrives in a
separate `client_id` claim. Confirmed in-branch: control-plane `values.yaml` L1191, L1786–1851 and
observability-plane `values.yaml` L517 all use `client_id` now. **User/group bindings using `claim: groups`
are unchanged.** Every site below is a service-account example:

- [ ] `platform-engineer-guide/authorization.md` L69, L110 (`claim: "sub"` in the subject config examples)
- [ ] `platform-engineer-guide/authorization.md` L76 (prose: "the JWT `sub` claim equals `openchoreo-backstage-client`")
- [ ] `platform-engineer-guide/authorization.md` L494–498 (default-bindings table — all five `sub:` entitlements,
      e.g. `sub:openchoreo-backstage-client`, `sub:service_mcp_client`)
- [ ] `platform-engineer-guide/authorization.md` L658 (claim examples `groups`, `sub`, `email` → add/replace `client_id`)
- [ ] `platform-engineer-guide/observability-alerting.mdx` L573 (prose), L592 (`claim: sub`)
- [ ] `platform-engineer-guide/backstage-configuration.mdx` L119 (prose), L138 (`claim: sub`)
- [ ] `platform-engineer-guide/workflows/workflow-workload-configuration.mdx` L82 (prose), L113 (`claim: sub`)
- [ ] `ai/sre-agent.mdx` L145 (prose), L160 (`claim: sub`)
- [ ] `ai/finops-agent.mdx` L144 (prose), L159 (`claim: sub`)
- [ ] `ai/portal-assistant.mdx` L220 (`claim: sub`)
- [ ] `reference/api/platform/clusterauthzrolebinding.md` L39 (claim examples), L117 and L164 (service-account examples)
- [ ] `reference/api/platform/authzrolebinding.md` L40 (claim examples)

## 5. Air-gapped installation

- [ ] L115 and L342 — "Thunder" → "ThunderID" in the prose/heading
- [ ] L348 — `./thunder-0.28.0.tgz` → `./thunderid-1.0.1.tgz`
- [ ] L351 — `--set deployment.image.registry=$REGISTRY/asgardeo` → `$REGISTRY/thunder-id`
      (chart default is `registry: ghcr.io/thunder-id`, `repository: thunderid`)
- [ ] L344–345 prose ("key is nested under `deployment.`") stays true — no edit
- [ ] L552 pull-secret table row — see Needs verification

## 6. Generated Helm reference

- [ ] `docs/reference/helm/control-plane.mdx` L303 still shows default `sub` for
      `openchoreoApi.config.security.subjects.service_account.mechanisms.jwt.entitlement.claim`.
      **Do not hand-edit** — run `node scripts/generate-helm-docs.js` with the migration branch checked out as
      the sibling `openchoreo/` directory and commit the diff.
- [ ] `docs/reference/helm/observability-plane.mdx` L165 lists `subjectTypes` only as an array type, so regen
      probably yields no visible change there; the regen run is the check.

## 7. Upgrade notes — `docs/platform-engineer-guide/upgrades/v1.2-to-v1.3.mdx`

The page exists (14 lines). **Replace** L11 ("There are no backward-incompatible changes to call out for this
upgrade yet.") — do not append below it. Reuse the wording from the branch's
`thunderid-migration-pr-description.md` "Breaking change" section. It must cover:

- [ ] Hand-written `ClusterAuthzRoleBinding` / `AuthzRoleBinding` CRs and any overridden
      `bootstrap.mappings` must move from `claim: sub` to `claim: client_id`.
- [ ] Chart-managed bindings self-migrate via the post-upgrade bootstrap job — no action needed.
- [ ] Thunder 0.28 also emits `client_id`, so existing installs may stay on 0.28 and defer the IdP migration.
- [ ] External-IdP users must confirm their provider emits `client_id`, or pin the claim back to `sub`.

## 8. Credential-retrieval guidance (mirrors PR 624 — do not author, rebase onto it)

Pattern: `kubectl get secret openchoreo-initial-credentials -n thunder -o jsonpath='{.data.<key>}' | openssl base64 -d -A; echo`
plus PR 624's tip to change the password or delete the secret after first login. Keys: `admin-password`,
`developer-password`, `pe-password`, `sre-password`, `thunder-console-admin-password`, and `<name>-client-secret`
for backstage / customer-portal / rca / system-app / service-mcp / workload-publisher / observer / finops-agent.

Pages PR 624 already covers: `docs/_constants.mdx` (drops `password`), `ai/mcp-servers.mdx` L152,
`getting-started/quick-start-guide.mdx` L135–136, `try-it-out/on-k3d-locally.mdx` L160–170 + L303,
`try-it-out/on-your-environment.mdx` L135–143 + L355–357 + L659, `platform-engineer-guide/identity-configuration.mdx` L44–47.

## No change needed

- Portal login credentials — `on-k3d-locally.mdx` L889, `on-your-environment.mdx` L659, `mcp-servers.mdx` L41–42,
  `quick-start-guide.mdx` L135–136 all render the shared `defaultCredentials` constant; PR 624 handles it centrally.
- Hostnames / token URLs (`thunder.openchoreo.localhost`, `thunder.${CP_BASE_DOMAIN}`) everywhere, incl.
  `catalog-sync.mdx` L77 and `installing-into-existing-backstage.mdx` L609–618. Namespace, release name,
  `thunder-service.thunder.svc` and `helm uninstall thunder` (`on-your-environment.mdx` L1212) also unchanged.
- `claim: groups` bindings — `authorization.md` L516, L542, L612, L626, L641 — user bindings are unchanged.
- Product-name prose already says "ThunderID": `backstage-configuration.mdx` L46, `cli-configuration.mdx` L24,
  `identity-configuration.mdx` L27+, `observability-alerting.mdx` L509, `workflows/external-ci.mdx` L65+,
  `workflows/workload-generation.md` L39, `workflow-workload-configuration.mdx` L10. Not a rename exercise.
- Docs helper scripts (`docs/getting-started/try-it-out/on-k3d-locally/*.sh`) contain no Thunder references.

## Needs verification

- [ ] `air-gapped-installation.mdx` L552 — row `Thunder | deployment.image.imagePullSecret=mirror-cred`.
      `helm show values` for 1.0.1 returns **no** `imagePullSecret` key (only `pullPolicy`), so it may be gone
      rather than renamed. Check with `helm template ... --version 1.0.1 --set deployment.image.imagePullSecret=x`
      and see whether it reaches the pod spec before rewriting the row.
- [ ] `secret-management.mdx` L148 lists platform secret `thunder-client-secret`. That string has zero
      occurrences in the openchoreo repo; the real key is `backstage-client-secret`. Pre-existing staleness,
      independent of this migration — confirm with the secrets owner.
- [ ] `catalog-sync.mdx` L76 and `installing-into-existing-backstage.mdx` L608, L617 hardcode
      `clientSecret: backstage-portal-secret`. **Not** in PR 624's file list, so the random-credentials change
      leaves them stale. Raise with the 624 author rather than absorbing them here.

## Coordination

Two dependency tracks, not one rebase:

- **Track A (this migration PR):** sections 1–7. Ships independently of the credentials work.
- **Track B (openchoreo PR 3508 → docs PR 624):** section 8. The credentials work is **not** on this migration
  branch — `install/prerequisites/thunder/generate-thunder-secrets.sh` is untracked (`?? install/prerequisites/thunder/`),
  `values-thunder.yaml` still carries literal `Admin@123` / `Dev@123` / `backstage-portal-secret` /
  `service_mcp_client_secret`, and nothing reads `openchoreo-initial-credentials`. Do not author section 8 here.

Merge-conflict points where both tracks touch the same block: `identity-configuration.mdx` L44–47 and
`on-k3d-locally.mdx` L303 — both `admin`/`admin` blocks PR 624 rewrites, both in migration-adjacent prose.
Rebase this PR onto 624, or land 624 first.
