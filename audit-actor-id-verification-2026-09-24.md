# Actor-ID fix verification — 2026-09-24 (k3d-openchoreo, server build 1aeed898)

Re-ran the 2026-09-22 exercise on the rebuilt cluster. **35 audit records, zero `unknown` actors.**
Config is unchanged (`audit.actor.id_claim: username`), so the fix is server-side.

## Result: fixed

| surface | actor.type | actor.id | user_agent | records |
|---|---|---|---|---|
| rest | user | admin@openchoreo.dev | `node` (Backstage) | 8 |
| rest | user | admin@openchoreo.dev | `Go-http-client/1.1` (occ) | 8 |
| rest | user | developer@openchoreo.dev | `Go-http-client/1.1` (occ) | 3 (all denied) |
| mcp | user | admin@openchoreo.dev | `Python-urllib/3.14` | 7 |
| mcp | service_account | **service_mcp_client** | `Python-urllib/3.14` | 7 |
| rest | service_account | **service_mcp_client** | `Python-urllib/3.14` | 2 |

Before the fix, the last two rows carried `actor.id: "unknown"`. They now carry the client ID.
32 success, 3 denied, plus one deliberate `failure` (bad-argument MCP `delete_environment`) added
afterwards to cover that path — it too carries `actor.id: service_mcp_client`.

The **portal** now renders `service_mcp_client` with a "service account" subtitle in the Actor
column of `/audit-logs`, where it previously printed "unknown". That was a separate rendering path
from the API response, so it is confirmed independently.

Scope note: only one service-account identity was exercised (`service_mcp_client`) — Thunder's
bootstrap registers others (`rca-agent-client`, `workload-publisher`, `finops-agent-client`) that
were not tested, as credentials were only available for the MCP one.

### Filtering now reaches service accounts

Against the observer (`POST /api/v1alpha1/audit-logs/query`), counts are exact:

| filter | records |
|---|---|
| `actor.id: [service_mcp_client]` | 9 |
| `actor.type: [service_account]` | 9 |
| `actor.type: [user]` | 26 |
| `actor.id: [admin@openchoreo.dev]` | 23 |
| `actor.id: [developer@openchoreo.dev]` | 3 |
| `result: [denied]` | 3 |
| `surface: [mcp]` | 14 |

`occ auditlogs --actor service_mcp_client` also returns the service-account records.

## Operations exercised (all three surfaces, then cleaned up)

- **Backstage UI** (admin): create project + service component via scaffolder, delete both via UI dialogs.
- **CLI `occ`** (admin): apply project + release binding, scaffold/apply component, create then update
  environment, delete component/environment/project.
- **CLI `occ`** (developer): create namespace, delete admin's project, update environment — all **denied**;
  `occ auditlogs` refused for lacking `auditlogs:view`.
- **MCP** (service account *and* admin user, same 7-op batch each): create/update/delete of
  project, component, environment.
- **REST** (service account): create + delete project.

Cluster left clean — no `audit2-*` resources remain. Note the local `default` occ context now points
at credential `admin2` (it pointed at `default` before this run), and `dev2` holds the developer login.

## Still open

- **`occ auditlogs` silently applies the context's project scope.** With the default context
  (`project: default`), a bare `occ auditlogs` and even `occ auditlogs --actor <x>` return
  "No audit records matched the query", because every record here belongs to some other project.
  Adding `--project audit2-mcp-svc` makes the same query return rows. Setting `--project ""` does not
  clear it. The command says nothing about the implicit filter, so the trail looks empty rather than
  narrowed — worth either dropping the context project from audit queries or naming the applied scope
  in the output. Whether this is pre-existing or new in this build is **not established**: no bare
  `occ auditlogs` was run as an authenticated admin on the previous cluster.
- **`occ` still sends `Go-http-client/1.1`** as its user agent, so CLI traffic is not self-identifying
  in the trail the way Backstage (`node`) is.
- **A denied audit-log read is still not itself audited** — the developer's refused `occ auditlogs`
  left no record.
- The MCP leg ran against the `/mcp` endpoint directly (JSON-RPC); the `openchoreo-cp` MCP client
  still does not register its tools in the Claude Code session, so the user-actor MCP records were
  produced with the token `occ` stored for the admin credential.
