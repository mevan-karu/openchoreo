# Audit: resolve actor.id from a per-mechanism claim

Scope: one fix, one PR. The subject-type detection bug found alongside it is
parked — see "Parked" at the end.

## Problem

`audit.actor.id_claim` is a single claim name applied to every actor type
(`internal/server/middleware/audit/adapter.go:73-83`). Both charts default it
to `username` (`install/helm/openchoreo-control-plane/values.yaml:1949`,
`install/helm/openchoreo-observability-plane/values.yaml:574`). A
`client_credentials` token carries no `username`, and `actorIDFromContext`
deliberately records `"unknown"` rather than falling back to `sub`
(`adapter.go:45-48`). Every service-account record therefore lands as
`actor.id: "unknown"` — matching the observation in
`audit-log-tryout-2026-09-22.md:62`.

## Design decisions

**The claim lives under the mechanism, not the subject type.**
`entitlement.claim` is already nested at
`subjects.<type>.mechanisms.<mech>.entitlement.claim`
(`security.go:227-230`), and the resolver selects a mechanism before reading
anything (`resolver.go:73-82`). "Which claim" is a JWT-shaped question — an
`api_key` mechanism has no claims at all. Placing it on the type would assert
a claim name for mechanisms that have none.

**Named `readable_id_claim` / `SubjectContext.ReadableID`.**
`SubjectContext.ID` is security-load-bearing: it signs remote-connect
capability tokens (`remoteconnect.go:125` -> `:503` -> `:1059`). The new
field must never be used that way. `ID` beside `ReadableID` makes that
distinction visible at every call site; `ID` beside `ActorID` reads as two
interchangeable identifiers. It names the property rather than the consumer,
so it survives a second reader (a UI, a log line).

**`SubjectContext.ID` is not repurposed.** Authorization does not read it —
`GetAuthzSubjectContext` (`internal/authz/core/converter.go:16-20`) copies
only Type and entitlements — but `remoteconnect.go:125` feeds it into the
signed capability's `Subject`. Changing its claim source would silently
change what a signed capability binds to.

**Resolving in `jwt/resolver.go` reads validated claims.** The resolver uses
`ParseUnverified`, but `jwt.Parse` verifies the signature against JWKS and
`validateClaims` checks issuer/audience at `jwt/middleware.go:100-164`;
`Detector.ResolveUserType` is only reached at `:186`. The resolver re-parses
a token the middleware has already accepted.

**The audit fallback is applied audit-side, not in the resolver.**
`audit.actor.id_claim` is audit config and the resolver is auth middleware.
The resolver resolves only the mechanism-level claim and leaves `ReadableID`
empty when unset or absent; `actorIDFromContext` applies the rest of the
chain. Consequence: the `MiddlewareConfig.ActorIDClaim` plumbing stays as the
middle rung — no signature changes to `ExtractActor`, `EmitFromContext`,
`MiddlewareConfig` or `mcpaudit`.

**Precedence:** `subjects.<type>.mechanisms.jwt.readable_id_claim` ->
`audit.actor.id_claim` -> `sub` -> `"unknown"` only if `sub` is absent too.

**`audit.actor.id_claim` stays, deliberately.** Dropping it was considered:
it landed 2026-09-17 (`48d803d6d`, #4759) and has shipped in one GA release,
`v1.3.0`, so the installed base that could have customized it is negligible,
and both failure modes would have been loud rather than silent — helm
refuses the values (`additionalProperties: false` on the actor block in both
`values.schema.json`) and koanf fails startup (`ErrorUnused: true`,
`internal/config/loader.go:113`). It stays anyway, to keep this change
non-breaking and to leave operators a single global setting. Reviewers asking
why two settings decide actor identity: the per-mechanism key is the
specific answer, the global one is the fallback for mechanisms that declare
none.

Trade-off accepted: `actor.id` can come from different claims across records
with nothing in the envelope saying which, which makes `--actor` filtering
and `policies[].match.actors` less reliable across actor types. Keeping the
global rung widens that slightly, since the chain is three deep rather than
two. See "Deferred".

## Code

- `internal/server/middleware/auth/subject/config.go` — `ReadableIDClaim` on
  `AuthMechanismConfig`, optional (no `ValidateConfig` requirement). Observer
  inherits the config key for free: `internal/observer/config/config.go:92`
  decodes `auth.subject_types` straight into these structs.
- `internal/openchoreo-api/config/security.go:227` — `ReadableIDClaim` on
  `MechanismConfig` (`koanf:"readable_id_claim"`); one line in
  `ToSubjectUserTypeConfigs` (`:271-286`), which currently copies only `Type`
  and `Entitlement`.
- `internal/server/middleware/auth/interface.go` — `SubjectContext.ReadableID`.
- `internal/server/middleware/auth/jwt/resolver.go:90-101` — read the matched
  mechanism's `ReadableIDClaim` from `claims` into `ReadableID`; leave empty
  when unset or absent.
- `internal/server/middleware/audit/adapter.go:73-83` — `actorIDFromContext`
  prefers `subjectCtx.ReadableID`, then the configured `audit.actor.id_claim`,
  then `subjectCtx.ID`.

## Config and charts

- `cmd/openchoreo-api/config.yaml` (subjects block ~`:84-111`)
- `cmd/observer/auth-config.yaml`
- `install/helm/openchoreo-control-plane/values.yaml` +
  `templates/openchoreo-api/configmap.yaml`
- `install/helm/openchoreo-observability-plane/values.yaml` +
  `templates/observer/observer-auth-configmap.yaml`

Both charts render the whole subjects block with `toYaml`
(`configmap.yaml:37`, `observer-auth-configmap.yaml:15`), so new keys pass
through the templates untouched. But the `@schema` annotations in both
`values.yaml` files enumerate the nested properties explicitly (observer's at
`values.yaml:~480-502`), so those must be extended for `readable_id_claim` or
schema validation rejects it. `make code.gen-check` catches a miss.

Shipped defaults are **not** decided here — see "Verification before
implementation". `client_id` for `service_account` is safe (the type is
detected by that claim's presence, so a matched token provably carries it),
but `username` for `user` is inherited from the very chart default that
caused this bug and is unverified.

## Docs

- `internal/server/middleware/audit/types.go:15` — the `Actor.ID` comment
  states the old single-claim semantics.
- `openapi/observer-api.yaml:2337` and
  `openapi/observability-logs-adapter-api.yaml:1566` — same sentence.
  `internal/observer/api/logsadapterclientgen/client.gen.go:304` is generated
  from these, so `make code.gen` is mandatory.
- `internal/occ/cmd/auditlogs/cmd.go:75` — `--actor` flag help.

## Tests

- Resolver populates `ReadableID` from the matched mechanism.
- Full precedence chain, including the absent-claim fall-through to `sub`.
- `ToSubjectUserTypeConfigs` carries the new field.
- Update `TestExtractActor_ConfiguredIDClaim`
  (`audit/middleware_test.go:115-165`).

## Verification before implementation

The `readable_id_claim` chart default depends on what Thunder actually puts
in its tokens, which is not knowable from this repo. Decode one user token
and one `client_credentials` token from the k3d cluster and report both claim
sets before setting any default. The bug being fixed is a chart default that
assumed a claim which was not there — do not repeat it by guessing.

If no suitable claim is confirmed for `user`, ship that default empty and let
the `audit.actor.id_claim -> sub` fallback carry it.

## Migration impact

### Safe

- **Old config, new binary.** The new key is optional and absent config takes
  the existing path. Nothing fails to start.
- **Go struct additions.** `auth.SubjectContext`, `subject.AuthMechanismConfig`
  and `MechanismConfig` all live under `internal/`, so there are no external
  consumers.
- **OpenAPI edits** are description-only; no request or response shape
  changes. `client.gen.go` regenerates but its types are unchanged.
- **`occ --actor` help text** only.
- **values.schema.json** gains one optional property. Existing values files
  that do not set it still validate.

### Changes recorded data — no config change required

An upgrade alone changes `actor.id` from `"unknown"` to the token's `sub` for
every actor whose token lacks the configured `audit.actor.id_claim` — today,
every service account. That is the fix, but it has consequences for anyone
already querying the trail:

- Saved queries, dashboards or alerts filtering on `actor.id: "unknown"`
  stop matching.
- A `policies[].match.actors` selector naming `"unknown"` stops matching.
- Historical records straddle the boundary: the same service account appears
  as `"unknown"` before the upgrade and as its `sub` after, so a by-actor
  aggregation over a window spanning the upgrade splits in two.

Belongs in the release note. No action required from operators.

### Breaking on rollback

`internal/config/loader.go:113` sets `ErrorUnused: true`, so openchoreo-api
**fails to start on any unknown config key**. An operator who upgrades and
then rolls the image back while keeping the new ConfigMap gets a startup
failure on `readable_id_claim`, not a graceful degrade.

Observer differs: its auth section is read with plain `yaml.Unmarshal`
(`internal/observer/config/config.go:226`), which ignores unknown fields, so
observer tolerates a rollback. Its audit section uses koanf with
`ErrorUnused: true` (`config.go:396`) like the API.

Mitigation is documentation, not code: rolling back the image requires
rolling back the ConfigMap with it. `helm rollback` does both; an image-only
revert does not.

## Gate

`make code.gen && make lint-fix && make test && make code.gen-check`

---

## Parked — subject-type detection by entitlement-claim presence

Found while investigating the above. Not in this PR.

`jwt/resolver.go:96` is the only production writer of `SubjectContext`; there
is no catch-all and no default type. Types are tried in priority order and
each matches iff *its entitlement claim is present*
(`detectUserTypeFromClaims`, `resolver.go:107-128`). First match wins.

With `user` (priority 1, `groups`) and `service_account` (priority 2,
`client_id`), a token without `groups` is tested against `service_account`
next and classified as one if it carries `client_id` — which OAuth tokens
routinely do. `GetAuthzSubjectContext` then hands the PDP `service_account`
plus `client_id` entitlements, so **a misclassified token is authorized as
the wrong subject type.**

The agreed design, for when this is picked up: a `match` block on the
mechanism, holding a list of claim conditions that must all hold, each
asserting presence (default), absence, or membership in a value set.
`match` absent keeps today's behaviour, so nothing breaks on upgrade.
Separating identity (`match`) from authorization (`entitlement`) is the
point.

The open decision that parked it: shipping `match` in the charts is what
makes it breaking, because a chart upgrade would then change authentication
and authorization for every deployment on chart defaults — a token currently
resolving to `service_account` could lose those permissions, or 401 outright.
Whether that fires depends on what Thunder emits, so the token decode above
informs this too.

## Deferred

An `actor.id_claim` field in the audit envelope would record which claim
produced each `actor.id`, closing the ambiguity this PR accepts. The schema
freeze (`audit-schema-v1-pr-description.md`) permits additions as a minor
bump to 1.1. Separate PR.
