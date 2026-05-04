Following up on the proposal and the meeting discussion. We evaluated the semantic traits and template reference approaches from the original discussion, and additionally evaluated the base component type (WorkloadType) approach proposed by @sameerajayasoma [above](#discussioncomment-16763403). Below is a summary of all three.

As discussed in the meeting, the core issue is that a component type today inlines both OC-internal constructs (config/secret injection, dependency wiring) and PE-owned policy (routing, workflows, validations) in a single definition. PEs creating a custom type must copy and understand OC internals they hardly need to touch, and become responsible for migrating their copies whenever OC evolves those internals. The approaches below aim to separate these two concerns.

---

## Approach 1: Template References

### How it works

Introduce a `ClusterResourceTemplate` CRD that holds a single resource definition. Component types reference shared templates by name via a `templateRef` field in their `resources` list. Unique resources remain inline.

```yaml
# Shared template — defined once, referenced by all CTs that need it
apiVersion: openchoreo.dev/v1alpha1
kind: ClusterResourceTemplate
metadata:
  name: env-config
spec:
  id: env-config
  forEach: ${configurations.toConfigEnvsByContainer()}
  var: envConfig
  template:
    apiVersion: v1
    kind: ConfigMap
    metadata:
      name: ${envConfig.resourceName}
      namespace: ${metadata.namespace}
    data: |
      ${envConfig.envs.transformMapEntry(index, env, {env.name: env.value})}
```

```yaml
# Component type — mixes templateRefs with inline routing resources
kind: ClusterComponentType
metadata:
  name: service
spec:
  workloadType: deployment
  allowedWorkflows: [...]
  resources:
    - templateRef:
        kind: ClusterResourceTemplate
        name: standard-deployment
    - templateRef:
        kind: ClusterResourceTemplate
        name: env-config
    - templateRef:
        kind: ClusterResourceTemplate
        name: file-config
    - templateRef:
        kind: ClusterResourceTemplate
        name: secret-env-external
    - templateRef:
        kind: ClusterResourceTemplate
        name: secret-file-external
    - id: service
      includeWhen: ${size(workload.endpoints) > 0}
      ...
    - id: httproute-external
      ...
```

### What it solves
- Shared resource bodies live in one place — a fix to the `env-config` template or the deployment template is made once and all referencing CTs pick it up immediately.
- The CT file remains readable: the full resource list is visible with inline and referenced entries mixed.

### Concerns
- **No invariant boundary** — The CT still owns the full resource list (by reference or inline). There is no structural distinction between OC-managed templates and PE-customizable ones. A PE can omit a required `templateRef` and nothing prevents it.
- **Schema contract implicit across files** — Each `ClusterResourceTemplate` consumes `parameters` and `environmentConfigs` fields without declaring them as a formal interface. When a PE creates a custom CT referencing these templates, the full set of fields already in use is only discoverable by opening each referenced template. Extending `environmentConfigs` or `parameters` with new fields risks colliding with names the templates already depend on, and such conflicts may only surface as render-time failures.

---

## Approach 2: Semantic Traits

### How it works

Introduce platform-managed `ClusterTrait` resources that group logically related resources and JSON Patch operations. Component types embed these traits via a `traits:` field. Traits both create shared resources and patch into the CT's primary workload via RFC 6902 operations.

Three platform traits are introduced:
- **`configuration`** — creates the four config/secret resource blocks and patches the workload to inject `envFrom`, `volumeMounts`, and `volumes`.
- **`service-routing`** — injects dependency connection env vars and creates path-prefix `HTTPRoute` resources with gateway validation rules.
- **`webapp-routing`** — same dependency injection, but creates subdomain-based `HTTPRoute` resources via `oc_dns_label`.

```yaml
# Trait — creates resources and patches into the CT's workload template
kind: ClusterTrait
metadata:
  name: configuration
spec:
  creates:
    - id: env-config
      forEach: ${configurations.toConfigEnvsByContainer()}
      ...
    - id: secret-env-external
      ...
  patches:
    - target:
        kind: Deployment
      patch:
        - op: add
          path: /spec/template/spec/containers/0/envFrom/-
          value: ${configurations.toContainerEnvFrom()}
        - op: add
          path: /spec/template/spec/containers/0/volumeMounts/-
          value: ${configurations.toContainerVolumeMounts()}
        - op: add
          path: /spec/template/spec/volumes/-
          value: ${configurations.toVolumes()}
```

```yaml
# Component type — embeds traits, declares only its primary workload
kind: ClusterComponentType
metadata:
  name: service
spec:
  traits:
    - name: configuration
    - name: service-routing
  resources:
    - id: deployment
      template:
        apiVersion: apps/v1
        kind: Deployment
        # no env/envFrom/volumeMounts — traits patch these in
        ...
    - id: service
      includeWhen: ${size(workload.endpoints) > 0}
      ...
```

### What it solves
- Config/secret blocks are removed from CT definitions entirely — the `configuration` trait owns them.
- Routing resources and gateway validations are centralised in routing traits.
- The patch engine (`ensureParentExists`) creates missing container arrays automatically — no placeholder fields needed in workload templates.

### Concerns
- **Deployment still duplicated** — `service`, `worker`, and `web-application` each declare their own `Deployment` resource inline. Traits only own the peripheral resources; the primary workload is still repeated per CT.
- **No OC invariant boundary** — A PE creating a custom CT can omit the `configuration` trait entirely, producing a CT that doesn't integrate with OC's config/secret management. There is no structural enforcement.

---

## Approach 3: Base Component Types

### How it works

Introduce a `ClusterBaseComponentType` CRD that encodes OC's integration contract for each workload kind. Every component type references a base via `baseType:`. The base is OC-owned and immutable to PEs. It contains everything OC requires from any workload of that kind. The CT layer sits on top and declares only policy: which workflows are allowed, which traits are allowed, routing resources, and semantic validations.

```yaml
# Base type — OC-owned, immutable, encodes the full integration contract
kind: ClusterBaseComponentType
metadata:
  name: base-deployment
spec:
  workloadType: deployment
  environmentConfigs:
    openAPIV3Schema:
      properties:
        replicas: { type: integer, default: 1 }
        resources: ...
        imagePullPolicy: ...
  resources:
    - id: service
      includeWhen: ${size(workload.endpoints) > 0}
      template: ...
    - id: deployment
      template:
        ...
        env: ${dependencies.toContainerEnvs()}
        envFrom: ${configurations.toContainerEnvFrom()}
        volumeMounts: ${configurations.toContainerVolumeMounts()}
        volumes: ${configurations.toVolumes()}
    - id: env-config
      ...
    - id: file-config
      ...
    - id: secret-env-external
      ...
    - id: secret-file-external
      ...
```

```yaml
# Default component type — references base, declares only policy and routing
kind: ClusterComponentType
metadata:
  name: service
spec:
  baseType:
    kind: ClusterBaseComponentType
    name: base-deployment

  allowedWorkflows:
    - kind: ClusterWorkflow
      name: paketo-buildpacks-builder
    ...

  allowedTraits:
    - kind: ClusterTrait
      name: observability-alert-rule

  validations:
    - rule: "${size(workload.endpoints) > 0}"
      message: "Service components must have at least one endpoint."

  resources:
    - id: httproute-external
      ...
    - id: httproute-internal
      ...
```

A PE creating a custom type only declares the delta. The `patches:` block applies surgical RFC 6902 operations to base resources without re-declaring the full template. The `parameters` and `environmentConfigs` schemas extend the base additively.

```yaml
# PE-authored custom type — only the delta from the base
kind: ClusterComponentType
metadata:
  name: gpu-worker
spec:
  baseType:
    kind: ClusterBaseComponentType
    name: base-deployment

  allowedWorkflows: [...]

  parameters:
    openAPIV3Schema:
      properties:
        gpuType: { type: string, default: nvidia-tesla-t4 }

  environmentConfigs:
    openAPIV3Schema:
      properties:
        gpuCount: { type: integer, default: 1, minimum: 1, maximum: 8 }

  patches:
    - target: deployment
      patch:
        - op: add
          path: /spec/template/spec/nodeSelector
          value:
            cloud.google.com/gke-accelerator: ${parameters.gpuType}
        - op: add
          path: /spec/template/spec/containers/0/resources/limits/nvidia.com~1gpu
          value: "${environmentConfigs.gpuCount}"

  validations:
    - rule: "${size(workload.endpoints) == 0}"
      message: "GPU worker components must not expose endpoints."
```

### What it solves
- The `Deployment`, config/secret blocks, and K8s `Service` all live in the base — eliminated from every CT definition.
- Clear OC invariant boundary: the base is the contract, the CT is the policy. A PE cannot accidentally produce a CT that bypasses OC's integration.
- PEs don't need to understand OC-internal CEL expressions; they inherit everything correct from the base.
- `patches:` enables surgical customization without full template re-declaration.
- Schema extension lets CTs add developer-facing parameters and environment config knobs on top of base defaults.

### Concerns
- **OC connection label contract cannot be enforced by the base** — The connections feature resolves dependency URLs by reading labels from rendered HTTPRoute resources. HTTPRoutes live at the CT level, so the base cannot guarantee the required OC labels (`openchoreo.dev/endpoint-name`, `openchoreo.dev/endpoint-visibility`) are present. A custom routing CT that omits them will silently break connection URL resolution.
- **HTTPRoute duplication between `service` and `web-application`** — The two types differ only in hostname construction (path-prefix vs subdomain). The K8s `Service` is shared via the base; the two HTTPRoute template sets remain per CT.
- **Breaking change for existing component types** — All existing CT definitions must be updated to reference a base type. Fields now owned by the base (`workloadType`, config/secret resources, the workload template) become redundant. A migration path is needed for CTs deployed in production clusters.

---

## Common Concerns

These apply to all three approaches:

- **Update propagation requires deliberate action** — Whether a shared resource changes in a `ClusterResourceTemplate`, a `ClusterTrait`, or a `ClusterBaseComponentType`, updated renders only reach components automatically if `autoDeploy: true` is set. Components with manual promotion require an explicit configure-and-deploy. The value of centralising a fix is a source-of-truth benefit, not automatic propagation.
- **Readability trade-off** — Any indirection (templateRef, embedded trait, baseType) means the full rendered output cannot be understood from a single file.

---

## Comparison

| | Template references | Semantic traits | Base component types |
|---|---|---|---|
| Config/secret block duplication | Eliminated (shared templates) | Eliminated (configuration trait) | Eliminated (in base) |
| Deployment duplication (service/worker/webapp) | Remains | Remains | Eliminated (in base) |
| K8s Service duplication | Remains | Eliminated (trait creates) | Eliminated (in base) |
| HTTPRoute duplication (service vs web-app) | Remains | Eliminated (separate routing traits) | Remains |
| OC invariant boundary | None | None | Enforced (base is the contract) |
| Custom type authoring burden | Medium — must know template names | Medium — must know trait + patch model | Low — reference base, declare delta |
| Surgical resource customization | No | No (full resource inline) | Yes (`patches:` block) |
| Schema extension (parameters / environmentConfigs) | No | No | Yes — CT extends base schema |
| New CRD required | Yes (`ClusterResourceTemplate`) | No | Yes (`ClusterBaseComponentType`) |
| Breaking change for existing CTs | No | No | Yes |
