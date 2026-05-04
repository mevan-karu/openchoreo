# ComponentType `extends` — Implementation Guide

**Tracks:** [Epic #3106](https://github.com/openchoreo/openchoreo/issues/3106) · [Design doc](./component-type-extensibility.md)

This document covers the implementation plan for Option A (`extends` field on `ComponentType`). It is intended as a working reference during development.

---

## Architectural fit

The component release flow is:

```
Component → (controller) → validateAndFetchComponentType()
                         → ComponentRelease (frozen snapshot of full ComponentTypeSpec)
                         → ReleaseBinding + rendering pipeline (reads from frozen snapshot)
                         → RenderedRelease
```

**Key insight:** The ComponentType controller (`internal/controller/componenttype/controller.go` and `clustercomponenttype/`) is a no-op — ComponentTypes are pure data definitions, never reconciled. The rendering pipeline (`internal/pipeline/component/pipeline.go`) works entirely from the frozen `ComponentTypeSpec` stored in `ComponentReleaseSpec.ComponentType.Spec` (`api/v1alpha1/componentrelease_types.go:62`).

This means: **flatten the inheritance chain at snapshot time**. The rendering pipeline needs zero changes. Historical releases correctly reflect the effective spec at snapshot time, including which parent version was active.

**Single injection point:** `validateAndFetchComponentType()` in `internal/controller/component/controller.go:211`. Everything downstream consumes the frozen, already-flattened spec.

---

## Implementation tasks

### 1. API types

**Files:**
- `api/v1alpha1/componenttype_types.go`
- `api/v1alpha1/clustercomponenttype_types.go`

Add an `Extends` field to `ComponentTypeSpec`:

```go
// Extends optionally names a parent ComponentType to inherit resources, schema,
// validations, and traits from. Resources with the same id override the parent's.
// +optional
Extends *ComponentTypeRef `json:"extends,omitempty"`
```

For `ClusterComponentTypeSpec`, the reference kind must be restricted to `ClusterComponentType` only (cluster-scoped types cannot reference namespace-scoped parents). For namespace-scoped `ComponentTypeSpec`, either kind is valid.

**Remove this kubebuilder marker** from `ComponentTypeSpec` (it must move to the webhook because the primary resource may now be inherited):

```go
// +kubebuilder:validation:XValidation:rule="self.workloadType == 'proxy' || self.resources.exists(r, r.id == self.workloadType)"
```

Run `make code.gen` after any type changes to regenerate CRD manifests and deepcopy.

---

### 2. Core: `FlattenComponentTypeSpec`

**New file:** `internal/componenttype/flatten.go`

This is the heart of the feature. The function walks the `extends` chain, resolves it recursively, and returns a merged `ComponentTypeSpec` with all inheritance applied.

**Signature:**

```go
// FlattenSpec resolves the extends chain for a ClusterComponentType and returns
// a fully merged ComponentTypeSpec. The returned spec is safe to snapshot into
// a ComponentRelease. Returns an error if the chain contains a cycle or if any
// parent cannot be fetched.
func FlattenSpec(ctx context.Context, c client.Client, spec ClusterComponentTypeSpec, selfName string) (ComponentTypeSpec, error)
```

**Merge rules by field:**

| Field | Strategy |
|---|---|
| `workloadType` | Inherited from root ancestor; child must match if declared |
| `resources` | Parent list as base; child entry with same `id` replaces parent's; new child `id`s append in declaration order |
| `environmentConfigs.openAPIV3Schema` | JSON deep-merge: parent `$defs` + `properties` as base, child fields override on key conflict |
| `parameters.openAPIV3Schema` | Same JSON deep-merge as above |
| `validations` | Union — parent rules first, child rules appended; child cannot remove parent rules |
| `allowedTraits` | Union — deduped by `(kind, name)` pair |
| `allowedWorkflows` | Union — deduped by `(kind, name)` pair; child cannot narrow parent's list (document this) |
| `traits` (embedded) | Union — `instanceName` collision between parent and child is a validation error |

**Cycle detection:**

```go
func resolveChain(ctx context.Context, c client.Client, name string, visited map[string]bool) ([]ComponentTypeSpec, error) {
    if visited[name] {
        return nil, fmt.Errorf("cycle detected in extends chain at %q", name)
    }
    visited[name] = true
    // fetch parent, recurse
}
```

Cap maximum chain depth (e.g. 10) to bound webhook latency.

---

### 3. JSON Schema merging

**New function:** `internal/componenttype/schema.go`

The `openAPIV3Schema` fields are stored as `*runtime.RawExtension` (raw JSON). There is no existing merge utility in the codebase.

```go
// mergeOpenAPIV3Schema merges parent and child raw JSON schemas.
// Child properties override parent properties on key conflict.
// $defs from parent and child are merged; name collision is an error.
func mergeOpenAPIV3Schema(parent, child []byte) ([]byte, error)
```

**What to merge:**
- `properties` map: child keys win on conflict
- `$defs` map: child keys win on conflict; if a PE reuses a `$def` name with a different definition, return an error (forbid silent conflicts in v1)
- `required` array: union
- `default`, `type`, `description` at the root level: child wins if present

**What not to merge:**
- Do not attempt deep structural merging of individual property schemas — if a child redeclares a property, its entire sub-schema replaces the parent's. Full replacement per key is simpler and predictable.

---

### 4. Admission webhook changes

**Files:**
- `internal/webhook/componenttype/webhook.go` (create if absent)
- `internal/webhook/clustercomponenttype/webhook.go` (create if absent)

On `CREATE` and `UPDATE`:

1. **Resolve parent** (if `extends` is set): fetch the named parent and verify it exists.
2. **Cycle detection**: walk the full chain from this type upward; fail if a cycle is found.
3. **`workloadType` match**: if the child declares `workloadType`, it must equal the resolved parent's `workloadType`.
4. **`$def` name collision check**: run `mergeOpenAPIV3Schema` as a dry-run to surface schema conflicts early.
5. **Validate flattened CEL**: call `ValidateClusterComponentTypeResourcesWithSchema` (already in `internal/validation/component/componenttype.go`) with the **merged** schema, not just the child's schema. This ensures CEL expressions that reference parent-defined schema fields are validated correctly.
6. **`instanceName` collision in embedded traits**: check that no trait `instanceName` in the child duplicates one in the resolved parent chain.

---

### 5. Component controller: inject flattening

**File:** `internal/controller/component/controller.go`

In `validateAndFetchComponentType()` (line 211), after fetching the `ClusterComponentType`, call:

```go
flatSpec, err := componenttype.FlattenSpec(ctx, r.Client, ct.Spec, ct.Name)
if err != nil {
    // set condition, return
}
ct.Spec = flatSpec
```

The rest of the function and all callers downstream receive the flattened spec transparently.

The same change is needed in the API service path that also constructs `ComponentRelease` resources (`internal/openchoreo-api/`).

---

### 6. `workloadType` and primary-resource constraint

The existing kubebuilder rule (moved to the webhook in task 1) must be evaluated against the **flattened** resources list, not just the child's declared resources. After merging, the check becomes:

```
flattened.workloadType == 'proxy' || flattened.resources.exists(r, r.id == flattened.workloadType)
```

If the primary resource is inherited from the parent and the child doesn't re-declare it, this check passes on the flattened list.

**`workloadType` when `extends` is set:**
- If the child omits `workloadType`, it inherits the parent's value. This requires making `workloadType` optional in the OpenAPI schema when `extends` is present — implement as a webhook cross-field validation rather than a kubebuilder enum marker change.
- If the child declares `workloadType`, it must match the parent's (validated in the webhook).

---

### 7. `occ` CLI (optional, lower priority)

The `occ` CLI (`cmd/occ/`, `internal/occ/`) should gain two subcommands:

- `occ componenttype tree <name>` — prints the resolved inheritance chain with each ancestor's contribution
- `occ componenttype diff <type-a> <type-b>` — side-by-side diff of two types' flattened specs

These are convenience features; they do not block the core implementation.

---

## Default component type evolution

With `extends` available, the four default types in `samples/getting-started/component-types/` should be refactored as follows.

### New: `base-deployment` (internal, not intended for direct developer use)

```yaml
apiVersion: openchoreo.dev/v1alpha1
kind: ClusterComponentType
metadata:
  name: base-deployment
spec:
  workloadType: deployment
  allowedWorkflows:
    - kind: ClusterWorkflow
      name: paketo-buildpacks-builder
    - kind: ClusterWorkflow
      name: gcp-buildpacks-builder
    - kind: ClusterWorkflow
      name: dockerfile-builder
    - kind: ClusterWorkflow
      name: ballerina-buildpack-builder
  environmentConfigs:
    openAPIV3Schema:
      # replicas, resources (cpu/memory), imagePullPolicy — current shared schema
  resources:
    - id: deployment        # current shared deployment template
    - id: env-config        # identical across all four types today
    - id: file-config
    - id: secret-env-external
    - id: secret-file-external
```

### `worker` extends `base-deployment`

```yaml
spec:
  extends: base-deployment
  allowedTraits:
    - kind: ClusterTrait
      name: observability-alert-rule
  validations:
    - rule: "${size(workload.endpoints) == 0}"
      message: "Worker components must not have endpoints."
```

### `service` extends `base-deployment`

```yaml
spec:
  extends: base-deployment
  allowedTraits:
    - kind: ClusterTrait
      name: observability-alert-rule
  validations:
    - rule: "${size(workload.endpoints) > 0}"
      message: "Service components must have at least one endpoint."
    - rule: >-  # gateway checks (unchanged)
  resources:
    - id: service           # new — not in base
    - id: httproute-external
    - id: httproute-internal
```

### `web-application` extends `service`

```yaml
spec:
  extends: service
  # Removes ballerina from allowedWorkflows — note: union semantics mean this
  # cannot narrow the parent list. web-application must declare allowedWorkflows
  # explicitly if restriction is needed, OR accept ballerina as allowed.
  validations:
    - rule: "${workload.endpoints.exists(name, endpoint, endpoint.type == 'HTTP')}"
      message: "Web-application components must have at least one HTTP endpoint."
  resources:
    - id: httproute-external   # replaces service's version — different hostname format
    - id: httproute-internal   # replaces service's version
```

### `scheduled-task` — stays standalone

`workloadType: cronjob` prevents it from extending any of the `deployment`-based types. The four config/secret resource blocks remain duplicated in this type. This is an accepted limitation of Option A.

---

## Open questions to resolve before implementation

1. **`allowedWorkflows` narrowing**: Should a child type be able to restrict the parent's list (e.g. `web-application` excluding `ballerina`)? Union semantics cannot support this. Options: (a) accept the limitation and document it, (b) add an `excludeWorkflows` field, (c) treat `allowedWorkflows` as child-replaces-parent. **Recommendation:** accept the limitation for v1 and document it clearly.

2. **Schema field conflict policy**: If a child redeclares a `$def` or property that the parent already defines — is it an error, or does the child win silently? **Recommendation:** child wins silently for `properties` (expected override behavior); `$def` name collision with a different structure is an error (prevents silent schema corruption).

3. **`workloadType` optionality**: Making `workloadType` optional when `extends` is set requires a kubebuilder validation change and a webhook cross-field rule. Alternatively, require child to always declare `workloadType` explicitly (simpler validation, slightly more verbose for PEs). **Recommendation:** require explicit declaration in v1; make it optional in a follow-up once the pattern is established.

4. **Namespace-scoped `ComponentType` extending `ClusterComponentType`**: This cross-scope reference is the common PE pattern (cluster-wide base, project-local override). It should be supported but requires the fetch logic to check both scopes. Confirm this is in scope for v1.

5. **Status field for resolved chain**: Add `status.resolvedAncestors []string` to `ClusterComponentTypeStatus` to make the inheritance chain observable via `kubectl get`. Low implementation cost, high operational value.
