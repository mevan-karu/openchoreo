# Response to reporter

Thanks for this. We've walked the code paths you cited and the finding is valid. The authorization
decision on `PUT .../workflowruns/{runName}` is made against ownership metadata taken from the request
body, while the object that actually gets mutated is selected by the URL. Nothing currently stops
those two from disagreeing.

Your description of the mechanism is accurate down to the line numbers, so we won't repeat it back. A
few scope points where our assessment differs:

**Namespace is not caller-controlled.** `namespaceName` comes from the URL path and feeds both the
authz hierarchy and the Kubernetes Get, so the two cannot diverge. A caller already needs
`workflowrun:update` in the target namespace. Since the namespace is our tenant boundary, this is
cross-project and cross-component escalation inside a single namespace rather than a cross-tenant
issue.

**The workflow reference cannot be repointed.** `spec.workflow.name` and `spec.workflow.kind` both
carry `self == oldSelf` CEL validation, so the API server rejects any update that changes them. The
genuinely mutable surface is labels, annotations, `spec.workflow.parameters` and `ttlAfterCompletion`.
The labels are the part that matters here.

**The role file cited is the wrong one.** `config/rbac/workflowrun_editor_role.yaml` governs direct
kubectl access to the CRD and has no bearing on our own policy engine. The real precondition is an
AuthzRoleBinding scoped to a project or component.

One dimension is worse than your report describes. You framed the impact as integrity manipulation,
but every other WorkflowRun check (Get, Logs, Events, Status, Delete, List) derives its authorization
scope from the stored labels. Rewriting those labels therefore transfers authority over the object
permanently. Once the write lands, the caller legitimately reads the victim's build logs and events
and can delete the run. We've reweighted this as a confidentiality and availability problem, not only
an integrity one.

We're treating the fix as two separate defects. The authorization check will read the stored object
instead of the request body, matching what the `workload` and `releasebinding` services already do.
Separately, the project and component labels become immutable on update, because the first fix on its
own only reverses which direction the bypass runs in.

We'll keep you posted as the patch lands, and we're glad to credit you in the advisory if you'd like.
Thanks again for a clear and well-evidenced report. The file and line references saved us real time.

One more thing worth noting: hitting a specific victim run isn't automatic. The URL's `runName` picks the object, not the labels in the body. So the attacker needs `workflowrun:update` somewhere, and needs to already know the target run's name.

That's not an equal bar everywhere. `TriggerWorkflow` names carry a random 8-hex suffix, not guessable over HTTP. Caller-chosen names are weaker, since they follow whatever convention a team uses, and can also leak via the UI, CI output, the observer API, or event streams, none of which we've audited.

Worth flagging: caller-chosen names aren't just theoretical. `occ apply` creates WorkflowRuns through this same endpoint using the name from the user's manifest (`internal/occ/cmd/apply/registry.go:888-899`), so a predictable naming scheme there is a real instance of that weaker case.
