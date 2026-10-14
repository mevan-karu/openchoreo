# Argo CD and OpenChoreo authorization models compared

## How this document was sourced

The OpenChoreo claims come from reading the files cited beside each claim, in the working tree at commit `14a1aead5` on branch `fix/actorID_claim`. Paths are relative to the repository root.

The Argo CD claims come from three pages fetched during this work:

- https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/
- https://argo-cd.readthedocs.io/en/stable/user-guide/projects/
- https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/

plus the markdown source of the first of those, https://raw.githubusercontent.com/argoproj/argo-cd/stable/docs/operator-manual/rbac.md, which is where the eight-column resource and action table described below came from.

One caveat that affects how much weight to put on the Argo CD quotations. The fetch tool available here returns a summarizing model's rendering of the page rather than the raw page bytes, so wording marked as a quotation below is wording that came back identically across two independent fetches of the same passage. Where only one fetch produced a sentence, it is described rather than quoted. Two places where the summarizer was demonstrably wrong are flagged in the limitations section, and the Argo CD Casbin model file itself was never obtained.

## Argo CD, in Argo CD's vocabulary

Argo CD's RBAC lives in a ConfigMap, `argocd-rbac-cm`, and is written in a Casbin-derived CSV dialect. Two line types exist. A `g` line assigns a user or a group to a role: `g, <user/group>, <role>`. A `p` line grants a permission: `p, <role/user/group>, <resource>, <action>, <object>, <effect>`, where the effect is `allow` or `deny` (rbac doc). The subject on a `p` line can be a role, a user, or a group directly, so roles are a convenience rather than a required indirection.

`<resource>` is a fixed vocabulary of Argo CD object kinds. The rbac doc's table lists `applications`, `applicationsets`, `clusters`, `projects`, `repositories`, `accounts`, `certificates`, `gpgkeys`, `logs`, `exec` and `extensions`, with the actions each supports: `get`, `create`, `update`, `delete`, `sync`, `action`, `override` and `invoke`, not every action on every resource. `<object>` is the instance identifier, and for applications the doc says "the expected `<object>` value in the policy structure is replaced by `<app-project>/<app-name>`". Finer-grained application sub-resource control is done by expanding the action instead, "specify the action as `<action>/<group>/<kind>/<ns>/<name>`".

Two roles are built in: `role:readonly`, read-only access to all resources, and `role:admin`, unrestricted access to all resources. `policy.default` names the role every authenticated user falls back to. `scopes` names which OIDC claims are consulted for group membership and defaults to `'[groups]'`, in addition to the subject itself. `policy.matchMode` chooses between glob matching (gobwas/glob) and regex matching.

AppProjects carry a second, narrower policy surface. A project defines `roles:` with `policies:` entries, and the projects doc states that "It is very important that policy roles follow the pattern `proj:<project-name>:<role-name>`" or authorization fails. Project roles can be bound to OIDC groups or handed out as JWT tokens created with `argocd proj role create-token PROJECT ROLE-NAME`. The projects doc states that "A role's policy can only grant access to that role", and that cross-project rules belong in `argocd-rbac-cm` instead.

## OpenChoreo, in OpenChoreo's vocabulary

OpenChoreo's authorization is a Casbin enforcer whose policy store is the Kubernetes API. Four CRDs feed it: `AuthzRole` and `ClusterAuthzRole` hold action lists, `AuthzRoleBinding` and `ClusterAuthzRoleBinding` bind a JWT claim value to roles at a scope (`internal/authz/casbin/enforcer.go:33-38`). Informers on those four kinds translate every add, update and delete into incremental Casbin policy mutations (`internal/authz/casbin/k8s_watcher.go:29-54`); the enforcer runs with auto-save disabled so nothing can write policy back through the Casbin API (`internal/authz/casbin/enforcer.go:103`).

The model is embedded in the binary (`internal/authz/casbin/rbac_model.conf`). The request tuple is `r = sub, resource, act, ctx` (line 6). The policy tuple is `p = sub, resource, role, role_ns, eft, cond, binding_name` (line 16). The grouping tuple is `g = _, _, _`, documented in the same file as role, action, namespace (lines 18-23). The matcher is one line (line 29):

```
m = r.sub == p.sub && resourceMatch(r.resource, p.resource) && g(p.role, r.act, p.role_ns) && condMatch(r.ctx, r.act, p.cond, p.eft, p.binding_name)
```

A role is a bag of action strings and nothing else (`api/v1alpha1/authzrole_types.go:11-21`, `api/v1alpha1/clusterauthzrole_types.go:10-21`). Actions are `resource:verb` strings from a fixed registry of roughly 250 constants in `internal/authz/core/actions.go`, covering the platform CRDs plus observability surfaces such as `logs:view`, `auditlogs:view` and `metrics:view`.

A binding names one claim-value pair, `EntitlementClaim{Claim, Value}` (`api/v1alpha1/types.go:71-81`), plus a list of role mappings, each with an optional scope and optional conditions, plus an effect that defaults to `allow` (`api/v1alpha1/authzrolebinding_types.go:47-62`). The scope is a hierarchy position, not a name pattern: project, component or resource for a namespaced binding (`api/v1alpha1/authzrolebinding_types.go:13-25`), with namespace added for a cluster binding (`api/v1alpha1/clusterauthzrolebinding_types.go:13-29`). `resourceHierarchyToPath` flattens that into a path such as `ns/acme/project/p1/component/c1`, and an empty hierarchy becomes `*` (`internal/authz/casbin/helpers.go:363-389`).

Subjects are formed as `claim:value` (`internal/authz/casbin/helpers.go:429-434`). The authenticated caller carries a claim name and a list of values in its `SubjectContext` (`internal/server/middleware/auth/interface.go:19-27`), taken from the configured entitlement claim of whichever user type matched the token (`internal/server/middleware/auth/jwt/resolver.go:89-111`), where a claim holding a string yields one value and a claim holding an array yields many (`internal/server/middleware/auth/jwt/resolver.go:144-161`).

Bindings can also carry ABAC conditions. A condition is a list of action patterns plus a CEL expression (`api/v1alpha1/types.go:487-501`), evaluated against a small attribute set registered per action: `resource.environment`, `resource.componentType`, `resource.resourceType`, `resource.workflow` (`internal/authz/core/condition_registry.go:36-98`). Conditions that do not target the requested action are ignored, and matching ones are OR'd (`internal/authz/casbin/helpers.go:120-136`, `195-212`).

## Concept mapping

| Concept | Argo CD | OpenChoreo |
|---|---|---|
| Subject | user, group, or `proj:<project>:<role>`, matched by glob or regex | `claim:value` string, compared with `==` |
| Role | `role:<name>` defined by the `p` lines that name it, or the two built-ins | `AuthzRole` / `ClusterAuthzRole` CRD holding an action list |
| Binding | `g, <user/group>, <role>` line, or an AppProject role's `groups:` | `AuthzRoleBinding` / `ClusterAuthzRoleBinding` CRD |
| Resource | fixed kind vocabulary (`applications`, `clusters`, …) plus an `<object>` identifier | hierarchy path string `ns/<n>/project/<p>/component/<c>` |
| Action | `get`, `create`, `sync`, `override`, `invoke`, … per resource kind | `resource:verb` from a fixed registry, e.g. `component:exec` |
| Scope / hierarchy | project prefix inside the object, `<app-project>/<app-name>` | four-level ownership path, prefix match downward |
| Default / fallback | `policy.default` grants a floor to every authenticated user | none; unmatched request is denied |
| Policy storage | `argocd-rbac-cm` ConfigMap, plus AppProject specs | four CRD kinds, watched into an in-memory Casbin enforcer |
| Authoring surface | edit the ConfigMap CSV, or `argocd proj role` commands | apply CRDs, or the REST API's PAP (`internal/authz/casbin/pap.go:19-380`) |

Reading across that table, the two systems agree on the shape of a row and disagree on almost every cell's type. Both put a subject, a permission and an object in one tuple and both use Casbin to evaluate it, but Argo CD's object is a name to be pattern-matched while OpenChoreo's is a path to be prefix-matched, and Argo CD's subject is a pattern while OpenChoreo's is a literal. The role row is the sharpest difference in authoring feel: in Argo CD a role has no declaration of its own, it exists because `p` lines mention it, so granting a permission and defining a role are the same edit. In OpenChoreo the two are separate objects with separate lifecycles, and a role that no binding references is still a valid, inert object. The fallback row is the one an operator will feel first, because Argo CD ships a lever that OpenChoreo simply does not have. The storage row explains the operational difference: an Argo CD change is a ConfigMap edit, an OpenChoreo change is a CRD apply that an informer turns into a policy delta (`internal/authz/casbin/k8s_watcher.go:537-555`).

## What is genuinely close

Both systems evaluate with Casbin and both settle a conflict the same way at the model level. OpenChoreo's effect line is `e = some(where (p.eft == allow)) && !some(where (p.eft == deny))` (`internal/authz/casbin/rbac_model.conf:26`), which is the deny-overrides semantics Argo CD's doc describes when it says a `deny` takes priority even where a more specific `allow` also matches.

Both accept an explicit `deny` as a first-class effect rather than only additive grants. OpenChoreo's `EffectType` enum is `allow` or `deny` with `allow` as the CRD default (`api/v1alpha1/types.go:477-485`, `api/v1alpha1/authzrolebinding_types.go:58-61`).

Both derive the subject from an OIDC token's group-ish claim by configuration rather than hard-coding one. Argo CD's `scopes` defaults to `'[groups]'`; OpenChoreo's entitlement claim is per user type and per auth mechanism, validated as non-empty at config load (`internal/server/middleware/auth/subject/config.go:30-34`, `87-90`), and the shipped bootstrap bindings use `groups` for humans and `client_id` for service identities (`install/helm/openchoreo-control-plane/values.yaml:1732` onward).

Both let one subject accumulate permissions from several grants, and both treat multiple group memberships as multiple chances to be allowed. OpenChoreo loops over every entitlement value and returns on the first allow (`internal/authz/casbin/pdp.go:262-287`).

Both ship a small set of named starting roles. Argo CD has `role:readonly` and `role:admin`. OpenChoreo's chart bootstraps `admin` (`install/helm/openchoreo-control-plane/values.yaml:1345`), `cluster-reader` (line 1351), `namespace-reader` (line 1365), `developer` (line 1383), `sre` (line 1459), `platform-engineer` (line 1516) and five service roles (lines 1259, 1292, 1303, 1326, 1336), applied through a post-install hook rather than as tracked release resources (`install/helm/openchoreo-control-plane/templates/authz/_authz-bootstrap.tpl:1-10`).

## What is genuinely different

**`g` means opposite things.** In Argo CD, `g` maps a subject to a role. In OpenChoreo, `g` maps a role to an action within a namespace (`internal/authz/casbin/rbac_model.conf:18-23`), and a role's actions are written by the role watcher as `{roleName, action, namespace}` triples (`internal/authz/casbin/k8s_watcher.go:206-209`, cluster roles use `"*"` as the namespace at line 233). Subject-to-role mapping in OpenChoreo happens in the `p` line's subject field, not in `g`. There is no role-inherits-role construct anywhere in the model.

**Subject matching is exact.** The matcher's first clause is `r.sub == p.sub` (`internal/authz/casbin/rbac_model.conf:29`). No glob, no regex, no wildcard subject. A binding must name the claim value literally. Argo CD, by contrast, runs the subject through glob or regex per `policy.matchMode`.

**Wildcards live in different places.** OpenChoreo supports `*` as an entire action, `resource:*` as an action prefix (`internal/authz/casbin/helpers.go:308-323`), `*` as the whole resource path (`internal/authz/casbin/helpers.go:90-92`), and `*` as the role namespace for cluster roles. It supports no pattern inside a resource path. Matching is strict hierarchical prefix with an appended separator, `strings.HasPrefix(requestResource, policyResource+"/")` (`internal/authz/casbin/helpers.go:100`), so a grant on `ns/acme` covers everything beneath it and cannot leak into `ns/acmecorp`.

**Scope is structural, not textual.** An Argo CD application policy scopes by writing an object pattern such as `my-project/*`. An OpenChoreo binding scopes by filling in hierarchy fields, and the CRD enforces the shape with CEL validation: component requires project, resource requires project, component and resource are mutually exclusive (`api/v1alpha1/authzrolebinding_types.go:28-30`, and the cluster equivalent at `api/v1alpha1/clusterauthzrolebinding_types.go:32-34`). You cannot express "every project whose name starts with pay".

**OpenChoreo has ABAC, Argo CD does not, in the material read.** A binding can attach CEL conditions restricted to named actions (`api/v1alpha1/authzrolebinding_types.go:39-44`), which is how "deploy to dev and staging but not prod" is expressed: `resource.environment in ["dev", "staging"]` (`api/v1alpha1/types.go:495-498`). Condition failures fail closed by effect, so a broken expression on an allow stops granting and a broken expression on a deny keeps denying (`internal/authz/casbin/helpers.go:40-48`, `195-212`). Nothing equivalent appeared in the Argo CD pages fetched.

**Policy authoring is an API, not a file.** OpenChoreo's PAP exposes create, get, list, update and delete for all four kinds through the control plane API, backed by ordinary Kubernetes writes (`internal/authz/casbin/pap.go:24-37` for the pattern, with the last handler at line 373). Argo CD's primary surface is a ConfigMap you edit, with `argocd proj role` commands for the project-scoped subset.

**Decisions can be introspected in bulk.** `GetSubjectProfile` returns, for a scope, every action the subject holds with the paths and CEL constraints attached (`internal/authz/casbin/pdp.go:54-85`, `131-231`), which is what a UI needs to grey out buttons. No equivalent appeared in the Argo CD docs read here.

## Where an assumption in one model is wrong in the other

**"The default policy gives everyone a floor."** Argo CD's rbac doc says, in wording that came back identically from two fetches, "All authenticated users get *at least* the permissions granted by the default policies. This access cannot be blocked by a `deny` rule." OpenChoreo has no such concept. An unmatched request returns `Decision: false` with the reason "no matching policies found" (`internal/authz/casbin/pdp.go:257-261`), which the API layer turns into `ErrForbidden` (`internal/openchoreo-api/services/authz.go:85-87`). An Argo CD configuration that leans on `policy.default` for baseline read access ports to an installation where authenticated users can do nothing.

**"A deny rule shuts a subject out."** In Argo CD the deny is evaluated against the whole matched policy set, subject patterns included. In OpenChoreo `check` calls `Enforce` once per entitlement value and breaks on the first true (`internal/authz/casbin/pdp.go:262-287`), and the matcher only considers policies whose `p.sub` equals that one entitlement. Reading the code, a deny bound to `groups:contractors` therefore cannot suppress an allow the same user holds through `groups:developers`, because the two never appear in the same evaluation. Deny-overrides-allow holds within one claim value and not across them. I could not find a test covering this: every deny case in `internal/authz/casbin/pdp_test.go` (the tables at lines 431-520 and around line 915) uses a single-element `entitlementValues`, and no test in that file passes two values with conflicting effects. Treat this as a code reading that the test suite does not confirm or refute. Nothing in the `EffectType` documentation (`api/v1alpha1/types.go:477-485`) or the binding CRD comments describes this boundary.

**"The deny is safe because the policy list is ordered or specific."** Neither system orders by specificity. Argo CD's doc is explicit that a more specific allow does not beat a deny. OpenChoreo's effect line is set-based in the same way. Someone carrying an ordered-ACL intuition from a firewall or an ingress config will be wrong in both.

**"Wildcards behave like paths."** The Argo CD doc warns, in wording reproduced identically by two fetches, that "Argo CD RBAC does not use `/` as a separator when evaluating glob patterns". One fetch added the doc's own example, that the pattern `delete/*/kind/*` matches `delete/<group>/kind/<namespace>/<name>` but also `delete/<group>/<kind>/kind/<name>`; that clause came back only once, so treat the example as reported rather than quoted. OpenChoreo's path matching is the opposite: separator-aware, prefix-only, no in-path patterns (`internal/authz/casbin/helpers.go:88-101`). A reviewer who has learned to be suspicious of Argo CD globs will over-suspect OpenChoreo paths, and a reviewer coming the other way will under-suspect Argo CD globs.

**"A role is scoped by where it is defined."** An OpenChoreo `AuthzRole` is namespaced, but the namespace is a role-identity qualifier, not a grant boundary: it becomes `role_ns` in the policy tuple and is matched exactly (`internal/authz/casbin/k8s_watcher.go:274-284`, `internal/authz/casbin/rbac_model.conf:16`). The grant boundary is the binding's scope path. A `ClusterAuthzRoleBinding` with no scope produces the path `*` and therefore grants everywhere (`internal/authz/casbin/helpers.go:363-368`), which is exactly what the shipped `admin-binding` does with the `admin` role's single `"*"` action (`install/helm/openchoreo-control-plane/values.yaml:1345-1347` and the mappings block from line 1732).

**"Authorization is on."** OpenChoreo can be configured with a `DisabledAuthorizer` whose `Evaluate` returns allow for everything and whose profile reports `"*"` on `"*"` (`internal/authz/disabled_authorizer.go:31-43`, `66-85`). There is no Argo CD analogue in the pages read. Any reasoning about OpenChoreo policy assumes the Casbin enforcer is the one wired in.

## Limitations of each model

### Argo CD

The default-policy floor cannot be clawed back by a deny, per the sentence quoted above, so the only way to reduce a baseline is to change `policy.default` itself.

Glob matching without `/` as a separator, with the doc's own example of a pattern matching more than intended, makes fine-grained sub-resource rules easy to write wrongly.

The resource vocabulary is closed and small. Everything an installation wants to control has to be expressed as one of the listed kinds with one of the listed actions, and the table shows several kinds supporting only a subset.

Project-scoped roles are a second policy surface with its own rules. The projects doc states that "A role's policy can only grant access to that role", and that cross-project rules must live in `argocd-rbac-cm`, so operators maintain policy in two places with different syntaxes for the same engine.

The user-management page warns that local users "will need additional RBAC rules set up, otherwise they will fall back to the default policy specified by `policy.default`", which makes a local account a quiet path to whatever the default grants.

Two things I could not verify. The first fetch of the rbac page reported that it contains no mention of project roles or the `proj:` pattern, and a later fetch of the same page reported a `proj:team-beta-project:admin` example; the first report is a summarizer false negative and no conclusion about the page's coverage should be drawn from it. The second is a claim, returned once, that an SSO user whose scope matches a local user name inherits that local user's roles. The user-management page fetch explicitly did not confirm it. I am not asserting it here.

Argo CD's actual Casbin model file was never obtained, only the CSV dialect as documented. Statements about its matcher internals, effect expression, or how `g` is chained are therefore out of scope rather than merely uncertain.

### OpenChoreo

There is no fallback role and no way to express one, so every install depends on the chart's bootstrap bindings or on bindings an operator writes. The chart applies those through a post-install and post-upgrade hook, and the values file warns that "Hand-edits made directly against these objects in the cluster do not survive the next upgrade" (`install/helm/openchoreo-control-plane/values.yaml:1225`).

Deny does not compose across entitlement values, as described above. A model that admits `deny` but cannot guarantee it across a subject's full group set is a sharp edge, and the CRD says nothing about it.

`GetSubjectProfile` and `Enforce` do not use the same scope predicate. `isWithinScope` matches in both directions, parent-of-scope and child-of-scope (`internal/authz/casbin/helpers.go:498-522`), while enforcement uses one-directional `resourceMatch` (`internal/authz/casbin/helpers.go:88-101`). The profile is therefore a superset view suitable for driving a UI and is not an authorization oracle. The code comments describe each function correctly on its own; nothing states the asymmetry between them.

Policy state is an in-memory projection maintained by informer deltas, with the enforcer's auto-save off (`internal/authz/casbin/enforcer.go:103`, `internal/authz/casbin/k8s_watcher.go:161-179`). The watcher logs warnings when an expected policy is absent on update or delete rather than rebuilding, for instance "binding delete: no policies matched for removal, possible key mismatch or stale policy" (`internal/authz/casbin/k8s_watcher.go:769-773`). The chart exposes a `resync_interval` defaulting to `10m` described as a safety net for missed events (`install/helm/openchoreo-control-plane/values.yaml:1219-1222`), which is an acknowledgement that drift is possible.

Role updates key on `metadata.generation`, so an update whose generation is unchanged is skipped entirely (`internal/authz/casbin/k8s_watcher.go:374-378` and the same guard in the three sibling handlers).

The ABAC attribute surface is four attributes on a fixed list of actions (`internal/authz/core/condition_registry.go:66-98`). Conditions on any other action are accepted by the CRD but have nothing to read: an action with no registered attributes gets an empty CEL activation (`internal/authz/casbin/helpers.go:618-621`), so an expression referencing `resource.environment` there fails to evaluate and falls closed by effect rather than being rejected at authoring time.

CEL expressions are compiled per evaluation (`internal/authz/casbin/helpers.go:215-222`, `596-611`), with no compiled-program cache. The only cache offered is a whole-decision cache, off by default, with a 5 minute TTL (`internal/authz/casbin/enforcer.go:63-74`, `install/helm/openchoreo-control-plane/values.yaml:1204-1216`); a decision cache means a revoked binding can still be honoured for up to that TTL.

`ErrCannotModifySystemMapping` and `ErrCannotDeleteSystemMapping` are declared (`internal/authz/core/types.go:154-155`) but I found no non-test code that returns them. A grep across `internal/` for those identifiers and for the `openchoreo.io/system` label the chart stamps on system roles and bindings (`install/helm/openchoreo-control-plane/templates/authz/_authz-bootstrap.tpl:33-36`, `84-86`) matched only those two declaration lines. On that evidence the PAP does not appear to refuse edits to system-labelled objects, but a negative grep is weaker than reading every caller, so this is reported as unverified rather than as a defect.

Subject identity for authorization is the entitlement claim only. The `SubjectContext` doc comment is explicit that `ReadableID` "must never be used to make an access decision" because the claim carrying it is operator-configurable (`internal/server/middleware/auth/interface.go:12-18`), and `ID` is documented as unique only within the issuer (line 20). Nothing in the policy tuple records an issuer, so two issuers emitting the same group name are the same subject to the enforcer.
