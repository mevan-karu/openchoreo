# Wiring Okta with OpenChoreo

In the [previous post](https://medium.com/@mevan.karu/your-idp-should-talk-to-your-idp-pluggable-identity-in-openchoreo-4e12362c3f63), we covered the foundational idea: an Internal Developer Platform should not manage its own identity in isolation. It should connect to the identity system your organization already runs, so that platform access reflects your real organizational state at all times. OpenChoreo is designed exactly for this, plugging into any OIDC-compliant identity provider without custom integrations.

This post is the first in a series of practical walkthroughs, each wiring OpenChoreo to a specific enterprise identity provider. We are starting with Okta. The setup covers Okta configuration and OpenChoreo Helm values. Okta has a few design decisions that differ from other OIDC providers; these are called out inline as they come up.

**Before you start, you will need:**
- A running OpenChoreo installation (control plane + observability plane)
- Okta admin access to create applications and configure authorization servers
- Helm access to both plane deployments

---

## Step 1: Set Up a Custom Authorization Server

Okta's Custom Authorization Server is the token issuer OpenChoreo will validate against. It controls what claims appear in tokens, what scopes are valid, and which applications can request tokens. You can use an existing one or create a new one.

**If your organization already has a custom authorization server**, you can reuse it. Skip to noting down the **Issuer URI** and **Audience** from its Settings tab. You will need both in Step 5.

**If you need to create one**, go to **Security → API** in the Okta Admin Console and click **Add Authorization Server**. Set a name, an audience (e.g. `openchoreo`), and save. Note down the **Issuer URI** and **Audience** from the Settings tab.

---

## Step 2: Configure Claims on the Access Token

This is where Okta diverges noticeably from other providers. Okta's custom authorization server does not automatically include user profile information or group membership in access tokens. Every claim must be explicitly configured.

OpenChoreo reads four claims from tokens:

| Claim | Purpose |
|---|---|
| `email` | Identifies the signed-in user in the developer portal |
| `given_name` | Display name in the portal |
| `family_name` | Display name in the portal |
| `groups` | Maps users to platform roles in the authorization engine |

Go to the **Claims** tab of your authorization server and click **Add Claim** for each of the following:

**email**
```
Name: email
Include in token type: Access Token → Always
Value type: Expression
Value: user.email
```

**given_name**
```
Name: given_name
Include in token type: Access Token → Always
Value type: Expression
Value: user.firstName
```

**family_name**
```
Name: family_name
Include in token type: Access Token → Always
Value type: Expression
Value: user.lastName
```

**groups**
```
Name: groups
Include in token type: Access Token → Always
Value type: Groups
Filter: Matches regex → .*
```
Include all groups with `.*`, or narrow to a specific prefix if your org uses one.

---

## Step 3: Create the OAuth Applications

OpenChoreo has several components that authenticate separately. Each one gets its own OAuth application in Okta.

**The developer portal (Backstage)** uses two grant types. Interactive user logins use authorization code. Backend calls from Backstage to the OpenChoreo control plane API use client credentials. Here is how to create it:

1. Go to **Applications → Applications** and click **Create App Integration**
2. Select **OIDC - OpenID Connect** and **Web Application**, then click **Next**
3. Set the app name (e.g. `OpenChoreo Backstage`) and enable both **Authorization Code** and **Client Credentials** grant types
4. Add the sign-in redirect URI: `https://<backstage-domain>/api/auth/openchoreo-auth/handler/frame`
5. Click **Save** and note down the **Client ID** and **Client Secret** (used in Step 5 under `backstage.auth.clientId` and the `backstage-secrets` Kubernetes secret)

The other applications follow the same flow with different settings:

**The CLI (`occ`)**: create it as a **Native Application** with **Authorization Code** and PKCE enabled. The redirect URI is `http://127.0.0.1:55152/auth-callback`. Port 55152 is the default the occ CLI listens on for the auth callback, use it exactly. There is no client secret for a native app.

**Service accounts** (Workload Publisher, Observer uid-resolver): create each as an **API Services** application. These are headless with no user interaction, so there is no redirect URI. Note down the client ID and client secret for each.

---

## Step 4: Add a Custom Scope for Service-to-Service Authentication

When an application uses the client credentials grant to request a token, Okta requires an explicit scope in the request. There is no implicit default, and built-in scopes like `openid` are restricted to user-facing flows. You need to create a custom scope for this purpose.

Here is how to add the scope:

1. Go to **Security → API**, open your authorization server, and click the **Scopes** tab
2. Click **Add Scope** and set the **Name** to `openchoreo_api` (or any name meaningful to your org; you will reference it in the OpenChoreo config)
3. Click **Create**

Then verify the access policy allows it for client credentials:

1. Click the **Access Policies** tab
2. Open the relevant policy rule and verify `openchoreo_api` is listed under allowed scopes for the **Client Credentials** grant type. 

---

## Step 5: Configure OpenChoreo

With the Okta side complete, the remaining work is in the OpenChoreo Helm values.

### Control Plane

Add the following to your control plane `values.yaml`. This covers the OIDC endpoints, the CLI public client registration, and the Backstage auth configuration:

```yaml
security:
  oidc:
    issuer: "https://{yourOktaDomain}/oauth2/{authorizationServerId}"
    wellKnownEndpoint: "https://{yourOktaDomain}/oauth2/{authorizationServerId}/.well-known/openid-configuration"
    jwksUrl: "https://{yourOktaDomain}/oauth2/{authorizationServerId}/v1/keys"
    authorizationUrl: "https://{yourOktaDomain}/oauth2/{authorizationServerId}/v1/authorize"
    tokenUrl: "https://{yourOktaDomain}/oauth2/{authorizationServerId}/v1/token"
    externalClients:
      - name: cli
        client_id: "<okta-cli-client-id>"
        scopes:
          - openid
          - profile
          - email
  jwt:
    audience: "openchoreo"

backstage:
  auth:
    clientId: "<okta-backstage-client-id>"
    redirectUrls:
      - "https://<backstage-domain>/api/auth/openchoreo-auth/handler/frame"
    oidcScope: "openid profile email"  # covers interactive user logins
    scope: "openchoreo_api"            # covers backend client credentials flow
```

Next, wire the service account identities to their platform roles. For client credentials tokens, the `sub` claim is the Okta client ID, so each service account is identified by it:

> **Important:** The `bootstrap.mappings` array replaces the chart defaults entirely when overridden. The chart ships with several default group-based bindings (admin, developer, platform-engineer, sre, workload-publisher). Your values file must carry all of them forward, not just the service account entries below. The full default set is in the [OpenChoreo Helm reference](https://openchoreo.dev/docs/reference/helm/control-plane).

```yaml
openchoreoApi:
  config:
    security:
      authorization:
        bootstrap:
          mappings:
            - name: backstage-catalog-reader-binding
              kind: ClusterAuthzRoleBinding
              system: true
              roleMappings:
                - roleRef:
                    name: backstage-catalog-reader
                    kind: ClusterAuthzRole
              entitlement:
                claim: sub
                value: "<okta-backstage-client-id>"
              effect: allow
            - name: observer-resource-reader-binding
              kind: ClusterAuthzRoleBinding
              system: true
              roleMappings:
                - roleRef:
                    name: observer-resource-reader
                    kind: ClusterAuthzRole
              entitlement:
                claim: sub
                value: "<okta-observer-client-id>"
              effect: allow
            # ... all other default mappings
```

### Observability Plane

Add the following to your observability plane `values.yaml`:

```yaml
security:
  oidc:
    issuer: "https://{yourOktaDomain}/oauth2/{authorizationServerId}"
    jwksUrl: "https://{yourOktaDomain}/oauth2/{authorizationServerId}/v1/keys"
    tokenUrl: "https://{yourOktaDomain}/oauth2/{authorizationServerId}/v1/token"
  jwt:
    audience: "openchoreo"

observer:
  oauthClientId: "<okta-observer-client-id>"
  oauthScope: "openchoreo_api"
  secretName: "observer-secret"
```

The `oauthScope` field is the custom scope from Step 4. Without it, the uid-resolver's client credentials requests will be rejected by Okta.

---

## Step 6: Onboard an Admin User

Before you can verify anything, your user needs platform access. OpenChoreo grants access based on the `groups` claim flowing from Okta, so the signed-in user must belong to a group that is bound to a platform role.

The default configuration ships with an `admin-binding` that maps the `admins` group to the admin role. Pick one of these two approaches:

**Option 1: Create the group in Okta.** Add a group named `admins` in your Okta directory and assign your user to it. No OpenChoreo configuration change needed.

**Option 2: Update the binding to match an existing group.** If you already have a relevant group in Okta (e.g. `platform-admins`), update the `admin-binding` entry in your control plane values:

```yaml
openchoreoApi:
  config:
    security:
      authorization:
        bootstrap:
          mappings:
            - name: admin-binding
              kind: ClusterAuthzRoleBinding
              roleMappings:
                - roleRef:
                    name: admin
                    kind: ClusterAuthzRole
              entitlement:
                claim: groups
                value: "platform-admins"   # replace with your actual Okta group name
              effect: allow
            # ... rest of your mappings
```

---

## Step 7: Verify the Integration

Navigate to the Backstage portal and sign in with your Okta account. If the login completes and platform resources (projects, components, environments) are visible and accessible, everything is working. A successful resource listing confirms the login flow and that the uid-resolver is authenticating to the control plane API correctly via client credentials.

For a quick check on the control plane side:

```bash
kubectl logs -l app.kubernetes.io/name=openchoreo-api \
  -n openchoreo-control-plane --tail=50
```

Look for any JWT validation or authentication errors. A clean log means tokens are being accepted correctly.

---

## What's Next

With Okta wired in, the next natural step is authorization: mapping your Okta groups to OpenChoreo roles so the right people have the right access. To learn more, refer to the [Authorization Configuration guide](https://openchoreo.dev/docs/platform-engineer-guide/authorization).

The next post in this series will walk through an integration with another enterprise identity provider.
