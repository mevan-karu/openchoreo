# Approach: Semantic Traits

Introduce platform-managed `ClusterTrait` resources that group logically related Kubernetes resources. Component types embed these traits via the `traits:` field rather than duplicating the resource definitions inline.

## What changes

Three new platform-managed traits:

- `configuration` — creates the four config/secret resource blocks (`env-config`, `file-config`, `secret-env-external`, `secret-file-external`) and patches the workload to inject dependency connection env vars (`dependencies.toContainerEnvs()`), `envFrom` references, volume mounts, and volumes. Applies to both `Deployment` and `CronJob` targets. Patch operations use JSON Patch `add` with append semantics so they compose safely with anything already declared in a custom component type's workload template.
- `service-routing` — creates the HTTPRoute resources using path-prefix routing on a shared hostname and carries the gateway validation rules that are only meaningful when this routing trait is in use.
- `webapp-routing` — creates HTTPRoute resources using subdomain routing via `oc_dns_label` and carries the HTTP endpoint validation rule. Kept separate from `service-routing` because the two types have fundamentally different routing semantics.

The Kubernetes `Service` resource is defined in the component type itself, not in the routing traits. Routing traits only own the HTTPRoute resources. This keeps the Service declaration close to the Deployment and makes it visible without opening the trait definition.

Component types declare only the resources that are unique to them (the `Deployment` or `CronJob`) and the `Service` for endpoint-serving types. No placeholder arrays are needed in workload templates — the patch engine (`ensureParentExists`) creates missing target arrays automatically when appending.

## Issues and Concerns

- Readability takes a hit. To fully understand what a component type renders, you have to read the component type file and then open each embedded trait. There is no single file that shows the complete picture.
- The `Deployment` resource is still duplicated across `worker`, `service`, and `web-application`. Traits don't help here because the deployment is the primary resource and is unique per type in spirit, even if the template is identical today.
- **Trait updates do not automatically propagate to existing deployments.** The component controller's trait watch index only covers traits attached directly to a `Component` (`spec.traits`). Traits embedded in a `ComponentType` via `CT.spec.traits` are not indexed against components, so updating an embedded trait does not re-trigger reconciliation. A PE must also touch the CT (or trigger a manual redeploy) for the change to take effect.
- Even when a CT change does trigger re-reconciliation, the updated render reaches the cluster automatically only if the component has `autoDeploy: true`. Components with `autoDeploy: false` require an explicit configure-and-deploy action.
- The two points above together mean the operational benefit of centralising a fix in one trait is primarily a source-of-truth benefit, not an automatic propagation benefit. Deployments still require a deliberate promotion step.
