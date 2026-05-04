# Implementation: Base Component Types

Reference for implementing the base component types approach described in `overview.md`.

---

## New CRD: `ClusterBaseComponentType`

```go
type ClusterBaseComponentTypeSpec struct {
    // WorkloadType is immutable after creation.
    WorkloadType       string             `json:"workloadType"`
    Parameters         *SchemaSection     `json:"parameters,omitempty"`
    EnvironmentConfigs *SchemaSection     `json:"environmentConfigs,omitempty"`
    // Resources must contain a primary resource whose id matches workloadType.
    Resources          []ResourceTemplate `json:"resources"`
}
```

No `allowedWorkflows`, `allowedTraits`, or `validations` — these are CT-level policy concerns. Base types are OC integration containers, not policy owners.

---

## Updated `ClusterComponentTypeSpec`

Add `baseType` and `patches` fields:

```go
type ClusterBaseComponentTypeRef struct {
    Kind string `json:"kind"` // always ClusterBaseComponentType
    Name string `json:"name"`
}

type ResourcePatch struct {
    // Target is the id of the base resource to patch.
    Target string      `json:"target"`
    Patch  []RFC6902Op `json:"patch"`
}

// In ClusterComponentTypeSpec:
BaseType *ClusterBaseComponentTypeRef `json:"baseType,omitempty"`
Patches  []ResourcePatch              `json:"patches,omitempty"`
```

When `baseType` is set:
- `workloadType` is optional in the CT spec; if declared it must match the base's
- The kubebuilder rule requiring `resources` to contain a primary workload resource is relaxed — if the base provides it, the CT need not repeat it
- `resources` and `patches` are both optional; if omitted, only the base resources are rendered
- `patches` are preferred over full `resources` replacement for surgical changes — the base resource's OC integration expressions are preserved without re-declaration

---

## Flattening at snapshot time

Flatten base + CT into a single `ComponentTypeSpec` at ComponentRelease snapshot creation time, inside `validateAndFetchComponentType()` in `internal/controller/component/controller.go`. The rendered `ComponentRelease.spec.componentType.spec` contains the fully merged effective spec. The rendering pipeline reads only from the frozen snapshot — zero changes needed in `internal/pipeline/component/`.

A `FlattenWithBase(ctx, client, ct ClusterComponentType) (ComponentTypeSpec, error)` function in a new `internal/componenttype/` package handles:
1. Fetch `ClusterBaseComponentType` by `ct.Spec.BaseType.Name`
2. Start with base resources as the ordered list
3. For each CT resource: if `id` matches a base resource, replace it in-place; otherwise append
4. For each CT patch: locate the target base resource by `id`, apply RFC 6902 operations to its template
5. Merge `parameters` and `environmentConfigs` JSON schemas (union of `properties` and `$defs`)
6. Append CT `validations` to base's (base currently has none)
7. Pass through CT `allowedWorkflows` and `allowedTraits` unchanged — base has no contribution
8. Return the merged `ComponentTypeSpec`

---

## Admission webhook

On `CREATE`/`UPDATE` of `ClusterComponentType` with `baseType` set:

1. Fetch the referenced `ClusterBaseComponentType` — error if not found
2. Validate `workloadType` match (if CT declares it)
3. Run `FlattenWithBase` as a dry run
4. Validate the merged schema for `$defs` and `properties` name collisions — error on conflict
5. Validate all CEL expressions in the CT's `resources` and `patches` against the **merged** schema (base + CT). A patch value of `${parameters.gpuType}` is valid only if `gpuType` exists in the merged parameters schema.
6. Validate that each `patches[].target` refers to an existing base resource `id` — error on unknown targets

---

## Component controller: watch `ClusterBaseComponentType`

Add a watch for `ClusterBaseComponentType` changes, mapping to all components whose CT references that base:

```go
Watches(&openchoreov1alpha1.ClusterBaseComponentType{},
    handler.EnqueueRequestsFromMapFunc(r.listComponentsForClusterBaseComponentType))
```

`listComponentsForClusterBaseComponentType` needs a new field index on `ClusterComponentType` keyed by `spec.baseType.name`, then a second lookup from CT names to components. This is a two-hop index lookup but follows the same pattern as existing watches.

---

## `ClusterBaseComponentType` controller

Can remain a no-op (same as `ClusterComponentType` controller today) — base types are pure data and are not reconciled.

---

## Immutability enforcement

The CRD marker alone cannot enforce "PE-immutable" — that requires RBAC:

- Base types are created by the platform team using a dedicated `ServiceAccount` with `ClusterRole` that includes `update`/`delete` verbs on `clusterbasecomponenttypes`
- PE `ClusterRole` grants only `get`/`list`/`watch` on `clusterbasecomponenttypes`
- An admission webhook can additionally enforce an immutability annotation: resources with `openchoreo.dev/immutable: "true"` reject all update requests except from a designated system SA

---

## `resources` MinItems=1 CRD constraint

The current `ClusterComponentTypeSpec` CRD has `// +kubebuilder:validation:MinItems=1` on the `resources` field. This must be relaxed when `baseType` is set — implement as a webhook rule: when `baseType` is present, `resources` is optional; when absent, `resources` must contain at least one entry whose `id` matches `workloadType`. Remove the kubebuilder marker and enforce entirely in the webhook.
