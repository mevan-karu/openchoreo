# Audit log population run — 2026-09-22 (k3d-openchoreo)

Read back with:
`POST http://observer.openchoreo.localhost:11080/api/v1alpha1/audit-logs/query`
or `occ auditlogs --since 2h` (admin credential), or the Backstage **Audit Logs** page.

**28 records**, window 15:57–16:23 UTC. Results: 24 success, 3 denied, 1 failure.
11 distinct actions, all in category `management` — nothing in `authorization` or `access`,
including the three denials, so `--category` does not separate this dataset.
Cluster left clean — every resource created was deleted.

## How each leg is identifiable

| Leg | surface | actor.type | actor.id | entitlement | user_agent | records |
|---|---|---|---|---|---|---|
| Backstage UI (admin) | rest | user | admin@openchoreo.dev | groups: admins | `node` | 8 |
| CLI `occ` 1.3.0 (admin, developer) | rest | user | admin@ / developer@ | groups: admins / developers | `Go-http-client/1.1` | 10 |
| MCP endpoint (service account) | mcp | service_account | `unknown` | client_id: service_mcp_client | `Python-urllib/3.14` | 9 |
| REST direct (service account) | rest | service_account | `unknown` | client_id: service_mcp_client | `curl/8.7.1` | 1 |

## Operations

### Backstage UI — admin@openchoreo.dev (Playwright-driven)
1. create_project `audit-ui-proj` (scaffolder template "Default Project Type")
2-4. create_project_release_binding × 3 (development, staging, production) — auto-deploy side effect
5. create_component `audit-ui-svc` (Service template, image nginx:1.27)
6. create_workload `audit-ui-svc-workload` — side effect of 5
7. delete_component `audit-ui-svc` (project page → Delete → confirm)
8. delete_project `audit-ui-proj` (catalog → Delete → confirm)

### CLI (occ) — admin@openchoreo.dev
9. create_project `audit-cli-proj` (`occ apply -f`)
10. create_project_release_binding `audit-cli-proj-development`
11. create_component `audit-cli-svc` (`occ component scaffold` → `occ apply`)
12. create_environment `audit-cli-env`
13. update_environment `audit-cli-env` (re-apply with changed description)
14. delete_environment `audit-cli-env`
15. delete_project `audit-cli-proj`

### CLI (occ) — developer@openchoreo.dev → denied
16. create_namespace `audit-dev-attempt` → **denied**
17. delete_project `audit-ui-proj` → **denied**
18. update_environment `audit-cli-env` → **denied**
- `occ auditlogs` was also refused ("needs the cluster-scoped auditlogs:view permission") but produced **no audit record** — the observer rejects before emitting.

### MCP endpoint — service_mcp_client (JSON-RPC over http://api.openchoreo.localhost:8080/mcp)
19. create_project `audit-mcp-proj`
20. create_component `audit-mcp-svc`
21. create_environment `audit-mcp-env`
22. update_project `audit-mcp-proj`
23. delete_component `audit-mcp-svc`
24. delete_environment (bad arguments) → **failure**
25. delete_project `audit-mcp-proj`
26. delete_project `audit-demo-svc`
27. delete_environment `audit-mcp-env`

### REST direct — service_mcp_client
28. create_project `audit-demo-svc` (path-proving call, event 01a0c9d5-dfb7-7123-8555-aea09e590e48)

## Observations worth checking

- **Service-account records carry `actor.id: "unknown"`.** `audit.actor.id_claim` is `username`
  (openchoreo-api-config), and a client_credentials token from Thunder has no `username` claim.
  The account is still identifiable via `entitlements.client_id`, but `--actor` filtering
  cannot reach it, and the UI prints "unknown".
- **`occ` sends `Go-http-client/1.1` as its user agent** — no product/version string, so CLI
  traffic is not self-identifying in the trail the way `node` (Backstage) is.
- **A denied audit-log read is not itself audited.** Reading the trail without `auditlogs:view`
  leaves no record of the attempt.
- The observability plane has `opentelemetry-collector` and `prometheus-operator` in
  CrashLoopBackOff; the audit path does not depend on them and was unaffected.
