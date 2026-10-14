 OpenChoreo is configured with an OAuth2/OIDC compliant identity provider. Thunder ships as the default, and a deployment can point it at something else. Clients authenticate against that IdP and call the OpenChoreo API with the access token it issues, and the claims in that token are everything we know about the caller. The middleware validates the token, then resolves a subject context out of those claims. That context holds two things: what kind of actor this is, and the entitlement values they carry.

Authorization runs off those entitlement values. Roles are bound to entitlements through `ClusterAuthzRoleBinding` and `AuthzRoleBinding`, and an action is allowed when a bound role covers it. So authentication decides whether you get in, and entitlements decide what you can do once you are there.

## Problem

OpenChoreo works out what kind of actor sent a token by looking for a configured claim. The [default config](https://github.com/openchoreo/openchoreo/blob/1aeed89856d088dc5160c0f3eaa39796d4a93611/install/helm/openchoreo-control-plane/values.yaml#L1176-L1194) gives `user` the `groups` claim at priority 1, and `service_account` the `client_id` claim at priority 2. [`detectUserTypeFromClaims`](https://github.com/openchoreo/openchoreo/blob/1aeed89856d088dc5160c0f3eaa39796d4a93611/internal/server/middleware/auth/jwt/resolver.go#L119) picks the first type whose claim is present and holds at least one value.

A missing claim does not mean one thing. It can mean we never asked for it, or that the IdP declined to send it, or that the user genuinely has no values for it. The token does not tell you which, and we read all three as "not a user".

So sign-in as a Thunder user who belongs to no group. Thunder leaves `groups` out of the token, `user` does not match, and you are classified as a service account. That is worse than a wrong label in audit records, because the authorization engine matches policies on the entitlement claim and its value. You are evaluated as `client_id:<your client>`, and you inherit any role bound to that client id.

No spec settles this. [RFC 9068 §2.2.3.1](https://www.rfc-editor.org/rfc/rfc9068.html#section-2.2.3.1) says to encode `groups` the way [RFC 7643 §2.5](https://www.rfc-editor.org/rfc/rfc7643.html#section-2.5) describes, and RFC 7643 treats an empty array and an absent attribute as the same state. Omitting is conformant, and so is sending `[]`. Entra omits and [says so outright](https://learn.microsoft.com/en-us/entra/identity-platform/access-token-claims-reference): "Claims are present only if a value exists to fill it. An application shouldn't take a dependency on a claim being present." Keycloak switched from `[]` to omitting in 22.0.1 and [closed the report as not planned](https://github.com/keycloak/keycloak/issues/22340), over this same ambiguity. Expect the next IdP to differ again, and expect it to be entitled to.

Our own fallback then hides it. [RFC 9068 §2.2](https://www.rfc-editor.org/rfc/rfc9068.html#section-2.2) requires `client_id` on every JWT access token, user flows included, so priority 2 matches every time and classification never fails in a way anyone would notice. There is no standard claim we could key off instead either. The [IANA JWT registry](https://www.iana.org/assignments/jwt/jwt.xhtml) has around 180 entries and none of them names the principal type or the grant. Entra has an optional `idtyp`, Thunder emits `grant_type`, and each IdP has its own answer.

## Prior art

This is not a new problem. Kubernetes and Argo CD both take whatever OIDC provider you give them while owning their own authorization, so both have had to answer it.

- **Kubernetes** keeps two things apart in its [AuthenticationConfiguration](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#using-authentication-configuration). `claimValidationRules` decides whether a token is accepted, either as a `claim` plus a `requiredValue` or as a CEL expression. `claimMappings.username` and `claimMappings.groups` decide what identity and groups RBAC then authorizes. Deciding and extracting are different fields.
- **Argo CD** is the closest to where we are. It runs Casbin like we do, and its [`scopes`](https://argo-cd.readthedocs.io/en/stable/operator-manual/rbac/) setting names which claims RBAC subjects are matched from. It defaults to `'[groups]'` and takes a list, so `'[groups, email]'` is valid. `policy.default` names the role an authenticated user falls back to when no policy matches them. The other half sits in [`oidc.config`](https://argo-cd.readthedocs.io/en/stable/operator-manual/user-management/), where `requestedScopes` and `requestedIDTokenClaims` declare which claims should arrive in the first place. The IdP specific part is claim names in config, and the authorization model underneath does not move.

The common thread is that deciding what someone is and reading what they are entitled to are two steps, not one, and the IdP specific part lives in config.

## Solution

Take the same split. A `match` block decides the subject type, and `entitlement` goes back to only pulling out values.

```yaml
subjects:
  service_account:
    priority: 1
    mechanisms:
      jwt:
        match:
          - claim: "grant_type"
            values: ["client_credentials"]
        entitlement:
          claim: "client_id"
  user:
    priority: 2
    mechanisms:
      jwt:
        entitlement:
          claim: "groups"
```

`match` is a list of conditions and all of them have to hold. A condition can check a claim against a list of accepted values, as above. It can also check that a claim is present at all, with `exists: true`. Entra is where that second form helps, because by default it only emits `idtyp` on app-only tokens, so the claim being there is itself the signal.

`match` is optional. A mechanism without one behaves as it does today and detects on the entitlement claim being present, so nothing that works now stops working. Where a `match` is set, it decides the type. A token that matches nothing gets a default type with no entitlements.

Classification then stops depending on what an IdP leaves out. Thunder points `match` at `grant_type`. Entra points it at `idtyp`. A user in no group stays a user with an empty entitlement list, because the `user` type no longer needs `groups` to be there.

A claim and a list of accepted values is deliberately less than Kubernetes allows. CEL is where this goes if a claim and a value turn out not to be enough.

## Concerns

Detection gets stricter for anyone running the shipped defaults. Making `match` optional protects a deployment that writes its own subject config. It does not protect the chart default, which is shaped for Thunder. Someone running the Helm defaults against another IdP could find their users locked out.

Appreciate any feedback on the direction.
