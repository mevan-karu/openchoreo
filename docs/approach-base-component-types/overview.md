# Approach: Base Component Types

Introduce a `ClusterBaseComponentType` CRD that encodes OpenChoreo's integration contract for each workload kind. A base type defines exactly what OC requires from any workload of that kind — configuration injection, secret management, dependency wiring, and the primary workload resource — so any CT built on top of a base automatically participates in all OC platform capabilities without re-implementing them.

Default component types (and PE-created custom types) reference a base via a `baseType:` field. The CT layer declares policy on top of the base: which workflows are allowed, which traits are allowed, routing resources, and semantic validations. Base types are managed exclusively by the platform team and contain no CT-level policy.

---

## Base type design

### What base types own

- `workloadType` — determines the primary Kubernetes workload kind
- `parameters` schema — workload-category-level developer parameters (e.g. CronJob scheduling config)
- `environmentConfigs` schema — per-environment knobs (replicas, resource limits, imagePullPolicy)
- The primary workload resource template (`deployment`, `cronjob`) wired to OC's integration points: `dependencies.toContainerEnvs()`, `configurations.toContainerEnvFrom()`, `configurations.toContainerVolumeMounts()`, `configurations.toVolumes()`
- OC configuration and secret management resources: `env-config`, `file-config`, `secret-env-external`, `secret-file-external`
- K8s `Service` resource (with `includeWhen: ${size(workload.endpoints) > 0}`) — part of OC's endpoint abstraction, included conditionally so worker-like CTs get no Service for free

### What base types do NOT own

- `allowedWorkflows` — which build workflows a CT supports is a CT-level policy decision. OC ships a set of default workflows; each CT declares the subset it supports.
- Routing resources (`HTTPRoute`) — differ per component type and live in the CT
- Validations — semantic rules (e.g. "must have endpoints") belong to the CT, not the base
- `allowedTraits` — trait policy is a CT-level concern
- Embedded traits — base types do not embed traits; trait composition is done at the CT level

### Why routing stays out of the base

`service` and `web-application` both extend `base-deployment` yet need completely different routing:

| | `service` | `web-application` |
|---|---|---|
| Hostname pattern | `{env}-{namespace}.{gateway-host}` | `{dns-label}.{gateway-host}` (subdomain per component) |
| Path rules | path-prefix rewrite per endpoint | no path rules — root path |
| CEL function | none | `oc_dns_label(...)` |

Putting routing into the base would require parameterizing these differences in ways that obscure the behavior. Instead the base is routing-agnostic: each CT adds exactly the HTTPRoute resources it needs. The K8s `Service` lives in the base with `includeWhen: ${size(workload.endpoints) > 0}` — it is part of OC's endpoint abstraction and is universal enough to belong there. `worker` inherits `base-deployment` and gets no Service rendered (condition is false) and adds no routing resources.

---

## How parameters and environmentConfigs compose

The effective schema seen by a developer is the **merge of base + CT**. The CT can add new properties on top of the base's schema; it cannot remove base-defined properties.

### Merge rules

| Field | Semantics |
|---|---|
| `parameters.properties` | Union — CT properties are added to the base's; key collision is an admission error |
| `parameters.$defs` | Union — CT `$defs` are added; name collision is an admission error |
| `environmentConfigs.properties` | Same union rules as parameters |
| `environmentConfigs.$defs` | Same union rules |
| `resources` | CT resources are **additive by default**; a CT resource with the same `id` as a base resource **replaces** it entirely |
| `patches` | CT patches are applied to the merged resource set after replacements; each patch targets a base resource by `id` and applies RFC 6902 operations. Preferred over full `resources` replacement for surgical changes. |
| `validations` | CT validations are appended to any base validations (base currently has none) |
| `allowedWorkflows` | Defined entirely by the CT; base has no opinion |
| `allowedTraits` | Defined entirely by the CT; base has no opinion |

### Example — PE creates `gpu-worker`

The PE only declares what differs from the base. `patches:` applies RFC 6902 JSON Patch operations against base resources by id — no need to re-declare the full Deployment template or any of its OC integration expressions:

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: ClusterComponentType
metadata:
  name: gpu-worker
spec:
  baseType:
    kind: ClusterBaseComponentType
    name: base-deployment

  workloadType: deployment

  allowedWorkflows:
    - kind: ClusterWorkflow
      name: paketo-buildpacks-builder
    - kind: ClusterWorkflow
      name: dockerfile-builder

  # Deployment-time architectural choice — determines which GPU node pool to target.
  parameters:
    openAPIV3Schema:
      type: object
      properties:
        gpuType:
          type: string
          default: nvidia-tesla-t4
          enum: [nvidia-tesla-t4, nvidia-a100-80gb]

  # Extends base-deployment's environmentConfigs (replicas, resources, imagePullPolicy)
  # with a per-environment GPU count knob.
  environmentConfigs:
    openAPIV3Schema:
      type: object
      properties:
        gpuCount:
          type: integer
          default: 1
          minimum: 1
          maximum: 8

  # Three surgical patches — the full Deployment template and all OC integration
  # expressions are inherited from base-deployment untouched.
  patches:
    - target: deployment
      patch:
        - op: add
          path: /spec/template/spec/nodeSelector
          value:
            cloud.google.com/gke-accelerator: ${parameters.gpuType}
        - op: add
          path: /spec/template/spec/tolerations
          value:
            - key: nvidia.com/gpu
              operator: Exists
              effect: NoSchedule
        - op: add
          path: /spec/template/spec/containers/0/resources/limits/nvidia.com~1gpu
          value: "${environmentConfigs.gpuCount}"

  validations:
    - rule: "${size(workload.endpoints) == 0}"
      message: "GPU worker components must not expose endpoints."
```

The developer using `gpu-worker` sees:
- `parameters`: `{ gpuType }` — from the CT; set once at authoring time
- `environmentConfigs`: `{ replicas, resources, imagePullPolicy, gpuCount }` — base contributes the first three, CT adds `gpuCount`

The config/secret resource blocks and all OC integration expressions (`${dependencies.toContainerEnvs()}`, `${configurations.toContainerEnvFrom()}`, etc.) come from the base untouched. The CT declares only the three GPU-specific patches.

---

## Concerns and open questions

### 1. OC connection label contract cannot be enforced by the base

The core promise of base types is that OC integration contracts live in the base, not in CT templates. This holds for configuration injection, secret management, and dependency wiring — all wired directly in the base workload resource. However, the connections feature resolves dependency URLs by reading labels from rendered routing resources (e.g. `openchoreo.dev/endpoint-name`, `openchoreo.dev/endpoint-visibility` on HTTPRoutes). Since HTTPRoutes live at the CT level, the base cannot enforce these labels. A PE authoring a custom routing CT who omits them will cause external URL resolution for connections to silently fail — an OC integration contract that the base architecture cannot protect.

### 2. HTTPRoute duplication between `service` and `web-application`

The `httproute-external` and `httproute-internal` templates differ only in hostname construction (path-prefix vs subdomain) and must be maintained separately in each CT. The K8s `Service` is shared via the base; the HTTPRoutes are not. The semantic-traits approach eliminates this duplication by putting routing resources in traits; this approach does not.

### 3. Breaking change for existing component types

Introducing `ClusterBaseComponentType` and making it a required foundation for component types is a breaking change for existing CT definitions. All existing CTs (`service`, `web-application`, `worker`, `scheduled-task`) must be updated to reference a base type. Fields that the base now owns (`workloadType`, config/secret resource blocks, the base workload template) become redundant in the CT spec and will need to be removed. The `workloadType` field in `ClusterComponentTypeSpec` becomes optional when `baseType` is set, which is an API change. A migration path and versioning strategy is needed for any CTs already deployed in production clusters.

