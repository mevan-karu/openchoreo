## Purpose

Updates the docs for the ThunderID 1.0.1 migration in openchoreo/openchoreo#4751.

## Changes

- **Install guides:** point the k3d, on-your-environment and air-gapped guides at the
  `thunder-id/helm-charts/thunderid` chart (1.0.1), with the new pod selector and bootstrap wording.
- **Service account claim:** service account examples in the authorization, Backstage,
  AI agent, observability and workflow guides and the AuthzRoleBinding API references now use `client_id` instead of `sub`.
- **Upgrade guide (v1.2 → v1.3):** lists the new `client_id` defaults and the values to set
  to keep the v1.2 `sub` behavior. It also notes that moving to ThunderID is optional.
- **Default credentials:** adds a table of the default console users with their roles.
  The ThunderID `admin` password is now generated on install, so the docs replace the stale
  `admin` / `admin` login with a command to retrieve it.

## Notes

- `reference/helm/control-plane.mdx` still shows the old `sub` default. It is generated from
  the main repo, so it needs regenerating after #4751 merges.
