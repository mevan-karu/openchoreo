# Migrate identity provider from Thunder to ThunderID 1.0.1

## What

Migrates the in-cluster identity provider from Thunder 0.28.0 to ThunderID 1.0.1.

The chart moves to `oci://ghcr.io/thunder-id/helm-charts/thunderid`, pinned to 1.0.1.
The `thunder` namespace, release name, service DNS names and hostnames are unchanged.

## Changes

- Point the installers (k3d, quick-start, kiac, e2e) and docs at the ThunderID chart.
- Rewrite the IdP bootstrap: ThunderID 1.0 replaces shell scripts with declarative
  resource documents, so users, groups, OAuth apps and CORS are now YAML resources.
- Update the k3d/e2e values for the new config layout: four databases, nested sqlite
  settings, and CORS as a `server_config` resource.
- Switch service-account authorization from the `sub` claim to `client_id` across the
  control-plane and observability-plane charts, agent configs and e2e fixtures.
- Update the OAuth application payloads for the ThunderID API.

## Breaking change

In ThunderID 1.0 a client-credentials token's `sub` is the application UUID, and the
client ID is only in `client_id`. The service-account claim defaults move to `client_id`
in both charts, so the change lands on the v1.3 `helm upgrade` for any install relying
on those defaults, whichever IdP it uses. Default bindings are rewritten by the
post-upgrade bootstrap job; the installs that must act are:

- those with service-account bindings outside the chart defaults (hand-written or
  API-created bindings, or overridden `bootstrap.mappings`) — `claim: sub` becomes
  `claim: client_id`;
- those whose IdP does not emit `client_id`, or that already set the claim explicitly —
  they set the claim in both charts and restate `bootstrap.mappings` to match (as the
  Dex e2e overlay does).

Existing installs can stay on Thunder 0.28: it already emits `client_id`, so no IdP
migration is required for this release. The v1.3 upgrade guide covers each case.

## Verification

Installed on a local k3d cluster with all planes: the bootstrap seeds all users, groups
and apps, Backstage OIDC login succeeds, and issued tokens carry the expected `groups`
and `client_id` claims.

## Notes for reviewers

- Docs changes ship separately on the `docs/thunderid-1.0.1-migration` branch in
  `openchoreo/openchoreo.github.io`.
- The generated Helm reference (`docs/reference/helm/control-plane.mdx`) still shows the
  old `sub` default. It is regenerated from this repo's `values.schema.json`, so it has to
  be regenerated once this branch lands on main.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
