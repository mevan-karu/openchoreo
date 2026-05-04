# OAuth2 Scope Configuration

The `openchoreo-observability-plane` Helm chart supports an optional OAuth2
scope parameter for the **Observer** and **RCA Agent** components. When your
identity provider requires an explicit scope in client credentials token
requests, you can supply it using the values below.

Both fields default to an empty string, which omits the `scope` parameter from
the token request entirely.

---

## Observer

```yaml
observer:
  oauthScope: "api:read"
```

## RCA Agent

```yaml
rca:
  oauth:
    scope: "api:read"
```

Or via `--set`:

```bash
helm upgrade --install openchoreo-observability-plane \
  oci://ghcr.io/openchoreo/helm-charts/openchoreo-observability-plane \
  --version 1.0.1-hotfix.1 \
  --namespace openchoreo-observability-plane \
  --reuse-values \
  --set observer.oauthScope="api:read" \
  --set rca.oauth.scope="api:read"
```

---

## Notes

- For multiple scopes use a space-delimited value: `"openid api:read api:write"`.
- When left empty (the default), the identity provider's default scope applies.
