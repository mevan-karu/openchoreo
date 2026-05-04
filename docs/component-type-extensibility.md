# Reducing the Burden of Writing Custom ComponentTypes

**Related:** [Discussion #1563](https://github.com/openchoreo/openchoreo/discussions/1563) · [Epic #3106](https://github.com/openchoreo/openchoreo/issues/3106)

**Goal:** Reduce the burden on Platform Engineers when creating custom component types by avoiding the need to copy and maintain the full definition of an existing type.

---

## Context

OpenChoreo v1.0.0 ships with:

- Four default `ClusterComponentType` resources: `service`, `web-application`, `worker`, `scheduled-task`
- A `Trait` system for extending capabilities — PEs can embed traits into a `ComponentType`, and developers can attach traits to individual `Component` resources (within the bounds the PE allows)
- An `allowedTraits` field so PEs control which traits developers may attach

The trait-based extension model from [#1563](https://github.com/openchoreo/openchoreo/discussions/1563) is implemented. What remains is the ability for a `ComponentType` to build on top of another — declaring only what is different rather than duplicating everything.

---

## The Duplication Problem

### For a PE creating a custom component type

Suppose a PE wants to create a `gpu-worker` — identical to `worker` except the Deployment enforces a GPU node selector and a minimum GPU resource limit. There is no mechanism to express "start from `worker` and change only the `deployment` resource." The PE must:

1. Copy the full `worker` YAML (~200 lines)
2. Modify the `deployment` resource
3. Own and maintain the copy indefinitely — including manually tracking any future changes to the upstream `worker` type (e.g. a new label convention, a change to the ExternalSecret API version, a fix to the secret rendering logic)

The config/secret block the PE copied but never intended to change becomes a maintenance liability. Any upstream fix to those resources must be manually replicated into every custom type that forked from a default.

### Within the default component types

The same problem exists in OpenChoreo's own defaults. The following resource blocks are copy-pasted verbatim across all four default types with zero variation:

```
- id: env-config           (forEach configurations.toConfigEnvsByContainer())
- id: file-config          (forEach configurations.toConfigFileList())
- id: secret-env-external  (forEach configurations.toSecretEnvsByContainer())
- id: secret-file-external (forEach configurations.toSecretFileList())
```

Additionally, `service`, `web-application`, and `worker` all define a near-identical `deployment` resource template, differing only in the presence and shape of routing resources. Any fix to how configs, secrets, or the base Deployment are rendered must be applied to all four files independently.

---

## Desired Experience

### Platform Engineer

> *"I declare only what is different. Everything I don't mention is inherited from the parent type."*

The PE should be able to:
- Reference an existing component type as a starting point
- Add new rendered resources (e.g. a `NetworkPolicy` the parent doesn't have)
- Replace a specific resource template with a customized version (e.g. the `deployment` resource with GPU fields)
- Extend the schema to expose new developer-facing parameters
- Tighten or extend the list of allowed traits

What the PE should **not** need to do:
- Copy and maintain resource templates they aren't changing
- Track upstream changes to the parent type and manually apply them to their copy

### Developer

The developer experience must remain unchanged. A developer picks a `ComponentType` by name, fills in its schema, and creates a `Component`. Whether the component type was authored from scratch or derived from another type is invisible to them.

---

## Proposed Approaches

### Option A: `extends` field on `ComponentType` (recommended)

A `ComponentType` declares `extends: <name>` to inherit from another `ComponentType`. The child gets all parent resources, schema, validations, and traits as a baseline and can:

- **Add** new resource entries
- **Replace** an inherited resource by declaring one with the same `id`
- **Extend** `environmentConfigs` / `parameters` schema with new fields
- **Add** to `allowedTraits` and `validations`

**Example — PE creates `gpu-worker` from `worker`:**

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: ClusterComponentType
metadata:
  name: gpu-worker
spec:
  extends: worker               # inherits env-config, file-config, secret-*, and deployment

  resources:
    - id: deployment            # replaces only the deployment resource
      template:
        apiVersion: apps/v1
        kind: Deployment
        metadata:
          name: ${metadata.name}
          namespace: ${metadata.namespace}
          labels: ${metadata.labels}
        spec:
          replicas: ${environmentConfigs.replicas}
          selector:
            matchLabels: ${metadata.podSelectors}
          template:
            metadata:
              labels: ${metadata.podSelectors}
            spec:
              nodeSelector:
                accelerator: nvidia-tesla   # locked by PE
              containers:
                - name: main
                  image: ${workload.container.image}
                  imagePullPolicy: ${environmentConfigs.imagePullPolicy}
                  resources:
                    limits:
                      nvidia.com/gpu: "1"   # enforced by PE
                      cpu: ${environmentConfigs.resources.limits.cpu}
                      memory: ${environmentConfigs.resources.limits.memory}
                  env: ${dependencies.toContainerEnvs()}
                  envFrom: ${configurations.toContainerEnvFrom()}
                  volumeMounts: ${configurations.toContainerVolumeMounts()}
              volumes: ${configurations.toVolumes()}
```

The four config/secret resources are inherited unchanged. The PE writes only the `deployment` replacement.

OpenChoreo's own defaults would also benefit — `web-application` could extend `service` and only replace the routing resources, eliminating the duplicated Deployment and config/secret blocks.

**Pros:**
- Directly solves the duplication problem with minimal new concepts
- Single CRD, no new types to learn
- Replace-by-id semantics are simple and predictable
- Applies equally to platform-shipped types and PE-authored types

**Concerns:**
- Multi-level chains are possible — requires cycle detection and a decision on whether to allow unlimited depth or cap it
- When a parent changes, children pick up the change automatically — needs clear documentation
- The unit of customization is a full resource replacement by `id`. Partial patching of a resource template is not straightforward: resources using `forEach` generate a variable number of Kubernetes objects at render time, making patch targets ambiguous. Whether partial patching makes sense for non-`forEach` resources (e.g. `deployment`) can be revisited later
- When a child re-declares a schema field the parent already defines — is it an error or does the child's definition win?
- `validations` and `allowedTraits`: should child entries be merged with the parent's (union) or replace them entirely?

---

### Option B: `BaseComponentType` CRD (two CRDs, single level)

A new `BaseComponentType` CRD captures foundational resource blocks. A `ComponentType` references exactly one `BaseComponentType`. No further chaining.

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: ClusterBaseComponentType
metadata:
  name: deployment-base
spec:
  workloadType: deployment
  environmentConfigs:
    openAPIV3Schema:
      # replicas, resources, imagePullPolicy
  resources:
    - id: deployment
      template: ...
    - id: env-config
    - id: file-config
    - id: secret-env-external
    - id: secret-file-external
---
apiVersion: openchoreo.dev/v1alpha1
kind: ClusterComponentType
metadata:
  name: service
spec:
  baseType: deployment-base
  resources:
    - id: service
    - id: httproute-external
    - id: httproute-internal
  validations: ...
  allowedTraits: ...
```

**Pros:**
- Clear platform/PE boundary
- Single-level keeps reasoning simple

**Cons:**
- New CRD to maintain and document — existing `ComponentType` resources without a `baseType` field must continue to work as today, which needs explicit validation
- A PE cannot extend another PE's `ComponentType` — only platform-owned bases
- Does not eliminate routing duplication between `service` and `web-application`
- Same resource replacement and schema conflict questions apply when a `ComponentType` adds or overrides fields from its `BaseComponentType`
- `validations` and `allowedTraits`: same union vs replace question as Option A

---

### Option C: No inheritance — improve the copy workflow

Accept that structural reuse between component types is out of scope and instead reduce the friction of the copy-and-maintain workflow. This keeps the data model simple and avoids the complexity of inheritance semantics.

Suggested improvements to the PE experience under this option:

- **`occ componenttype fork <source> <new-name>`** — a CLI command that copies an existing component type, updates the name, and strips any cluster-specific annotations, giving the PE a clean starting point
- **`occ componenttype diff <type-a> <type-b>`** — a CLI command to compare two component types side by side, making it easier to understand what diverged from a base and spot unintended differences across related types
- **Well-documented base snippets** — publish the shared resource blocks (e.g. the config/secret block) as documented reference snippets that PEs can paste and know are the canonical version

---

## Comparison

| | Option A (`extends`) | Option B (`BaseComponentType`) | Option C (no inheritance) |
|---|---|---|---|
| PE creates `gpu-worker` from `worker` | Extends `worker`, replaces `deployment` | References `deployment-base`, but can't build on `worker` | Full copy of `worker` |
| Config/secret block duplication (×4) | Inherited from common parent | Owned by `BaseComponentType` | Duplicated ×4 |
| `service` vs `web-application` routing duplication | `web-application` extends `service` | Both extend `deployment-base`; routing still duplicated | Duplicated |
| PE extends another PE's custom type | Supported | Not supported | Not supported |
| New CRDs | None | 1 (`BaseComponentType`) | None |
| Inheritance depth | Multi-level (with cycle detection) | Single level | N/A |

Each approach has its own pros and cons — the best fit for OpenChoreo depends on the level of flexibility and capability we want to offer to Platform Engineers. We welcome community input to finalize the approach.
