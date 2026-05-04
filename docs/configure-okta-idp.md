# Configuring Okta as the Identity Provider for OpenChoreo

This guide walks through integrating Okta as the identity provider for OpenChoreo. By the end you will have:

- An Okta authorization server configured for OpenChoreo
- OAuth 2.0 applications created for the Backstage portal, CLI, and CI workflows
- OpenChoreo control plane and observability plane configured to validate Okta-issued JWTs
- Groups claims flowing through so authorization policies can reference your Okta groups

## Prerequisites

- An Okta account with admin access (Okta Workforce Identity Cloud or Okta Developer Edition)
- OpenChoreo installed with Thunder as the default identity provider (or a fresh install where you are choosing the identity provider)
- `kubectl` access to the cluster running the OpenChoreo control plane
- The Helm CLI installed

---

## Step 1: Create an Okta Authorization Server

OpenChoreo validates JWTs using a specific issuer URL and JWKS endpoint. Okta provides these through a **Custom Authorization Server**.

1. In the Okta Admin Console, navigate to **Security → API**.
2. Click **Add Authorization Server**.
3. Set:
   - **Name**: `openchoreo` (or any name that makes sense in your org)
   - **Audience**: `openchoreo` (this becomes the `aud` claim in issued tokens — note it down)
   - **Description**: optional
4. Click **Save**.
5. From the authorization server's **Settings** tab, note down the **Issuer URI**. It will look like:
   ```
   https://{yourOktaDomain}/oauth2/{authorizationServerId}
   ```

The OIDC discovery, JWKS, authorization, and token endpoints are derived from this issuer:

| Endpoint | URL |
|---|---|
| Well-known | `{issuer}/.well-known/openid-configuration` |
| JWKS | `{issuer}/v1/keys` |
| Authorization | `{issuer}/v1/authorize` |
| Token | `{issuer}/v1/token` |

### Add a Custom Scope for Service-to-Service Authentication

Okta requires an explicit scope in every `client_credentials` token request. OpenChoreo service accounts (Backstage backend, Observer uid-resolver) use this grant type to authenticate with the control plane API, so you need a scope they can request. Built-in Okta scopes such as `offline_access` are restricted to user-facing flows and are rejected for `client_credentials`.

1. In the authorization server, click the **Scopes** tab.
2. Click **Add Scope**.
3. Set:
   - **Name**: `openchoreo_api` (or any name meaningful to your org — note it down)
   - **Display phrase** and **Description**: optional
4. Click **Create**.

Ensure the authorization server's access policy has a rule that grants this scope for the `client_credentials` grant type. In the **Access Policies** tab, open the relevant rule and confirm `openchoreo_api` is listed under allowed scopes.


---

## Step 2: Configure Access Token Claims

Okta's custom authorization server does not automatically include user profile claims or group membership in access tokens — each claim must be explicitly added. OpenChoreo components read the following claims from access tokens:

| Claim | Used by | Purpose |
|---|---|---|
| `email` | Backstage | Display the signed-in user's email |
| `given_name` | Backstage | Display the user's first name |
| `family_name` | Backstage | Display the user's last name |
| `groups` | Authorization engine | Map users to platform roles |

In the Okta Admin Console, go to **Security → API**, open your authorization server, and click the **Claims** tab. Add each of the following claims using **Add Claim**:

**email**
- **Name**: `email`
- **Include in token type**: **Access Token** → **Always**
- **Value type**: **Expression**
- **Value**: `user.email`

**given_name**
- **Name**: `given_name`
- **Include in token type**: **Access Token** → **Always**
- **Value type**: **Expression**
- **Value**: `user.firstName`

**family_name**
- **Name**: `family_name`
- **Include in token type**: **Access Token** → **Always**
- **Value type**: **Expression**
- **Value**: `user.lastName`

**groups**
- **Name**: `groups`
- **Include in token type**: **Access Token** → **Always**
- **Value type**: **Groups**
- **Filter**: **Matches regex** → `.*` (includes all groups; narrow this with a more specific regex if needed)

> **Note:** If your OpenChoreo authorization policies use a different claim name for groups (e.g., `roles`), set the claim name here to match. You will also need to adjust the [Subject Types configuration](https://openchoreo.dev/docs/platform-engineer-guide/authorization#subject-types) accordingly.

---

## Step 3: Create OAuth Applications

### 3.1 Backstage (Developer Portal)

Backstage uses two grant types: `authorization_code` for interactive user logins and `client_credentials` for backend service authentication.

1. In the Okta Admin Console, go to **Applications → Applications**.
2. Click **Create App Integration**.
3. Select **OIDC - OpenID Connect** and **Web Application**, then click **Next**.
4. Configure:
   - **App integration name**: `OpenChoreo Backstage`
   - **Grant type**: enable both **Authorization Code** and **Client Credentials**
   - **Sign-in redirect URIs**: `https://<backstage-domain>/api/auth/openchoreo-auth/handler/frame`
   - **Sign-out redirect URIs**: `https://<backstage-domain>` (optional)
   - **Controlled access**: assign to the groups or users who should have access to the developer portal
5. Click **Save**.
6. Note down the **Client ID** and **Client Secret** from the application's General tab.

**Token format:** Okta issues opaque tokens by default for web applications. To ensure Backstage receives JWT access tokens:

1. Open the application, go to the **General** tab and scroll to **Login** section.
2. Under **Login initiated by**: confirm settings are correct.
3. Navigate to the application's **Sign On** tab.
4. In the **OpenID Connect ID Token** section, confirm **Issuer** points to your custom authorization server.

**Add the application to the authorization server:**

1. Go to **Security → API**, open your authorization server.
2. Click the **Access Policies** tab.
3. Add a policy (or use the default policy) that applies to the Backstage application.
4. Add a rule granting `authorization_code` and `client_credentials` to the application.

---

### 3.2 CLI (`occ`)

The CLI uses `authorization_code` with PKCE — it is a public client and does not use a client secret.

1. In the Okta Admin Console, go to **Applications → Applications**.
2. Click **Create App Integration**.
3. Select **OIDC - OpenID Connect** and **Native Application**, then click **Next**.
4. Configure:
   - **App integration name**: `OpenChoreo CLI`
   - **Grant type**: enable **Authorization Code** and **Refresh Token**
   - **Sign-in redirect URIs**: `http://127.0.0.1:55152/auth-callback`
   - **PKCE enforcement**: select **Require PKCE as additional verification**
   - **Controlled access**: assign to the groups or users who should have CLI access
5. Click **Save**.
6. Note down the **Client ID**. There is no client secret for a native/public application.

---

### 3.3 Workload Publisher (CI Workflows)

CI/CD workflows authenticate using `client_credentials` with a client secret.

1. In the Okta Admin Console, go to **Applications → Applications**.
2. Click **Create App Integration**.
3. Select **API Services**, then click **Next**.
4. Configure:
   - **App integration name**: `OpenChoreo Workload Publisher`
5. Click **Save**.
6. Note down the **Client ID** and **Client Secret**.

---

### 3.4 Observer (uid-resolver)

The uid-resolver is a sub-component of the Observer service that resolves component, project, and environment identities by calling the OpenChoreo control plane API. It authenticates using `client_credentials`.

1. In the Okta Admin Console, go to **Applications → Applications**.
2. Click **Create App Integration**.
3. Select **API Services**, then click **Next**.
4. Configure:
   - **App integration name**: `OpenChoreo Observer Resource Reader`
5. Click **Save**.
6. Note down the **Client ID** and **Client Secret**.

> Okta's Custom Authorization Server does not require a specific scope for `client_credentials` grants by default. Leave the scope empty unless your authorization server policy explicitly requires one.

---

### 3.5 SRE Agent (Optional)

The SRE Agent performs AI-powered root cause analysis. It authenticates with the control plane using `client_credentials`. This section is only required if you plan to use the SRE Agent.

1. In the Okta Admin Console, go to **Applications → Applications**.
2. Click **Create App Integration**.
3. Select **API Services**, then click **Next**.
4. Configure:
   - **App integration name**: `OpenChoreo SRE Agent`
5. Click **Save**.
6. Note down the **Client ID** and **Client Secret**.

---

## Step 4: Configure the Control Plane

Add the following to your control plane `values.yaml`. Replace every `<placeholder>` with your actual values.

**OIDC and JWT settings** — point these at your Okta custom authorization server. Replace `{yourOktaDomain}` (e.g. `dev-1234567.okta.com`) and `{authorizationServerId}` (e.g. `aus1ab2cd3ef4gh5ij`) throughout:

```yaml
security:
  oidc:
    issuer: "https://{yourOktaDomain}/oauth2/{authorizationServerId}"
    wellKnownEndpoint: "https://{yourOktaDomain}/oauth2/{authorizationServerId}/.well-known/openid-configuration"
    jwksUrl: "https://{yourOktaDomain}/oauth2/{authorizationServerId}/v1/keys"
    authorizationUrl: "https://{yourOktaDomain}/oauth2/{authorizationServerId}/v1/authorize"
    tokenUrl: "https://{yourOktaDomain}/oauth2/{authorizationServerId}/v1/token"
  jwt:
    audience: "openchoreo"
```

The `security.jwt.audience` value must match the **Audience** configured on the Okta authorization server in Step 1.

**CLI client** — register the CLI's public client ID so the control plane exposes it to the `occ login` flow:

```yaml
security:
  oidc:
    externalClients:
      - name: cli
        client_id: "<okta-cli-client-id>"
        scopes:
          - openid
          - profile
          - email
```

**Backstage** — add the client ID, redirect URL, and scopes. The client secret is read from the `backstage-secrets` Kubernetes secret (see [Backstage Configuration](https://openchoreo.dev/docs/platform-engineer-guide/backstage-configuration#authentication) for how to create it):

```yaml
backstage:
  secretName: "backstage-secrets"
  auth:
    clientId: "<okta-backstage-client-id>"
    redirectUrls:
      - "https://<backstage-domain>/api/auth/openchoreo-auth/handler/frame"
    oidcScope: "openid profile email"
    # Okta requires an explicit scope for client_credentials token requests.
    # Set this to the custom scope created in Step 1.
    scope: "openchoreo_api"
```

**Bootstrap role bindings** — Backstage and the Observer uid-resolver both authenticate via `client_credentials`; the `sub` claim in their JWTs is the client ID. These bindings grant them the roles they need to call the control plane API:

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
```

> **Note:** The `bootstrap.mappings` array is replaced in full when overridden. The snippet above shows only the two service account bindings that change when moving from Thunder to Okta. Your values file must also carry forward all the other default bindings from the chart (`admin-binding`, `developer-binding`, `platform-engineer-binding`, `sre-binding`, `workload-publisher-binding`, `mcp-tryout-client-binding`), updating each service account `value` field to the matching Okta client ID. See the [Authorization Configuration](https://openchoreo.dev/docs/platform-engineer-guide/authorization#customizing-bootstrap-roles-and-bindings) guide for details.

**SRE Agent bootstrap binding (optional)** — only add this if you created the SRE Agent application in Step 3.5:

```yaml
openchoreoApi:
  config:
    security:
      authorization:
        bootstrap:
          mappings:
            - name: rca-agent-binding
              kind: ClusterAuthzRoleBinding
              system: true
              roleMappings:
                - roleRef:
                    name: rca-agent
                    kind: ClusterAuthzRole
              entitlement:
                claim: sub
                value: "<okta-sre-agent-client-id>"
              effect: allow
```

---

## Step 5: Configure the Observability Plane

Add the following to your observability plane `values.yaml`.

**OIDC and JWT settings** — used to validate tokens on the Observer API and to obtain tokens for the uid-resolver:

```yaml
security:
  oidc:
    issuer: "https://{yourOktaDomain}/oauth2/{authorizationServerId}"
    jwksUrl: "https://{yourOktaDomain}/oauth2/{authorizationServerId}/v1/keys"
    tokenUrl: "https://{yourOktaDomain}/oauth2/{authorizationServerId}/v1/token"
  jwt:
    audience: "openchoreo"
```

**uid-resolver OAuth client** — the Observer uses these credentials to fetch tokens from Okta and call the control plane API. The client secret must be stored in a Kubernetes secret under the key `UID_RESOLVER_OAUTH_CLIENT_SECRET`; refer to the [Secret Management](https://openchoreo.dev/docs/platform-engineer-guide/secret-management) guide for how to provision it:

```yaml
observer:
  oauthClientId: "<okta-observer-client-id>"
  secretName: "observer-secret"
  # Okta requires an explicit scope for client_credentials token requests.
  # Set this to the same custom scope created in Step 1.
  oauthScope: "openchoreo_api"
```

**SRE Agent OAuth client (optional)** — only required if you created the SRE Agent application in Step 3.5. Store the client secret in OpenBao under `secret/rca-oauth-client-secret` (see the [SRE Agent Configuration](https://openchoreo.dev/docs/ai/sre-agent#authentication-and-authorization) guide for secret provisioning steps), then add to your observability plane values:

```yaml
rca:
  secretName: "rca-agent-secret"
  oauth:
    clientId: "<okta-sre-agent-client-id>"
```

---

## Step 6: Verify the Configuration

**1. Check control plane pods are healthy:**

```bash
kubectl get pods -n openchoreo-control-plane
```

**2. Check for authentication configuration errors in the API logs:**

```bash
kubectl logs -l app.kubernetes.io/name=openchoreo-api -n openchoreo-control-plane --tail=50
```

Look for lines that confirm the JWKS endpoint was reached and keys were loaded successfully.

**3. Test the CLI login:**

```bash
occ login
```

This should open a browser window redirecting to your Okta login page. After signing in, you should be returned to the terminal with a success message.

**4. Test the Backstage portal:**

Navigate to your Backstage URL. You should be redirected to Okta for login. After authenticating you should land on the OpenChoreo developer portal.

---

## Troubleshooting

**Token validation fails with "invalid issuer":**
- The `security.oidc.issuer` value must exactly match the `iss` claim in Okta-issued JWTs.
- Verify by decoding a token at [jwt.io](https://jwt.io) and comparing the `iss` field to your configured issuer.

**User profile claims (email, given_name, family_name) or groups are missing from the token:**
- Confirm each claim was added to the **Access Token** (not just the ID Token) in the authorization server claims configuration (Step 2).
- Decode an access token at [jwt.io](https://jwt.io) and verify the expected fields are present.

**Audience validation fails:**
- The `security.jwt.audience` value must match the **Audience** field set on the Okta authorization server (Step 1).
- If you left the Okta audience as the default (`api://default`), update the Helm value to match.

**Backstage login redirects to a blank page or errors:**
- Confirm the redirect URI registered in Okta exactly matches `https://<backstage-domain>/api/auth/openchoreo-auth/handler/frame` — Okta performs exact string matching including protocol and trailing slashes.

**CLI login hangs or callback fails:**
- Confirm the redirect URI `http://127.0.0.1:55152/auth-callback` is registered in the Okta CLI application.
- Confirm the application type is **Native Application** with PKCE enabled.

---

## Next Steps

- [Configure Authorization](https://openchoreo.dev/docs/platform-engineer-guide/authorization) to define roles and role bindings that reference your Okta groups
- [Set up Secret Management](https://openchoreo.dev/docs/platform-engineer-guide/secret-management) to store client secrets in Kubernetes secrets rather than Helm values
- Review the [Control Plane Helm reference](https://openchoreo.dev/docs/reference/helm/control-plane) for all available `security.*` configuration options