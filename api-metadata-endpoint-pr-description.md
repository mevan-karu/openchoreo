## Purpose

Clients have no way to ask OpenChoreo how it is configured, so each one guesses. The clearest case is the audit trail. Component logs resolve their observer through Environment, DataPlane and `observabilityPlaneRef`, but audit logs belong to the control plane and have no environment to resolve through. occ has no audit logs support yet. The in-progress Backstage Audit Logs page tries a ClusterObservabilityPlane named `default`, then an ObservabilityPlane `default` in namespace `default`, and adds an app-config override (`openchoreo.observability.auditLogs.observerUrl`) for installs where that naming guess is wrong.

This adds `GET /api/v1alpha1/metadata`, a document describing how the installation is configured. Its first entry tells clients whether audit logging is enabled and which observer serves the audit logs:

```json
{"features":{"auditLogs":{"enabled":true,"observerURL":"https://observer.example.com"}}}
```

It also makes audit logging opt-in. The observability plane that stores audit logs is optional, so audit is disabled by default, and enabling it requires naming that plane.

## Approach

The platform engineer enables audit and names the observability plane in the existing audit config:

```yaml
openchoreoApi:
  config:
    audit:
      enabled: true                       # default false
      observabilityPlaneRef:
        kind: ClusterObservabilityPlane   # or ObservabilityPlane
        name: default
        namespace: ""                     # for an ObservabilityPlane
```

`enabled` in the response comes straight from `audit.enabled`. When it is true, a new `services/metadata` reads the referenced plane with `controller.GetObservabilityPlaneFromRef` and returns its `spec.observerURL`. A referenced plane that does not exist is a misconfiguration, since audit is off unless an operator enables it and names a plane. It is logged as a warning and the response carries `enabled: true` without `observerURL`, instead of failing. Any other lookup error returns 500.

The endpoint requires authentication but no permission. It inherits the spec's global `bearerAuth`, so the JWT middleware runs on it like any other authenticated route. It describes configuration and grants nothing: querying the trail still needs `auditlogs:view` on the observer. For that reason the service has no authz wrapper and reads the plane with the API server's own client. A user without `observabilityplane:view` still gets the URL, which is the point. It is not public, because a document saying "audit is enabled, the observer is at this host" is platform reconnaissance for an anonymous caller.

Observer does not need the reference, so it stays out of the shared `auditconfig.AuditConfig`. openchoreo-api's `AuditConfig`, previously an alias of the shared type, now embeds it and adds `ObservabilityPlaneRef`, which keeps the key under `audit` in the API config. The embedded field carries `koanf:",squash,flatten"`: `squash` for mapstructure decoding, `flatten` for fatih/structs, which the loader uses to turn `Defaults()` into a map. Without `flatten`, every config load fails with `'config.AuditConfig' has invalid keys: AuditConfig`.

Startup validation only checks that the reference is configured: `kind` and `name` are required when `audit.enabled` is true. It does not check the values or whether the plane exists in the cluster. A reference that does not match a plane is handled at request time like a missing plane, except a name the Kubernetes client refuses to put in a URL (one containing `/` or `%`), which returns 500.

Defaults and supporting changes:

- `audit.enabled` defaults to false for both openchoreo-api and observer: in the shared `auditconfig.AuditDefaults()`, openchoreo-api's shipped `config.yaml`, `openchoreoApi.config.audit.enabled` in the control plane chart, and `observer.audit.enabled` in the observability plane chart.
- The k3d and quick-start installs follow the chart defaults, so audit is off there until enabled.
- `GetMetadata` is listed in `RESTExemptions` as a read, which the audit coverage gate requires.
- The generated server, client and models, the occ client mock, and both charts' `values.schema.json` are regenerated.

## Related Issues

https://github.com/openchoreo/openchoreo/issues/4697
https://github.com/openchoreo/openchoreo/issues/4731

## Checklist

- [x] Tests added or updated (unit, integration, etc.)
- [ ] Samples updated (if applicable)
- [ ] Added `backport/<release-branch>` label if this should be backported (e.g., `backport/release-v1.0`)
- [x] This PR includes AI-generated code or content

## Remarks

This changes existing behaviour. An install that relied on the old default stops emitting audit events after upgrading, from both openchoreo-api and observer. Re-enabling it on openchoreo-api now also requires `observabilityPlaneRef`, or the API server fails validation at startup. This needs a line in the release notes and upgrade docs.

No client uses the endpoint yet. Replacing the Backstage Audit Logs page's `default`-plane guess and its `auditLogs.observerUrl` override is a follow-up in backstage-plugins, and occ can use the endpoint when it gains audit logs support. Clients should treat `enabled: true` without `observerURL` as audit logging being on with no observability plane to query.

When `security.enabled` is false the JWT middleware passes every request through, so this endpoint is readable without a token. That is true of every authenticated endpoint and is not new here.

The path is `v1alpha1` on purpose. The response always includes `features.auditLogs`, and clients are expected to ignore fields they do not recognise, so later features can be added without breaking existing readers.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
