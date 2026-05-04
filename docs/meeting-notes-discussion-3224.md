# Meeting Notes — Component Type Extensibility

**Discussion:** https://github.com/openchoreo/openchoreo/discussions/3224

---

## Discussion Points

- Acknowledged the duplication problem in the proposal, but noted it is not a major concern on its own. The real concern is that component types embed OpenChoreo-specific resource configurations (e.g. ExternalSecrets, connection wiring) that PEs are expected to copy and maintain without necessarily understanding what those pieces do.

- A secondary concern is long-term maintainability: a PE who owns a full copy of a component type is implicitly responsible for tracking every upstream change to the original (bug fixes, API version bumps, label convention changes). There is currently no mechanism to surface or reconcile that drift.

- Discussed whether **embedded traits** could address the inheritance need without introducing an `extends` field or a new CRD. Two sub-approaches were explored:

  - **Resource-level traits:** Expose individual rendered resources (httproutes, services, configs, secrets) as composable traits so a PE can assemble a component type by picking what they need rather than defining everything inline.
    - Concern: several resources, like the base `Deployment`, don't map naturally to the trait model and would feel forced.

  - **Semantic traits that patch across resources:** Define higher-level traits (e.g. an `endpoint` trait covering httproute + service + ingress concerns, a `secret-injection` trait covering both env and file mounts) that operate on multiple underlying resources as a unit.
    - This sidesteps the awkward per-resource granularity but needs careful design to avoid tight coupling between trait internals and component type structure. Decided to evaluate feasibility further before committing.

- Discussed adding a **`templateRef` attribute** on individual resources within a component type, letting the resource body live in a separate file rather than inline.
  - A PE could point shared resource blocks (configs, secrets) at common templates and only inline what is unique to their type.
  - Needs further evaluation to catch edge cases, particularly around how template changes propagate to component types that reference them.

- A concern raised across all of the above approaches: splitting a component type's definition across multiple files or traits makes things noticeably harder to follow. Anyone writing a new trait, or trying to understand what a component type does end to end, would need to trace through multiple files, which hurts both the authoring and debugging experience.

---

## Action Items

- Further evaluate the semantic traits approach and the `templateRef` approach to understand their usability, surface any issues or concerns, and identify any blockers before the next meeting.
- Schedule a follow-up meeting to review findings from both evaluations and decide on the way forward.
