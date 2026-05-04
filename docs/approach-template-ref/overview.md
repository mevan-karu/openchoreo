# Approach: templateRef for Resource Definitions

Allow individual resource entries within a component type to reference an externally defined template via a `templateRef` field, pointing to a new `ClusterResourceTemplate` CRD. Resource definitions that are shared across component types live in template objects; unique resources remain inline.

## What changes

A new `ClusterResourceTemplate` CRD holds a single resource definition (including `id`, optional `forEach`/`var`, and `template`). Component type resource lists can mix `templateRef` entries (pointing to a shared template) with inline `template` entries (for resources unique to that type).

## Issues and Concerns

- Introduces a new CRD (`ClusterResourceTemplate`) that needs to be maintained, documented, versioned, and validated.
- Readability is improved compared to the traits approach (the component type still shows the full resource list), but understanding the full rendered output still requires opening the referenced template files.
- When a `ClusterResourceTemplate` changes, every component type referencing it is affected. There is no per-component-type pinning or versioning — all consumers move together.
- The `Deployment` resource is still duplicated across `worker`, `service`, and `web-application` at the template level (all three point at `standard-deployment`), but the template body is now shared. If the deployment shape needs to differ per type, the PE would need separate template objects or fall back to inline.
- A PE authoring a custom component type still needs to know which `ClusterResourceTemplate` names exist and what they expect in terms of context variables (`environmentConfigs`, `configurations`, etc.). This knowledge is not surfaced by the component type itself.
- Deletion or rename of a `ClusterResourceTemplate` would silently break all component types that reference it unless validation is added to catch dangling refs at admission time.
