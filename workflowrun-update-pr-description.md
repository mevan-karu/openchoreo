## Purpose
> Briefly describe the problem or need driving this PR and how it resolves the issue. Include links to related issues if applicable.

`UpdateWorkflowRun` authorized requests using the project/component labels and workflow
reference from the request body, instead of the values on the stored `WorkflowRun`. This
meant authorization could be evaluated against a hierarchy the caller doesn't actually
own. It also allowed the project/component ownership labels to be changed freely on
update, with no immutability guard.

## Approach
> Summarize the solution and implementation details.

- `service_authz.go`: `UpdateWorkflowRun` now fetches the existing run first and builds
  the authz hierarchy and workflow attribute from its stored labels/spec, matching the
  existing pattern in `workload`, `releasebinding`, `projectreleasebinding`, and
  `resourcereleasebinding`. It also guards against a nil request, matching the internal
  service's existing check, before that fetch dereferences it.
- `service.go`: the project/component labels are now immutable on update, consistent with
  `spec.workflow.kind`/`name`. A request that omits these labels reuses the stored values
  (backward compatible for callers that don't send them), while a request that explicitly
  supplies a different value — or that adds ownership labels to a previously standalone
  run — is rejected.
- `service.go`: the referenced workflow (`spec.workflow.kind`/`name`) is now also checked
  for immutability at the service layer, mirroring the CRD's own CEL rule and making that
  invariant explicit and testable here rather than relying on it alone.

## Related Issues
> Include any related issues that are resolved by this PR.

N/A

## Checklist
- [x] Tests added or updated (unit, integration, etc.)
- [ ] Samples updated (if applicable)
- [ ] Added `backport/<release-branch>` label if this should be backported (e.g., `backport/release-v1.0`)
- [ ] This PR includes AI-generated code or content

## Remarks
> Add any additional context, known issues, or TODOs related to this PR.

No API-type or OpenAPI-spec changes; `make code.gen` is a no-op for this change.
