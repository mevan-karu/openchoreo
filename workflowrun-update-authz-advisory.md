# Authorization bypass on WorkflowRun update allows cross-project takeover of workflow runs

Severity: High
CWE-639 (Authorization Bypass Through User-Controlled Key), CWE-862 (Missing Authorization)

## Summary

The OpenChoreo API server authorized `PUT .../workflowruns/{runName}` against ownership metadata taken from the request body, while the object it actually mutated was selected by the URL path. Nothing forced the two to agree.

A `WorkflowRun`'s membership in a project and component is carried entirely by its `openchoreo.dev/project` and `openchoreo.dev/component` labels. Because the `workflowrun:update` check read those labels from the submitted object rather than the stored one, a caller holding `workflowrun:update` on one component could name a run belonging to a different project or component in the URL, supply labels for the component they legitimately control, pass the check, and have the unrelated run rewritten with their own ownership labels.

The namespace is taken from the URL and feeds both the authorization check and the Kubernetes read, so it cannot diverge. This is cross-project and cross-component escalation inside a single namespace, not a cross-tenant issue.

## Impact

Rewriting the ownership labels transfers authority over the run. Every other WorkflowRun check (view, logs, events, status, delete, list) derives its scope from the stored labels, so once the write lands the policy engine legitimately grants the attacker access to an object they never had rights to. They can then:

- Read the victim's build logs. These stream raw pod logs from the workflow plane and routinely carry source paths, registry destinations, and credentials a build step echoed by accident.
- Read the run's Kubernetes events.
- Delete the run.

So the primary impact is confidentiality and availability, not only integrity. On the integrity side, `spec.workflow.parameters` can be overwritten on a pending run, and manipulating the two ownership labels can either wedge the run in a permanent `ComponentValidationFailed` condition or downgrade it to a standalone run that skips component validation.

Exploitation requires an authenticated caller with an `AuthzRoleBinding` or `ClusterAuthzRoleBinding` granting `workflowrun:update` at project or component scope in the victim's namespace, plus knowledge of the target run's name. An anonymous caller cannot reach this.

## Patches

The authorization check now loads the stored WorkflowRun and derives every ownership attribute from it instead of from the request body. Separately, the `openchoreo.dev/project` and `openchoreo.dev/component` labels are now immutable on update, so an authorized caller cannot hand their own run to another project either.

| Line | Affected | Fixed in |
|---|---|---|
| 1.0.x | 1.0.0 through 1.0.5 | 1.0.6 |
| 1.1.x | 1.1.0 through 1.1.6 | 1.1.7 |
| 1.2.x | 1.2.0 through 1.2.4 | 1.2.5 |

Upgrade to the patched release on your line. No configuration change is needed after upgrading.

## Workarounds

If you cannot upgrade immediately, restrict who holds `workflowrun:update`.

Audit your `AuthzRole` and `ClusterAuthzRole` definitions for that action and remove it from any role granted to principals you would not trust with write access to every workflow run in the namespace. Among the roles shipped with OpenChoreo, only the system role `workload-publisher` lists `workflowrun:update` explicitly, and `admin` holds it through its `*` wildcard. Custom roles in your own values file may also include it, so check those.

Treat any credential bound to `workload-publisher` as privileged. CI systems are the usual holder, and a leaked CI token is enough to run this.

## Credits

Reported by @Tike00.
