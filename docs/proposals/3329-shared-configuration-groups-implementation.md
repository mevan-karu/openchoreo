# Shared Configuration Groups — Implementation Plan

**Related Proposal**: [3329-shared-configuration-groups.md](./3329-shared-configuration-groups.md)
**Issue**: https://github.com/openchoreo/openchoreo/issues/3329

---

## Phase 1: API types (`api/v1alpha1`)

### 1.1 New file: `configurationgroup_types.go`

Introduces the `ConfigurationGroup` CRD and all supporting types.

```go
// ConfigurationGroupOwner identifies the project scope of a ConfigurationGroup.
// When ProjectName is set, the group is visible only within that project.
// When omitted, the group is visible to all projects in the namespace.
type ConfigurationGroupOwner struct {
    // +optional
    ProjectName string `json:"projectName,omitempty"`
}

// EnvironmentGroup is a named alias for a set of environments, used to avoid
// repeating the same value entry per environment.
type EnvironmentGroup struct {
    Name         string   `json:"name"`
    Environments []string `json:"environments"`
}

// ConfigurationValueFrom holds indirect sources for a configuration value.
type ConfigurationValueFrom struct {
    // +optional
    SecretKeyRef *SecretKeyRef `json:"secretKeyRef,omitempty"`
}

// ConfigurationValue is a single environment-scoped value for a configuration key.
// Exactly one of Environment or EnvironmentGroupRef must be set.
// Exactly one of Value or ValueFrom must be set.
// +kubebuilder:validation:XValidation:rule="has(self.environment) != has(self.environmentGroupRef)",message="exactly one of environment or environmentGroupRef must be set"
// +kubebuilder:validation:XValidation:rule="!(has(self.value) && has(self.valueFrom))",message="value and valueFrom are mutually exclusive"
type ConfigurationValue struct {
    // Specific environment this value applies to.
    // +optional
    Environment string `json:"environment,omitempty"`

    // Named environment group this value applies to.
    // Mutually exclusive with environment.
    // +optional
    EnvironmentGroupRef string `json:"environmentGroupRef,omitempty"`

    // Plain-text string value or inline file content.
    // +optional
    Value string `json:"value,omitempty"`

    // Indirect value source (e.g. a secret reference).
    // +optional
    ValueFrom *ConfigurationValueFrom `json:"valueFrom,omitempty"`
}

// ConfigurationEntry is a single keyed configuration with per-environment values.
type ConfigurationEntry struct {
    Key    string               `json:"key"`
    Values []ConfigurationValue `json:"values"`
}

// ConfigurationGroupSpec defines the desired state of a ConfigurationGroup.
type ConfigurationGroupSpec struct {
    // +optional
    Owner *ConfigurationGroupOwner `json:"owner,omitempty"`

    // Named environment group aliases to reduce repetition in values.
    // +optional
    EnvironmentGroups []EnvironmentGroup `json:"environmentGroups,omitempty"`

    // Flat list of keyed configurations with per-environment values.
    Configurations []ConfigurationEntry `json:"configurations"`
}

// ConfigurationGroupStatus defines the observed state of a ConfigurationGroup.
type ConfigurationGroupStatus struct {
    // Name of the latest ConfigurationGroupVersion child CR (e.g. "v3").
    // Used for user display and to protect the current version from deletion.
    // Resolution of version: latest reads the ConfigurationGroup spec directly
    // and does not use this field.
    // +optional
    CurrentVersion string `json:"currentVersion,omitempty"`
}

// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:resource:scope=Namespaced
// +kubebuilder:printcolumn:name="Current Version",type=string,JSONPath=`.status.currentVersion`
// +kubebuilder:printcolumn:name="Project",type=string,JSONPath=`.spec.owner.projectName`
type ConfigurationGroup struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec   ConfigurationGroupSpec   `json:"spec,omitempty"`
    Status ConfigurationGroupStatus `json:"status,omitempty"`
}

// +kubebuilder:object:root=true
type ConfigurationGroupList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []ConfigurationGroup `json:"items"`
}
```

### 1.2 New file: `configurationgroupversion_types.go`

Introduces the `ConfigurationGroupVersion` CRD — an immutable snapshot of a
`ConfigurationGroup` spec. Created automatically by the controller; never hand-edited.

The spec hash of the parent at snapshot time is stored as a label
(`configurationgroup.openchoreo.dev/spec-hash`) on each `ConfigurationGroupVersion` CR.
The controller uses this label (not a field in the parent status) to detect whether the
current spec already has a corresponding version, following the standard Kubernetes
Deployment → ReplicaSet pattern.

```go
// ConfigurationGroupVersionSpec is a frozen copy of a ConfigurationGroup spec
// at the time a change was detected.
type ConfigurationGroupVersionSpec struct {
    // Copied from the parent ConfigurationGroup at snapshot time.
    // +optional
    Owner *ConfigurationGroupOwner `json:"owner,omitempty"`

    // Sequential version identifier (e.g. "v2").
    Version string `json:"version"`

    // Frozen copy of the parent's environmentGroups at snapshot time.
    // +optional
    EnvironmentGroups []EnvironmentGroup `json:"environmentGroups,omitempty"`

    // Frozen copy of the parent's configurations at snapshot time.
    Configurations []ConfigurationEntry `json:"configurations"`
}

// +kubebuilder:object:root=true
// +kubebuilder:resource:scope=Namespaced
// +kubebuilder:printcolumn:name="Version",type=string,JSONPath=`.spec.version`
// +kubebuilder:printcolumn:name="Project",type=string,JSONPath=`.spec.owner.projectName`
type ConfigurationGroupVersion struct {
    metav1.TypeMeta   `json:",inline"`
    metav1.ObjectMeta `json:"metadata,omitempty"`

    Spec ConfigurationGroupVersionSpec `json:"spec,omitempty"`
}

// +kubebuilder:object:root=true
type ConfigurationGroupVersionList struct {
    metav1.TypeMeta `json:",inline"`
    metav1.ListMeta `json:"metadata,omitempty"`
    Items           []ConfigurationGroupVersion `json:"items"`
}
```

**Labels on `ConfigurationGroupVersion`:**

| Label | Value | Purpose |
|---|---|---|
| `configurationgroup.openchoreo.dev/parent` | `<group-name>` | List all versions for a given group |
| `configurationgroup.openchoreo.dev/spec-hash` | `<hash>` | Detect duplicate versions; idempotent reconcile |

### 1.3 New shared type: `ConfigurationGroupRef`

Used in Workload references. Add to `types.go` or a new `configurationgroupref_types.go`.

```go
// ConfigurationGroupRef references a ConfigurationGroup or a specific
// ConfigurationGroupVersion for injection into a workload.
type ConfigurationGroupRef struct {
    // Name of the ConfigurationGroup.
    Name string `json:"name"`

    // Version to resolve. Defaults to "latest", which reads the ConfigurationGroup
    // spec directly. A named version (e.g. "v2") resolves from the corresponding
    // ConfigurationGroupVersion child CR.
    // +optional
    // +kubebuilder:default=latest
    Version string `json:"version,omitempty"`

    // Key within the group to inject. Required for individual key references
    // (env[].valueFrom and files[].valueFrom). Must be omitted for bulk
    // references (envFrom[] and filesFrom[]).
    // +optional
    Key string `json:"key,omitempty"`
}
```

### 1.4 Update `EnvVarValueFrom` in `workload_types.go`

`FileVar` reuses `EnvVarValueFrom` for its `ValueFrom` field, so this single change
covers both individual env and individual file references.

```go
type EnvVarValueFrom struct {
    // +optional
    SecretKeyRef *SecretKeyRef `json:"secretKeyRef,omitempty"`

    // Reference to a ConfigurationGroup or ConfigurationGroupVersion.
    // +optional
    ConfigurationGroupRef *ConfigurationGroupRef `json:"configurationGroupRef,omitempty"`
}
```

Add a CEL validation rule to `EnvVar` and `FileVar` to ensure `secretKeyRef` and
`configurationGroupRef` are not both set:

```
// +kubebuilder:validation:XValidation:rule="!(has(self.secretKeyRef) && has(self.configurationGroupRef))",message="secretKeyRef and configurationGroupRef are mutually exclusive"
```

### 1.5 New types for bulk injection in `workload_types.go`

```go
// EnvFromSource injects all keys from a ConfigurationGroup as environment variables.
type EnvFromSource struct {
    // +optional
    ConfigurationGroupRef *ConfigurationGroupRef `json:"configurationGroupRef,omitempty"`
}

// FileFromSource injects all keys from a ConfigurationGroup as files under a directory.
type FileFromSource struct {
    // Base directory under which all group keys are mounted as files.
    MountPath string `json:"mountPath"`

    // +optional
    ConfigurationGroupRef *ConfigurationGroupRef `json:"configurationGroupRef,omitempty"`
}
```

### 1.6 Update `Container` in `workload_types.go`

```go
type Container struct {
    // ... existing fields ...

    // Bulk-inject all keys from a ConfigurationGroup as environment variables.
    // +optional
    EnvFrom []EnvFromSource `json:"envFrom,omitempty"`

    // Bulk-inject all keys from a ConfigurationGroup as files under a directory.
    // +optional
    FilesFrom []FileFromSource `json:"filesFrom,omitempty"`
}
```

---

## Phase 2: `ConfigurationGroup` controller (`internal/controller/configurationgroup`)

New controller package. Responsible for auto-versioning and environment coverage validation.
Does not perform resolution — that is Phase 3.

### 2.1 File structure

| File | Responsibility |
|---|---|
| `controller.go` | `Reconciler` struct, `Reconcile` loop, watch setup |
| `controller_hash.go` | `ComputeConfigurationGroupHash` — wraps the shared `hash.ComputeHash` utility |
| `controller_version.go` | Version number determination, `ConfigurationGroupVersion` child CR construction |
| `controller_validation.go` | Environment coverage validation against `DeploymentPipeline` |
| `controller_test.go` + `suite_test.go` | envtest-based tests |

### 2.2 RBAC markers

```go
// +kubebuilder:rbac:groups=openchoreo.dev,resources=configurationgroups,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=openchoreo.dev,resources=configurationgroups/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=openchoreo.dev,resources=configurationgroups/finalizers,verbs=update
// +kubebuilder:rbac:groups=openchoreo.dev,resources=configurationgroupversions,verbs=get;list;watch;create;delete
// +kubebuilder:rbac:groups=openchoreo.dev,resources=deploymentpipelines,verbs=get;list;watch
// +kubebuilder:rbac:groups=openchoreo.dev,resources=projects,verbs=get;list;watch
```

### 2.3 Watch setup

The pipeline-to-project relationship lives on the `Project` CR (`spec.deploymentPipelineRef`),
not on the `DeploymentPipeline` itself. This means there are three distinct triggers for
coverage re-validation, each requiring a separate watch.

```go
func (r *Reconciler) SetupWithManager(mgr ctrl.Manager) error {
    return ctrl.NewControllerManagedBy(mgr).
        For(&v1alpha1.ConfigurationGroup{}).
        // Re-queue the parent when a child ConfigurationGroupVersion is deleted.
        Owns(&v1alpha1.ConfigurationGroupVersion{}).
        // Trigger 2: re-validate when a project attaches or changes its pipeline reference.
        // This covers the case where a pipeline is created and then attached to a project
        // via project.spec.deploymentPipelineRef — that attachment is a Project update,
        // not a DeploymentPipeline event.
        Watches(&v1alpha1.Project{}, handler.EnqueueRequestsFromMapFunc(
            r.configurationGroupsForProject,
        )).
        // Trigger 3: re-validate when an already-attached pipeline's environment list changes.
        // A DeploymentPipeline create event alone does not re-queue anything — no groups
        // are affected until a Project references it, which is caught by Trigger 2.
        Watches(&v1alpha1.DeploymentPipeline{}, handler.EnqueueRequestsFromMapFunc(
            r.configurationGroupsForPipeline,
        )).
        Complete(r)
}
```

**`configurationGroupsForProject`** — maps a `Project` update to all project-scoped
`ConfigurationGroup` CRs where `spec.owner.projectName == project.name`. Handles the
pipeline attach/change case.

**`configurationGroupsForPipeline`** — maps a `DeploymentPipeline` update to all
project-scoped `ConfigurationGroup` CRs whose project references that pipeline:
1. Find all `Project` CRs where `spec.deploymentPipelineRef.name == pipeline.name`
2. For each such project, list `ConfigurationGroup` CRs with `spec.owner.projectName == project.name`
3. Re-queue all of them

Does not fire on `DeploymentPipeline` create events since no project references it yet.

### 2.4 Reconcile loop

```
Fetch ConfigurationGroup
  → not found: return (already deleted)

Handle deletion
  → owner references on child ConfigurationGroupVersion CRs handle garbage collection
  → no custom finalizer needed unless additional cleanup is required

Compute spec hash
  → hash.ComputeHash(spec.Configurations + spec.EnvironmentGroups)

List existing ConfigurationGroupVersion CRs
  → label selector: configurationgroup.openchoreo.dev/parent=<name>

Check for existing version with matching hash
  → if found: ensure status.currentVersion is set; skip to coverage validation

No matching version found → create new ConfigurationGroupVersion
  → determine next version number (see 2.5)
  → build child CR with frozen spec, owner reference, and labels
  → update status.currentVersion

Run environment coverage validation (see 2.6)        ← always runs, not just on new version
  → only for project-scoped groups (spec.owner.projectName is set)
  → set CoverageValid condition on ConfigurationGroup status
  → triggered by: group spec change, Project.deploymentPipelineRef change,
    or DeploymentPipeline environment list change
```

### 2.5 Version number determination (`controller_version.go`)

```
List all ConfigurationGroupVersion CRs for this parent
  → label selector: configurationgroup.openchoreo.dev/parent=<name>

For each child, parse spec.version ("v3" → 3)
Take max; next = max + 1
If no children exist, next = 1

New child name: {group-name}-v{next}
New child spec.version: "v{next}"
```

**Owner reference on child:**

```go
metav1.OwnerReference{
    APIVersion:         groupVersion,
    Kind:               "ConfigurationGroup",
    Name:               parent.Name,
    UID:                parent.UID,
    Controller:         ptr(true),
    BlockOwnerDeletion: ptr(true),
}
```

This means deleting a `ConfigurationGroup` cascades to all its `ConfigurationGroupVersion`
child CRs via Kubernetes garbage collection.

**Labels set on each new `ConfigurationGroupVersion`:**

```go
labels.Set{
    "configurationgroup.openchoreo.dev/parent":    parent.Name,
    "configurationgroup.openchoreo.dev/spec-hash": currentHash,
}
```

### 2.6 Environment coverage validation (`controller_validation.go`)

Applies only to project-scoped groups (`spec.owner.projectName` is set). Namespace-wide
group coverage is validated at reference time in the ReleaseBinding controller (Phase 3).

**Algorithm:**

```
Fetch the Project by spec.owner.projectName
  → if not found: set CoverageValid=Unknown, return

Fetch the DeploymentPipeline via project.spec.deploymentPipelineRef.name
  → if not found or not yet attached: set CoverageValid=Unknown, return

Collect all environment names from the pipeline

For each ConfigurationEntry in spec.configurations:
  For each pipeline environment:
    Resolve which ConfigurationValues cover it:
      - direct match: value.environment == envName
      - group match: value.environmentGroupRef resolves to a group whose
        spec.environmentGroups[].environments contains envName
    If no value covers this environment → record as missing

If any missing entries found:
  → set condition ConfigurationGroupCoverageValid=False with details
Else:
  → set condition ConfigurationGroupCoverageValid=True
```

Whether this validation blocks version creation (hard rejection) or is advisory
(condition only) is deferred to Open Concern #2. The controller sets the condition
in both cases; blocking can be enforced via a validating webhook layered on top.

### 2.7 Status conditions on `ConfigurationGroup`

Add to `ConfigurationGroupStatus`:

```go
Conditions []metav1.Condition `json:"conditions,omitempty"`
```

| Condition type | Status | Meaning |
|---|---|---|
| `Ready` | `True` / `False` | Controller has successfully reconciled and at least one version exists |
| `CoverageValid` | `True` | All pipeline environments are covered |
| `CoverageValid` | `False` | One or more pipeline environments have no matching value |
| `CoverageValid` | `Unknown` | Project or pipeline not yet found; validation deferred |

---

## Phase 3: `ReleaseBinding` controller updates (`internal/controller/releasebinding`)

Config group resolution is a deployment-time operation. It runs when a `ReleaseBinding` is
reconciled — on first deployment (creation) and on every subsequent promotion (update). No
watches on `ConfigurationGroup` or `ConfigurationGroupVersion` are added; the controller has
no responsibility to react to group changes between deployments. Re-deploying a component is
the explicit trigger for picking up group changes.

### 3.1 New file: `controller_configgroup.go`

**`fetchConfigurationGroupSpec(ctx, name, version, namespace) (ConfigurationGroupSpec, resolvedVersion, error)`**

Fetches the right spec depending on the version reference:
- `version == "latest"` or empty → fetch `ConfigurationGroup` CR directly; return its
  `spec` and `status.currentVersion` as the resolved version name (for recording in the snapshot)
- `version == "v2"` (any named version) → fetch `ConfigurationGroupVersion` CR named
  `{name}-v2`; return its `spec` and `"v2"` as the resolved version name

Both paths return the same `ConfigurationGroupSpec` shape so callers need no branching.

**`resolveValueForEnvironment(entry ConfigurationEntry, envName string, envGroups []EnvironmentGroup) *ConfigurationValue`**

Finds the matching `ConfigurationValue` for a target environment within a single entry:
- Direct `environment == envName` match wins over an `environmentGroupRef` match
- Returns `nil` if no value covers the environment

**`resolveConfigurationGroups(ctx, workload, envName, projectName, namespace) (resolvedWorkload, error)`**

Main resolution function. Builds a resolved workload with all `configurationGroupRef` entries
replaced by concrete `value` or `valueFrom.secretKeyRef` entries. Applies the override chain
by building a keyed map and writing sources in precedence order:

```
1. envFrom[]   referencing namespace-wide groups   (broadest)
2. envFrom[]   referencing project-scoped groups
3. env[]       inline values (no configurationGroupRef)
4. env[]       individual configurationGroupRef refs
5. ReleaseBinding WorkloadOverrides.Container.Env  (most specific)
```

Same ordering applies to `filesFrom[]` / `files[]` / `WorkloadOverrides.Container.Files`.

Scope is determined by checking whether the fetched group's `spec.owner.projectName` is set.

For namespace-wide groups, if `resolveValueForEnvironment` returns `nil` for the target
environment, return an error — the reference cannot be resolved and the deployment should fail
with a clear message naming the group and the missing environment.

### 3.2 Where resolution fits in the existing reconcile flow

```
buildWorkloadFromRelease(componentRelease)        ← existing
        ↓
resolveConfigurationGroups(snapshotWorkload, ...)  ← new, replaces all configurationGroupRef
        ↓                                            entries with concrete values/secretKeyRefs
collectSecretReferences(snapshotWorkload, ...)    ← existing, now sees a fully resolved workload
        ↓
RenderInput{...}                                  ← existing, unchanged
```

### 3.3 Recording resolved versions in the RenderedRelease

After resolution, the controller records which version was actually used for each
`configurationGroupRef` in the `RenderedRelease` annotation or status, so the snapshot is
fully auditable:

- `version: latest` reference → records `status.currentVersion` from the fetched group
  (e.g. `"v3"`) at the time of this deployment
- `version: "v2"` reference → records `"v2"`

On the next re-deployment with `version: latest`, the controller fetches fresh and may record
a newer version — this is the intended behaviour.

### 3.4 New RBAC markers

Add to `controller.go`:

```go
// +kubebuilder:rbac:groups=openchoreo.dev,resources=configurationgroups,verbs=get;list;watch
// +kubebuilder:rbac:groups=openchoreo.dev,resources=configurationgroupversions,verbs=get;list;watch
```

`watch` is required for the controller-runtime informer cache even though no reactive
re-queuing is set up.

---

## Phase 4: Pipeline context (`internal/pipeline/component/context`)

_To be planned after Phase 3 is approved._

---

## Phase 5: REST API (`internal/openchoreo-api`)

### 5.1 OpenAPI spec (`openapi/openchoreo-api.yaml`)

Add new paths. Project-scoped and namespace-wide groups use separate route prefixes
since their scope is structurally different.

**Project-scoped groups:**
```
GET    /namespaces/{namespace}/projects/{project}/configuration-groups
POST   /namespaces/{namespace}/projects/{project}/configuration-groups
GET    /namespaces/{namespace}/projects/{project}/configuration-groups/{name}
PUT    /namespaces/{namespace}/projects/{project}/configuration-groups/{name}
DELETE /namespaces/{namespace}/projects/{project}/configuration-groups/{name}
GET    /namespaces/{namespace}/projects/{project}/configuration-groups/{name}/versions
GET    /namespaces/{namespace}/projects/{project}/configuration-groups/{name}/versions/{version}
```

**Namespace-wide groups** (no project prefix):
```
GET    /namespaces/{namespace}/configuration-groups
POST   /namespaces/{namespace}/configuration-groups
GET    /namespaces/{namespace}/configuration-groups/{name}
PUT    /namespaces/{namespace}/configuration-groups/{name}
DELETE /namespaces/{namespace}/configuration-groups/{name}
GET    /namespaces/{namespace}/configuration-groups/{name}/versions
GET    /namespaces/{namespace}/configuration-groups/{name}/versions/{version}
```

Run `make code.gen` after updating the spec to regenerate `api/gen/`.

### 5.2 New service packages

**`services/configurationgroup/`**

Full CRUD for `ConfigurationGroup` CRs:

| Operation | Notes |
|---|---|
| `Create` | Writes `ConfigurationGroup` CR with `spec.owner.projectName` set (project-scoped) or absent (namespace-wide) |
| `Get` | Fetches by name; returns spec + `status.currentVersion` |
| `List` | Lists by namespace; optionally filtered by `spec.owner.projectName` |
| `Update` | Updates `spec` — controller detects the change and auto-creates a new version |
| `Delete` | Deletes the CR; Kubernetes GC cascades to all child `ConfigurationGroupVersion` CRs via owner references |

**`services/configurationgroupversion/`**

Read-only — versions are controller-managed and never created or modified via the API:

| Operation | Notes |
|---|---|
| `Get` | Fetches a specific `ConfigurationGroupVersion` by name (e.g. `kafka-config-v2`) |
| `List` | Lists all versions for a parent group using label selector `configurationgroup.openchoreo.dev/parent=<name>`, ordered by version number |

### 5.3 New HTTP handler packages

- `api/handlers/configurationgroup/` — wires `services/configurationgroup` to the generated OpenAPI server interfaces
- `api/handlers/configurationgroupversion/` — wires `services/configurationgroupversion` to the generated interfaces

Both follow the existing handler pattern in the codebase.

### 5.4 MCP handlers (`mcphandlers/`)

New tools following the existing pattern:

| Tool | Description |
|---|---|
| `list_configuration_groups` | List groups in a namespace, optionally filtered by project |
| `get_configuration_group` | Get a group by name |
| `create_configuration_group` | Create a new group |
| `update_configuration_group` | Update an existing group spec |
| `delete_configuration_group` | Delete a group |
| `list_configuration_group_versions` | List all versions for a group, ordered by version number |
| `get_configuration_group_version` | Get a specific version snapshot |

---

## Phase 6: CLI (`cmd/occ`)

New package `internal/occ/cmd/configurationgroup/` following the existing command pattern
(`cmd.go`, `params.go`, `configurationgroup.go`).

### 6.1 Sub-commands

| Command | Description |
|---|---|
| `occ configurationgroup list` | List groups in a namespace. `--project` filters to project-scoped groups; omitting it lists namespace-wide groups |
| `occ configurationgroup get <name>` | Get a group; shows spec and `status.currentVersion` |
| `occ configurationgroup create -f <file>` | Create a group from a YAML file |
| `occ configurationgroup update <name> -f <file>` | Update a group spec from a YAML file; controller auto-creates a new version |
| `occ configurationgroup delete <name>` | Delete a group; cascades to all child versions via owner references |
| `occ configurationgroup versions <name>` | List all `ConfigurationGroupVersion` CRs for a group, ordered by version number |
| `occ configurationgroup get-version <name> <version>` | Get a specific version snapshot (e.g. `kafka-config v2`) |

### 6.2 Flags

| Flag | Commands | Notes |
|---|---|---|
| `--namespace` / `-n` | All | Required |
| `--project` / `-p` | `list`, `get`, `create`, `update`, `delete`, `versions`, `get-version` | Scopes to a project; omit for namespace-wide groups |

### 6.3 Registration

- Add `NewConfigurationGroupCmd` to the root command in `internal/occ/root/root.go`
- Parent command aliases: `cg`, `configurationgroups`

---

## Cross-cutting

| Task | Notes |
|---|---|
| `make code.gen` | Regenerate CRDs, deepcopy, Helm manifests after every type change |
| Register new CRDs in `cmd/manager/main.go` | `ConfigurationGroup` and `ConfigurationGroupVersion` |
| Register types in `SchemeBuilder` in `groupversion_info.go` | Both new types |
