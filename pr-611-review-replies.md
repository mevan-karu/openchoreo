# PR 611 — review reply drafts

Not posted. Text for the two CodeRabbit findings deliberately not fixed.

## `tls.verify Off` on the audit OUTPUT (config.yaml)

This mirrors the chart's existing posture rather than introducing it: the two pre-existing
`[OUTPUT]` blocks that ship container logs and k8s events carry the same setting (lines 52
and 118). Turning it on for the audit output alone would leave the chart inconsistent, and
turning it on for all three needs a CA bundle mounted into the collector — a chart-wide
change with its own rollout, not something to fold into an audit PR. Better as its own
issue covering all three outputs together.

## Timeline buckets aligning to epoch rather than `startTime`

Intended. The contract's requirement is that the buckets cover the window *contiguously*,
which `min_doc_count: 0` plus `extended_bounds` already delivers — no gap can be charted as
a continuous line. Epoch alignment is `date_histogram`'s normal behaviour and what a caller
reading OpenSearch's timeline elsewhere already sees. Adding `offset` and `hard_bounds` to
shift phase interacts with the coarsening in `ResolveTimelineInterval` in ways that are easy
to get subtly wrong, for a bucket boundary that no consumer currently depends on. Worth
revisiting if the portal needs exact phase alignment.
