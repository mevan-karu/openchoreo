# Security Report Review — WorkflowRun Update Authorization (BOLA / CWE-639)

**Verdict: VALID.** The core defect is real and confirmed in source. Severity **HIGH** stands, but the
report's characterisation of both the impact and the blast radius is wrong in several places, and one
of its claims (that the same pattern affects other resources) does not generalise. Corrections below.

---

## 1. The defect is real

Confirmed by reading the three cited files.

The authorization subject and the mutation target are chosen independently:

| | source |
|---|---|
| mutation target | `request.RunName` (URL) — `api/handlers/workflows.go:353` |
| authz hierarchy `project` / `component` | `wfRun.Labels` (request body) — `services/workflowrun/service_authz.go:95` → `:60-71` |
| authz ABAC attr `resource.workflow` | `wfRun.Spec.Workflow.{Kind,Name}` (request body) — `service_authz.go:98` |
| object actually mutated | `Get` by `wfRun.Name` + URL namespace, then `existing.Labels/Annotations/Spec = wfRun.*` — `services/workflowrun/service.go:132-145` |

`UpdateWorkflowRun` in `service.go` has **no** owner-consistency guard between the submitted body and
the stored object, unlike its siblings (see §4). The wrapper never reads the stored object at all.

**Single entry point.** `WorkflowRunService` is constructed only via `NewServiceWithAuthz`
(`services/handlerservices/services.go:133`). There is no MCP tool for workflow-run update
(`mcphandlers/components.go` exposes create/list/get/status/logs/events only) and the autobuild
webhook path reaches `TriggerWorkflow`, not `UpdateWorkflowRun`. So the fix has exactly one place to
live.

---

## 2. What the report got wrong

### 2.1 No cross-tenant / cross-namespace escalation

The report says the caller can "submit a request whose URL names a different run/**namespace**". That
is false. `namespaceName` is taken from the URL path and is used for *both* the authz hierarchy
(`service_authz.go:95` passes it straight through) and the Kubernetes `Get`
(`service.go:134`). They cannot diverge. The caller must already hold `workflowrun:update` **in the
target namespace**.

Since the namespace is OpenChoreo's tenant boundary, this is **not** cross-tenant. It is
**cross-project / cross-component escalation inside one namespace** — a holder of a component-scoped
`AuthzRoleBinding` reaching objects belonging to other projects and components in the same namespace.
Still a genuine authorization bypass; just a smaller radius than claimed.

### 2.2 `spec.workflow.name` and `.kind` are CEL-immutable — the spec cannot be repointed

`api/v1alpha1/workflowrun_types.go:36,43` carry
`+kubebuilder:validation:XValidation:rule="self == oldSelf"` on both `kind` and `name`. The API server
rejects any update that changes them. So the attacker **cannot** repoint a victim's run at a different
workflow. The genuinely mutable surface is:

- `metadata.labels` (**the authorization-bearing field** — no immutability anywhere)
- `metadata.annotations`
- `spec.workflow.parameters`
- `spec.ttlAfterCompletion`

This constrains the attack but does not defeat it, because the attacker must *echo back* the stored
workflow name — see §2.4.

### 2.3 Wrong precondition cited

The report points at `config/rbac/workflowrun_editor_role.yaml`. That is a **Kubernetes RBAC**
manifest governing direct `kubectl` access to the CRD; it has nothing to do with the platform's own
PDP. The real precondition is an `AuthzRoleBinding` / `ClusterAuthzRoleBinding` whose `TargetScope`
is a project or component and which grants the `workflowrun:update` action.

### 2.4 The `spec.workflow.name` echo requirement — and why it doesn't save you

Because of §2.2, the PUT only succeeds if the body carries the victim run's exact stored workflow
name, which the attacker cannot read (`GetWorkflowRun` authorizes on *stored* labels and correctly
denies). Two reasons this is a weak barrier:

- **Shared cluster workflows.** The default install ships four cluster-scoped workflows that every
  component draws from — `paketo-buildpacks-builder`, `gcp-buildpacks-builder`,
  `ballerina-buildpack-builder`, `dockerfile-builder` (`samples/getting-started/all.yaml`), and
  `ClusterComponentType.allowedWorkflows` lists exactly these. A guess succeeds within four tries in
  a stock deployment.
- **The endpoint is an oracle** (see §3.3).

Severity **HIGH** is therefore justified, with the preconditions stated honestly: *component-scoped
`workflowrun:update` in the victim's namespace, knowledge or a guess of the workflow name, and
knowledge of the target run's name (see §2.5).*

### 2.5 Name uniqueness does not close the hole — and the bypass is directional

A namespace cannot hold two projects or two components with the same name, so the label pair in the
request body unambiguously denotes exactly one real component. That is worth being precise about,
because it is what makes the attack work rather than what prevents it.

**Uniqueness makes the *claimed* scope real; nothing makes the *target* belong to it.** A
`WorkflowRun`'s membership in a project/component is carried **solely** by its
`openchoreo.dev/project` and `openchoreo.dev/component` labels — confirmed by grep: those two labels
are the only ownership signal anywhere in `services/workflowrun/`, used for authz
(`service_authz.go:61-62`), for list filtering (`service.go:176,181`), and set at creation
(`service.go:927-928`). There is no ownerReference-based check, and no binding between a run's *name*
and its owner: `CreateWorkflowRun` performs no name validation, and while `TriggerWorkflow` generates
`<component>-run-<8 hex>`, that is a convention the authz layer never consults.

So the vulnerable link is **run-name → labels**, not name → name. Uniqueness of component names has
no bearing on whether the run selected by the URL carries the labels supplied in the body. Uniqueness
guarantees "component A" is a well-defined address; it does not guarantee the object being mutated
lives at that address.

**The bypass runs in one direction only.** The check evaluates the *destination* labels (from the
body), which yields an asymmetry worth stating explicitly:

| direction | stored labels | body labels | authz evaluated against | outcome |
|---|---|---|---|---|
| **Theft** — pull victim's run into your scope | `project-b`/`component-b` | `project-a`/`component-a` | `component-a` (attacker holds it) | **allowed — bypass** |
| **Planting** — push your run into victim's scope | `project-a`/`component-a` | `project-b`/`component-b` | `component-b` (attacker lacks it) | denied |

So yes: it bypasses authz between components and projects, in the theft direction — which is the
damaging one, since every read and delete check afterwards consults the now-attacker-owned stored
labels (§3).

**The practical gate is target-name discovery, not uniqueness.** To steal a run the attacker must
name it in the URL, and `ListWorkflowRuns` filters per item on stored labels
(`service_authz.go:117` via `FilteredList`, `pagination.go:234`), so the API will not enumerate other
components' runs. Against `TriggerWorkflow`-generated names the 4-byte random suffix (2^32) is not
brute-forceable over HTTP. That materially lowers *practical* exploitability, with three caveats:

- Runs created via `POST` carry caller-chosen names, so any predictable team convention is guessable.
- Because authz currently runs *before* the `Get`, the endpoint distinguishes 404 (no such run) from
  422/200 (exists) — a namespace-wide run-name existence oracle.
- Run names leak through channels outside this endpoint (UI, CI output, observer API, event streams).
  I have **not** audited those leak channels; that audit is what would settle exploitability, and it
  is worth doing before deciding how urgently to ship.

---

## 3. What the impact actually is — the report understates it

The report calls this "integrity manipulation". The far more serious consequence is that
**rewriting the labels permanently transfers authority over the object.**

Every other WorkflowRun authorization check derives its hierarchy from the **stored** labels:

- `GetWorkflowRun` — `service_authz.go:132`
- `GetWorkflowRunLogs` — `:169`
- `GetWorkflowRunEvents` — `:185`
- `GetWorkflowRunStatus` — `:201`
- `DeleteWorkflowRun` — `:148`
- `ListWorkflowRuns` — `:117`

### 3.1 Confidentiality (primary — not in the report)

Attacker rewrites the victim run's `openchoreo.dev/project` + `openchoreo.dev/component` labels to
their own component. From that moment the PDP legitimately grants them view rights. They then read
the victim's **build logs** (`GetWorkflowRunLogs` streams raw pod logs from the workflow plane) and
Kubernetes **events**. Build logs routinely carry source paths, registry destinations, and
accidentally-echoed credentials. This is a cross-project confidentiality breach, not metadata tampering.

### 3.2 Availability (also not in the report)

Post-relabel, `DeleteWorkflowRun` also passes → the attacker can delete another project's runs.
Separately, setting exactly one of the two labels drives the controller into a permanent
`ComponentValidationFailed` condition (`internal/controller/workflowrun/controller_validate.go:51-55`),
and clearing both silently downgrades a component run to a "standalone" run that skips
component/ComponentType validation entirely (`:46-48`).

### 3.3 Information disclosure via oracle

The 422-vs-200 response on the PUT lets an unauthorized caller enumerate a run's `spec.workflow.name`
without being able to `GET` it. This is disclosure in its own right, and it only disappears if the
fix **denies before touching the object** — which the proposed fix does.

### 3.4 Integrity (what the report described) — the weakest of the four

`spec.workflow.parameters` can be overwritten. Practically limited: the controller skips revalidation
once the run is in flight or has a `RunReference`
(`internal/controller/workflowrun/controller.go:129-135`), so a *completed* run is largely inert.
A pending run is the exposed case.

---

## 4. Scope — the pattern does **not** generalise

The repo already has the correct pattern in four places. These fetch the stored object and authorize
against on-disk ownership:

- `services/workload/service_authz.go:57-76` — *"Fetch the existing workload to get owner info for authz"*
- `services/releasebinding/service_authz.go:61-79`
- `services/projectreleasebinding/service_authz.go:55-74`
- `services/resourcereleasebinding/service_authz.go:56-76`

**`WorkflowRun` is the sole outlier that is actually exploitable.**

### Not vulnerable, though superficially similar

`UpdateComponent` (`component/service_authz.go:75-93`) and `UpdateResource`
(`resource/service_authz.go:67-86`) do build the hierarchy from body-supplied
`Spec.Owner.ProjectName`. But both internal services reject an owner mismatch **before any
mutation**:

- `component/service.go:126-128` → `"spec.owner.projectName is immutable"`
- `resource/service.go:117-119` → same

backed by `spec.owner` CEL immutability (`component_types.go:40`, `resource_types.go:15`). A caller
who lies about the project to pass authz gets a 422 and the stored object is untouched. No
unauthorized mutation is reachable. Correct outcome, but by ordering rather than by design — noted as
optional hardening in §6, deliberately **not** part of the fix.

All other `Update*` authz wrappers use `ResourceHierarchy{Namespace: namespaceName}` (namespace from
URL) or `Project: <name-from-URL>`, so none are affected.

---

## 5. The fix — two distinct defects

### Defect A — authorization decided on caller-supplied input

**File:** `internal/openchoreo-api/services/workflowrun/service_authz.go`, `UpdateWorkflowRun` (lines 90-105)

Fetch the stored object first and derive **every** identity-bearing attribute from it:

1. `existing, err := s.internal.GetWorkflowRun(ctx, namespaceName, wfRun.Name)`; propagate
   `ErrWorkflowRunNotFound` unchanged (the handler already maps it to 404 at `workflows.go:360`).
2. `Hierarchy: constructHierarchyForAuthzCheck(namespaceName, existing.Labels)`.
3. `Workflow: formatWorkflowAttr(namespaceName, existing.Spec.Workflow.Kind, existing.Spec.Workflow.Name)`.
4. `ResourceID: existing.Name`.

This mirrors `workload/service_authz.go:57-76` exactly — same shape, same ordering, same
small get-then-check TOCTOU window those call sites already accept. Follow the precedent; do not
invent a new mechanism. This alone closes the cross-component bypass **and** the §3.3 oracle, because
the denial now happens before any write is attempted.

### Defect B — ownership labels are mutable on update

Fix A stops an *unauthorized* caller. It does **not** stop a legitimately-authorized caller from
relabelling their own run into a project/component they do not own — which, per §3, hands the object
away. This needs its own guard.

**Fix A alone inverts the polarity of §2.5 rather than removing it.** Today authz reads the
*destination* labels, so theft is allowed and planting denied. After A it reads the *stored* labels,
so theft is correctly denied — but planting becomes allowed: an attacker updating their own run
(stored `component-a`, which they hold) passes the check, and the service then writes
`component-b` into the labels, injecting an attacker-controlled run into the victim's scope where the
victim's controller will validate and potentially execute it. A without B trades one directional
bypass for the other. This is the argument for treating them as two defects.

**File:** `internal/openchoreo-api/services/workflowrun/service.go`, `UpdateWorkflowRun` (lines 120-165)

Inside the `retry.RetryOnConflict` closure, **after** the `Get` populates `existing` and **before**
`existing.Labels = wfRun.Labels`, reject any change to the two ownership labels:

```go
for _, key := range []string{ocLabels.LabelKeyProjectName, ocLabels.LabelKeyComponentName} {
    if wfRun.Labels[key] != existing.Labels[key] {
        return &services.ValidationError{Msg: fmt.Sprintf("label %s is immutable", key)}
    }
}
```

Wording and error type mirror `component/service.go:126-128`. Placement inside the retry closure is
deliberate: `existing` is re-read on each conflict retry, so the check stays authoritative across the
retry window.

Two details to settle while implementing:
- `retry.RetryOnConflict` only retries on conflict, so returning a `*services.ValidationError` from
  the closure aborts and propagates. Confirm it survives the `errors.Is(err, ErrWorkflowRunNotFound)`
  / `ExtractValidationError` branches at `service.go:149-160` — a `*services.ValidationError` should
  be returned as-is rather than wrapped into the generic 500 at `:158`.
- A missing key and an empty-string value must compare equal, so an omitted label cannot be used to
  strip ownership. The map-index form above gives `""` for both, which is the behaviour wanted.

### Ordering
A and B are independent and can land in one commit. B is the one that must not be dropped — without
it the object can still be given away.

---

## 6. Optional hardening (explicitly out of scope of the fix)

- Move `UpdateComponent` / `UpdateResource` authz onto stored state for consistency with §4's
  precedent. Currently correct only because the internal service happens to validate before mutating;
  making the wrapper self-sufficient removes the ordering dependency. **No known exploit** — do not
  bundle this into the security fix.
- Consider CRD-level CEL or a validating webhook making `openchoreo.dev/project` and
  `openchoreo.dev/component` immutable on `WorkflowRun`, as defence in depth behind Defect B. Note
  that `TriggerWorkflow` (`service.go:922-938`) sets these at creation, so an immutability rule must
  permit the initial set.

---

## 7. Tests

`services/workflowrun/service_authz_test.go` already has the mock-PDP harness
(`NewTestServiceWithAuthz`, `newWorkflowRun`) and `testutil.NewFakeClient()`.

1. **Defect A regression.** Stored run labelled `project-b`/`component-b`; PDP configured to allow
   only `project-a`/`component-a`. PUT to the stored run's name with body labels
   `project-a`/`component-a`. Assert **both**: the call returns `ErrForbidden`, **and** re-reading via
   the fake client shows labels/annotations/spec unchanged. Asserting only the 403 would pass against
   the current buggy code paths in some orderings — the unchanged-object half is the load-bearing
   assertion.
2. **Defect B regression.** Authorized caller (PDP allows `project-a`/`component-a`), stored run
   labelled `project-a`/`component-a`, body attempts to change them to `project-b`/`component-b`.
   Assert a `*services.ValidationError` and that the stored labels are unchanged. Include a variant
   that *omits* the labels entirely, to cover the strip case.
3. **Positive control.** Authorized caller updating only `spec.workflow.parameters` and annotations,
   labels unchanged → succeeds. Guards against over-restricting the legitimate path.
4. **Hierarchy source.** Extend the existing `TestConstructHierarchyForAuthzCheck` block with a case
   asserting the update path reads `existing.Labels`, not the request body.

---

## 8. Pre-PR checklist

```sh
make code.gen && make lint-fix && make test && make code.gen-check
```

No API-type or OpenAPI-spec change is required for either defect, so `code.gen` should be a no-op —
if it is not, something in the change strayed out of scope.
