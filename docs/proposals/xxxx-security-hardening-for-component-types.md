# No Pod and Container Security Context Support in Default Component Types

Based on an offline discussion with @manjulaRathnayaka and @yashodgayashan, we noticed that the default component types (`service`, `worker`, `web-application`, `scheduled-task`) produce workloads with no pod or container security context configured. This means containers run as root, hold full kernel capabilities, and have the default Kubernetes ServiceAccount token auto-mounted, none of which is appropriate for production deployments.

The challenge is that enforcing strict security defaults introduces friction for users who are just trying things out, as many common images run as root or write to the container filesystem. Shipping with no controls at all, however, leaves production deployments exposed. We discussed two possible approaches to address this.

---

## Approach A: Security block in `environmentConfigs` or `parameters`

Add a `security` block containing a field per control (e.g. `runAsNonRoot`, `dropAllCapabilities`, `readOnlyRootFilesystem`). The component type template emits the corresponding `securityContext` fields based on the configured values. All fields default to permissive so the quickstart experience is unchanged.

Where the block lives determines the level at which the security posture is controlled:

- **`environmentConfigs`** - security is enforced at the application's environment level, meaning the posture can vary per environment. For example, a relaxed configuration in dev and a hardened one in production.
- **`parameters`** - security is enforced at the application level, meaning the posture is set once for the component and remains consistent across all environments it is deployed to.

The two are not mutually exclusive and can be used together to cover both levels.

---

## Approach B: `pod-security` ClusterTrait

A `pod-security` ClusterTrait is shipped as part of the platform with two capabilities:

- **`creates`** - provisions a dedicated `ServiceAccount` per component.
- **`patches`** - applies JSON Patch operations to the Deployment to inject `securityContext` fields at both the pod and container level.

The default component types remain unchanged. PEs embed this trait into their component types when they want to enforce security hardening. The embedded trait bindings are CEL expressions evaluated against the component context, so the PE can wire the trait's parameters to the component type's own `environmentConfigs` or `parameters` schema (e.g. `${environmentConfigs.security.runAsNonRoot}`). This is already supported: the rendering pipeline resolves embedded trait bindings against the component context before building the trait context, and trait patching of existing resources is also already implemented via `spec.patches` with JSON Patch operations.

Keeping the security logic in a dedicated ClusterTrait means it can be updated independently without modifying component type templates, and the same trait can be reused across any component type.

---

## Value addition applicable to both approaches

Regardless of which approach is taken, a `preset` shorthand aligned to the [Kubernetes Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/) (e.g. `preset: baseline` or `preset: restricted`) can be supported alongside the individual field controls. This lets teams adopt a well-known security level with minimal configuration while retaining the ability to override specific fields. For example, a team could apply `preset: restricted` but temporarily disable `readOnlyRootFilesystem` for images that are not yet ready.

---

Both approaches cover the same set of pod-level and container-level security controls. The key difference is where the configuration lives. Approach A adds the security block directly into the component type, which impacts the readability of the component type definition as it grows with additional fields and schema. Approach B extracts that concern into a dedicated ClusterTrait, keeping the component type clean while the PE wires the two together. 

Personally I am more aligned with the trait approach, mainly because it gives PEs the same building block to use in their own custom component types as well, making it a more natural fit for the extensibility model OpenChoreo is built around.

We would love to hear from the community on which direction feels right, or if there are other approaches worth considering.

---

**@sameerajayasoma** Thanks for raising this. I looked into this and what it does is create an isolated UID namespace per pod, so a process running as root inside the container actually maps to a high unprivileged UID on the host rather than actual UID 0. The security context fields limit what a process can do inside the container, while user namespaces limit the blast radius if the container boundary itself is breached.

This works along with the approaches we discussed in the above proposal. In terms of how this impacts the proposal, `hostUsers: false` would naturally be another knob in the security block, sitting alongside the existing controls in either approach. However this has some non-trivial infrastructure requirements including Linux kernel 6.3 or later, containerd v2.0+ or CRI-O v1.25+, and filesystems that support idmap mounts.
