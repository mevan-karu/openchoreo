Thanks! Source routing is actually in place already, in the Fluent Bit config of the community logs modules.

Each module has an `auditLogs.producers` allowlist of namespace and container pairs, which by default covers only the API server and the observer ([values.yaml](https://github.com/openchoreo/community-modules/blob/73f0c85f762c6cd84c0cf6363b31dfca0c39ea95/observability-logs-opensearch/helm/values.yaml#L137-L149)). Fluent Bit matches that list against the log file path the kubelet writes, not the log line ([opensearch config.yaml](https://github.com/openchoreo/community-modules/blob/73f0c85f762c6cd84c0cf6363b31dfca0c39ea95/observability-logs-opensearch/helm/templates/fluent-bit/config.yaml#L58-L71), [openobserve config.yaml](https://github.com/openchoreo/community-modules/blob/73f0c85f762c6cd84c0cf6363b31dfca0c39ea95/observability-logs-openobserve/helm/templates/fluent-bit/config.yaml#L59-L79)). If any other pod prints an audit-shaped line, it stays an ordinary container log and never reaches the audit index.

What we've deferred to later phases is the hardening beyond that:

- Stronger producer identity. Right now the trust boundary is the namespace, so anyone who can create pods in the control plane or observability plane namespace could write audit records.
- Tamper evidence. Records aren't signed or hash-chained yet.
- Client-submitted audit events. Only the platform components produce them today.
