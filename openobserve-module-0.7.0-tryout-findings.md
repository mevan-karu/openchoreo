# observability-logs-openobserve 0.7.0 — tryout findings

Tried on 2026-09-18 by following the module README at tag
[`observability-logs-openobserve-0.7.0`](https://github.com/openchoreo/community-modules/tree/observability-logs-openobserve-0.7.0/observability-logs-openobserve).

Setup: single-cluster k3d, OpenChoreo `1.3.0-rc.1`. I replaced `observability-logs-opensearch 0.6.0` with
`observability-logs-openobserve 0.7.0` (Fluent Bit + audit logs enabled). I also removed
`observability-tracing-opensearch 0.6.0` and repointed `observability-events-otel-collector 0.1.2` to OpenObserve.

## Summary

| # | Area | Severity | Finding |
|---|------|----------|---------|
| 1 | Chart | **Critical** | The audit `rewrite_tag` rule is misparsed. Audit records are discarded, and most openchoreo-api/observer container logs are dropped too |
| 2 | Chart | High | Fluent Bit starts before OpenObserve is ready, and the log backlog it reads at startup is dropped |
| 3 | Docs | High | The base install command fails as written because `clusterInstance` is required |
| 4 | Docs | High | There's no guide for moving from OpenSearch. The events collector and tracing silently break |
| 5 | Docs | Medium | Every command pins `0.0.0-latest-dev`, which contradicts the compatibility table |
| 6 | Docs | Medium | The prerequisite `observer.logsAdapter.enabled` doesn't exist in the 1.3.0 chart |
| 7 | Docs | Medium | The credentials recipe doesn't match what the OpenChoreo install already seeds in OpenBao |
| 8 | Chart | Medium | No retention setting for container logs or events (OpenSearch module defaults to 30d) |
| 9 | Chart | Low | The adapter exits and crash-loops until OpenObserve is reachable |
| 10 | API | Low | The multi-cluster audit docs promise cluster attribution, but the audit read API doesn't return it |

## 1. The audit rewrite rule is misparsed (critical)

`helm/templates/fluent-bit/config.yaml:63`:

```
Rule          $log "msg":"AUDIT-LOG" audit.{{ $p.producer }} false
```

Fluent Bit (4.2.3, the image this chart deploys) splits `Rule` on quotes. `Rule` is declared as
`FLB_CONFIG_MAP_SLIST_4` (`plugins/filter_rewrite_tag/rewrite_tag.c:591`), and the tokenizer (`token_retrieve` in
`src/flb_slist.c:107-180`) treats a token that starts with `"` as quoted and ends it at the next `"`. The rule
therefore becomes five tokens:

```
[0] $log   [1] msg   [2] :"AUDIT-LOG"   [3] audit.<producer>   [4] false
```

`process_config` (`rewrite_tag.c:138-178`) reads tokens 0 to 3 as key, regex, tag and keep:

- regex = `msg`
- tag = `:"AUDIT-LOG"`
- keep = `audit.<producer>`, which evaluates to false
- the real `false` is ignored

Because keep still comes out false, nothing looks wrong.

Consequences seen on the cluster:

- Audit records get the tag `:"AUDIT-LOG"`. That tag matches no `audit.*` filter or output, so the records are silently discarded.
  Fluent Bit metrics: `audit_emitter_openchoreo-api` 198 and `audit_emitter_observer` 228 emitted, but
  `parser.4 records=0` and `http.1 proc_records=0`. The observer audit query returns `total: 0` even though
  `create_project` and `delete_project` were written by openchoreo-api.
- The regex `msg` matches almost every structured log line from `api-server` and `observer`. Because `keep=false`,
  those lines are removed from the container-logs stream too (`rewrite_tag.2 drop_records=198` of 235). So
  turning on audit logs **takes openchoreo-api and observer out of platform logs.**
- No error or warning is logged anywhere.

Side-by-side run with `fluent-bit:4.2.3`, where each rule got the same audit line and the same normal line:

| Rule | Audit line tagged as | Normal line |
|---|---|---|
| This module: `$log "msg":"AUDIT-LOG" audit.x false` | `:"AUDIT-LOG"` (lost) | re-tagged `:"AUDIT-LOG"` (lost) |
| OpenSearch module: `$log AUDIT-LOG(?!-) audit.x false` | `audit.x` | unchanged |
| Escaped quotes: `$log \"msg\":\"AUDIT-LOG\" audit.x false` | `audit.x` | unchanged |

**Fix:** use the same rule as the OpenSearch module:

```
Rule          $log AUDIT-LOG(?!-) audit.{{ $p.producer }} false
```

It has no quotes or spaces, so it stays a single token, and the two modules stay consistent. The `(?!-)` lookahead
stops it matching longer names that start with `AUDIT-LOG-`. Escaping the quotes
also works, because an unquoted token isn't unescaped and `\"` is a literal quote in the regex. That version was
used to verify the fix on the cluster: `create_project`, `delete_project` and `read_audit_log` reached the
`audit_logs` stream with `openchoreo_cluster_instance=openchoreo`, and read back through
`POST /api/v1alpha1/audit-logs/query`. api-server logs came back in platform logs as well.

Nothing tests the rendered Fluent Bit rule. A test that runs Fluent Bit against a sample audit line and a normal
line would have caught this.

## 2. Log backlog is dropped at startup

On a fresh install Fluent Bit comes up before OpenObserve, reads every existing container log file
(`Read_from_Head On`) and fails to deliver it:

```
[error] [output:http:http.0] no upstream connections available to openobserve:5080   (x200)
[error] [engine] chunk '1-1789732298.217156930.flb' cannot be retried: task_id=5, input=tail.0 > output=http.0
```

Result: `http.0 dropped_records=5956`, `retries_failed=88`. Because the tail DB records those lines as read, they
are never sent again. Suggest a higher `Retry_Limit` (or `no_limits`) on the outputs, filesystem buffering, or an
init container that waits for OpenObserve.

## 3. The base install command fails as written

`fluent-bit.enabled` defaults to `true`, and `templates/fluent-bit/config.yaml` fails when
`fluentBitCustomizations.clusterInstance` is empty. The README's first command sets neither:

```
Error: execution error at (observability-logs-openobserve/templates/fluent-bit/config.yaml:6:4):
fluentBitCustomizations.clusterInstance is required. It names the cluster these records came from and cannot be defaulted.
```

The later "Enable log collection" and "Enable audit log collection" steps then fail too: they use
`helm upgrade --reuse-values` without `--install`, and no release exists. The text ("enable Fluent Bit in the
already installed Helm chart") suggests Fluent Bit is off by default, but it's on. Either default Fluent Bit to
off, or add `clusterInstance` to the first command and fold the steps into one.

Working command:

```bash
helm upgrade --install observability-logs-openobserve \
  oci://ghcr.io/openchoreo/helm-charts/observability-logs-openobserve \
  --namespace openchoreo-observability-plane --create-namespace --version 0.7.0 \
  --set fluent-bit.enabled=true \
  --set fluentBitCustomizations.clusterInstance=<cluster-name> \
  --set auditLogs.enabled=true
```

## 4. No guide for moving from OpenSearch

The README covers a clean install only. Replacing the OpenSearch module hits these undocumented problems:

- **Tracing breaks.** `observability-tracing-opensearch` is normally installed with `openSearch.enabled=false` and
  uses the OpenSearch cluster owned by the logs module's release. Uninstalling the logs module deletes it.
- **Events break.** `observability-events-otel-collector` keeps exporting to `https://opensearch:9200` and
  fails (`Exporting failed. Rejecting data.`). The logs adapter reads events from `k8s_events`, which nothing writes
  until the collector is reconfigured using the "OpenObserve" section of the events module README. That README's
  example uses stream `k8s-events`. OpenObserve normalizes it to `k8s_events`, so it works, but the two READMEs
  should use the same name and link to each other.
- **Leftovers.** `helm uninstall` leaves the `opensearch-master-opensearch-master-0` PVC (8Gi) and the
  `opensearch-admin-credentials` ExternalSecret and Secret.
- Audit lines written between removing the old module and fixing #1 are lost.

Suggest a short "Migrating from observability-logs-opensearch" section: uninstall the tracing module (or
switch it to `observability-tracing-openobserve`), repoint the events collector, and clean up the PVC and secret.

## 5. Version pinning

Every command uses `--version 0.0.0-latest-dev`, even at the `0.7.0` tag. The compatibility table says 0.7.x for
v1.3.0. The note at the top covers this, but at a release tag the commands should use the released version.

## 6. Nonexistent prerequisite value

The README says to deploy the observability plane with `observer.logsAdapter.enabled="true"`. The 1.3.0-rc.1
chart has no such key: `observer.logsAdapter` has only `url` and `timeout`. No change to the observability plane is
needed, because the module's adapter Service is `logs-adapter:9098`, which is the observer's default
`LOGS_ADAPTER_URL`. The prerequisite should say that instead.

## 7. Credentials recipe

- The OpenChoreo k3d/quick-start OpenBao seed already has `secret/openobserve-admin-email` and
  `secret/openobserve-admin-password` (property `value`). The README creates a different path,
  `openobserve-admin-credentials`, with `ZO_ROOT_USER_*` properties, which leaves two copies. Either reuse the
  seeded paths in the ExternalSecret or mention them.
- The placeholder `YOUR_USERNAME` should be an email (`ZO_ROOT_USER_EMAIL`), and the README should state
  OpenObserve's root password rules (the e2e config already had to change its password to meet them).
- `kubectl exec -it` isn't needed for `bao kv put` and fails in non-TTY shells.

## 8. No container log retention setting

The setup job sets retention only on `audit_logs` (365d). The `default` and `k8s_events` streams show
`data_retention: 0` and fall back to OpenObserve's global default. The OpenSearch module sets
`dataRetentionTime.containerLogs: 30d` and `k8sEvents: 30d`. With a 10Gi PVC this deserves a setting (or at
least a docs note about `ZO_COMPACT_DATA_RETENTION_DAYS`).

## 9. Adapter crash-loop on startup

`logs-adapter-openobserve` exits when OpenObserve isn't reachable yet:

```
Failed to connect to OpenObserve. Cannot continue without it. Hence shutting down
```

It restarted 4 times on a fresh install before settling. It recovers on its own, but it looks like a failure. It
should retry instead of exiting.

## 10. Audit read API lacks cluster attribution (OpenChoreo, not the module)

The README says that with a shared audit destination, "records from each cluster keep their
`openchoreo_cluster_instance`, so a shared destination can still tell the clusters apart." That's true of the
stored records. But `AuditLogCollectorInfo` in `openapi/observer-api.yaml` only has
`namespaceName`/`podName`/`containerName`, and `query_audit_logs` has no cluster filter. Through the observer
there's no way to tell the clusters apart.

## Verified working

- ExternalSecret → `openobserve-admin-credentials` synced from OpenBao.
- `openobserve-setup-logs` Job completed. It created the alert template and destination, plus the `audit_logs` stream with 365d.
- Container logs → `default` stream, read back through `GET /api/v1alpha1/platform-logs` with
  `clusterInstance=openchoreo`.
- Audit logs end to end, **after the #1 fix only**.
- Events collector → `k8s_events` stream. The events read path through the observer wasn't tested, because the cluster has no components.
- Audit producer allowlist defaults match the real containers (`openchoreo-api/api-server`, `observer/observer`).

## Cluster state after the tryout

- The #1 fix is a **hand-patched `fluent-bit` ConfigMap**. The next `helm upgrade` of this release reverts it
  and breaks audit again. The rule is hardcoded in the template, so no `--set` can fix it.
- The OpenSearch PVC and `opensearch-admin-credentials` ExternalSecret are still there.
