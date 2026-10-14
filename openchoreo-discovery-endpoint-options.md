# Discovery endpoint for OpenChoreo features: research and options

**Question.** Is there a standard pattern for a discovery endpoint that tells clients which OpenChoreo
features exist and how to reach them? The first concrete requirement: advertise the `observerURL`
of the observability plane that serves the audit trail.

**Short answer.** No single standard covers "platform feature discovery". The common approach is a
**well-known or versioned JSON metadata document served by the main API**, modelled on OIDC/OAuth
discovery. The one IETF standard aimed at API discovery, RFC 9727 `api-catalog`, advertises links
only. It cannot say whether a feature is enabled or which version it runs. Recommendation: add an
OpenChoreo capabilities document to `openchoreo-api` (Option 1), readable by any authenticated
caller (§5).

---

## 1. Why this is needed now

Clients already work out the observer URL on their own, and they don't all do it the same way:

| Where | How it resolves the observer | Scope |
|---|---|---|
| `internal/occ/cmd/component/logs.go:321` | Environment → DataPlane → `observabilityPlaneRef`, falling back to ObservabilityPlane `default` | per environment |
| `backstage-plugins/packages/openchoreo-client-node/src/observability-url-resolver.ts:156` (`resolveForPlatform`, branch `feat/audit-logs`) | ClusterObservabilityPlane `default`, then ObservabilityPlane `default` in namespace `default` | platform-wide (audit) |
| `openchoreo.observability.auditLogs.observerUrl` (Backstage app-config, `ObservabilityService.ts:229`) | Manual override. It exists because the `default` naming guess doesn't hold everywhere | platform-wide (audit) |

Audit logs have no Environment to resolve through; the comment at `observability-url-resolver.ts:148`
says so. That leaves each client to pick a guess or add an override flag. Two clients reimplementing
a naming convention, plus an escape hatch, means the server should publish this as a contract.

Two decisions in `openchoreo-audit-delivery-plan.md` §1.6 affect the design:

- **D-A:** telemetry targeting is chart-level configuration, not a CRD field.
- **D-B:** audit is scoped to the control plane, so each installation has exactly one audit
  destination, and it is set by a chart value.

So the server can publish **one** audit observer URL per installation. It can come from chart
config, matching D-A, instead of a CRD naming convention.

## 2. What exists today in OpenChoreo

`openchoreo-api` already serves unauthenticated bootstrap endpoints (`openapi/openchoreo-api.yaml`):

- `GET /version`: build info only.
- `GET /openapi.json`: the spec.
- `GET /.well-known/oauth-protected-resource`: RFC 9728 metadata, with the OpenChoreo extension
  fields `openchoreo_clients` and `openchoreo_security_enabled`
  (`internal/openchoreo-api/api/handlers/oauth_metadata.go`). `occ login` and `occ remote` read it.

The observer serves the same RFC 9728 document (`internal/observer/api/handlers/oauth_metadata.go`).
`internal/openchoreo-api/config` has no observer reference today.

## 3. Standards and prior art

| Pattern | What it is | Fit for "advertise features and URLs" |
|---|---|---|
| **RFC 8615** `/.well-known/` | Reserved URI prefix for site-wide metadata; names are registered with IANA | The namespace, not a format. A private name like `openchoreo-configuration` works but isn't registered |
| **OIDC Discovery / RFC 8414** | JSON document of endpoints and supported features at a well-known path. Extensible, with optional `signed_metadata` | The model most "discovery" endpoints copy: flat JSON, endpoint URLs plus `*_supported` lists. Scoped to authorization servers |
| **RFC 9728** protected resource metadata | OAuth resource metadata. §2 allows extra parameters | Already in use here, but the RFC scopes it to OAuth: "information needed to interact with an OAuth 2.0 protected resource". Not general service discovery |
| **RFC 9727** `/.well-known/api-catalog` (June 2025, Standards Track) | `application/linkset+json` listing APIs with `item`, `service-desc` and `service-doc` relations | Standard and a good match for "here are our APIs and their specs". Carries links, not state: no enabled flag, no version, no per-feature settings |
| **Kubernetes discovery** (`/api`, `/apis`, aggregated v2) | Lists group-versions and resources so kubectl can adapt | Right idea (capability negotiation), but tied to the k8s resource model |
| **Argo CD** `GET /api/v1/settings` | Unauthenticated settings for the UI and CLI before login (SSO config, URL, feature flags) | Closest product precedent. Also a warning: it leaked OIDC ClientID and other fields to anonymous callers, patched in v2.11.3 / v2.10.12 / v2.9.17 |
| **GitLab** `GET /api/v4/metadata` | Instance version and feature info, replacing `/version` | A versioned API path, not well-known |
| **Harbor** `GET /api/v2.0/systeminfo` | System info and capabilities; returns less when unauthenticated | Precedent for **tiered disclosure** by auth state |
| **GitHub** `GET /meta` | Infrastructure metadata (domains, IP ranges, keys) | Precedent for a single stable metadata document |

In practice, products use either an OIDC-style well-known JSON document or a versioned
`/api/.../settings` or `/metadata` endpoint. Both are plain JSON the product defines, and both
often return fewer fields to unauthenticated callers.

## 4. Options

### Option 1 (recommended): OpenChoreo capabilities document on `openchoreo-api`

A JSON document listing features and how to reach them, defined in the OpenAPI spec like any other
operation.

```http
GET /api/v1alpha1/metadata     # authenticated (bearerAuth), no specific permission
```

```json
{
  "features": {
    "auditLogs": {
      "enabled": true,
      "observerURL": "https://observer.example.com"
    }
  }
}
```

`enabled` means clients can query audit logs, not that audit events are emitted. `observerURL` is
omitted when `enabled` is false. Clients ignore fields they don't recognize.

- **Where the value comes from (decided):** the existing audit config names the observability
  plane, as `openchoreoApi.config.audit.observabilityPlaneRef.{kind, name, namespace}` in the
  control-plane chart and `audit.observability_plane_ref` in the API config. The API server reads
  that ObservabilityPlane or ClusterObservabilityPlane with its own client, since the endpoint
  requires no permission, and advertises its `spec.observerURL`.
  - `enabled` is true only when a reference is set, the plane exists, and it has an observer URL.
    It is independent of `audit.enabled`, which controls emission from this API server.
  - A reference to a missing plane is logged and reported as `enabled: false`; other lookup
    errors return 500.
  - `namespace` is required for ObservabilityPlane and rejected for ClusterObservabilityPlane at
    startup.
- **Result:** occ and Backstage read one field. `resolveForPlatform`'s `default` chain and the
  `auditLogs.observerUrl` override can be removed.
- **Extensible:** each new feature (RCA agent, FinOps, remote-connect) adds a block, not an
  endpoint. Clients ignore keys they don't recognise, as with OIDC metadata.
- **Path choice:** `/api/v1alpha1/metadata`, under the versioned API, not `/.well-known/`. A well-known URI signals a document
  readable before login, and this one requires authentication (§5). Under `/api/v1` it gets the
  API's global `bearerAuth` security and its middleware without a special case.
- **Cost:** one new operation, a small config block, and client updates.

### Option 2: RFC 9727 `/.well-known/api-catalog`

Publish a linkset that lists the openchoreo-api and observer APIs with `service-desc` links to
their OpenAPI documents.

- **Pro:** an actual IETF standard, and generic tooling can use it.
- **Con:** it only says where things are. "Is audit enabled?" and per-feature settings would need
  a second document or non-standard link attributes. Useful **alongside** Option 1, not instead
  of it.

### Option 3: No new endpoint; make the reference explicit

Put the audit observer reference somewhere clients can already read. For example, a platform-level
reference to an observability plane, or an annotation or label on the ObservabilityPlane or
ClusterObservabilityPlane that serves audit.

- **Pro:** smallest change, no new API surface.
- **Con:** each client still resolves it itself, and needs view permission on plane resources.
  A CRD field works against D-A. The lookup still isn't a published contract.

### Rejected: adding `openchoreo_observer_url` to `/.well-known/oauth-protected-resource`

This is the easiest path: the `openchoreo_clients` extension sets a precedent, the endpoint is
already public, and occ and Backstage already fetch it. It mixes concerns, though. RFC 9728 is
metadata for OAuth protected resources, not service discovery. Each feature added would push that
document further from its purpose, and generic MCP/OAuth clients would read platform
topology they have no use for.

## 5. Decision: authenticated only

**Any authenticated caller can read the document. It is not public, and it needs no specific
permission.**

The document describes how the installation is configured, so clients can adapt: show or hide
the Audit Logs page, and know which observer to call. It grants no access. Reading audit records
still requires `auditlogs:view` on the observer (`openapi/observer-api.yaml:552`).

Why not public: `/version` and `/.well-known/oauth-protected-resource` use `security: []`, but
this document reveals platform layout, such as internal hostnames and which features are enabled.
Argo CD's anonymous `/api/v1/settings` leak (GHSA-87p9-x75h-p4j2) is the failure to avoid.

What follows from this decision:

- **Path is `/api/v1alpha1/metadata`, not under `/.well-known/`.** It inherits the spec's global `bearerAuth`;
  the operation adds no `security: []` override.
- **Clients read it after login.** occ already fetches the OAuth metadata, then logs in, then calls
  the API. Backstage calls the API with the user's token. Neither needs the document before auth.
- **The same response for every caller.** No per-caller or per-permission shaping, so the response
  can be cached and doesn't need an authz evaluation. Whether a user may *use* a feature stays with
  the existing checks (`useAuditLogsPermission` in the UI, `auditlogs:view` on the observer).
- **Content rule for new fields:** configuration a signed-in user may know. No secrets, no
  credentials, no values that only administrators should see.

Considered and not taken: a tiered response (Harbor model: fewer fields anonymously) and
permission-shaped blocks (the `auditLogs` block shown only to `auditlogs:view` holders). Both add
complexity without a client that needs them.

## 6. Sources

- RFC 8615, Well-Known URIs: https://www.rfc-editor.org/rfc/rfc8615.html
- RFC 8414, OAuth 2.0 Authorization Server Metadata: https://www.rfc-editor.org/rfc/rfc8414.html
- RFC 9728, OAuth 2.0 Protected Resource Metadata: https://www.rfc-editor.org/rfc/rfc9728.html
- RFC 9727, api-catalog: https://www.rfc-editor.org/rfc/rfc9727.html
- Kubernetes API discovery: https://kubernetes.io/docs/concepts/overview/kubernetes-api/
- Argo CD unauthenticated settings advisory: https://github.com/argoproj/argo-cd/security/advisories/GHSA-87p9-x75h-p4j2
- Argo CD issue on anonymous API data: https://github.com/argoproj/argo-cd/issues/22809
- Harbor API spec (`/systeminfo`): https://github.com/goharbor/harbor/blob/main/api/v2.0/swagger.yaml
- GitLab metadata API: https://agile.bu.edu/gitlab/help/api/version.md
- GitHub meta API: https://docs.github.com/en/rest/meta/meta
