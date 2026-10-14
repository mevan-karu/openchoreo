# fix(api): give the audit logger its own JSON handler

## Purpose
> Briefly describe the problem or need driving this PR and how it resolves the issue. Include links to related issues if applicable.

Audit records go to stdout alongside application logs, and the next phase of the epic has a log collector pick them out and route them to their own index. That only works if the records are reliably parseable.

They aren't. `NewLogger` reused the application logger's handler, so the audit format followed `logging.format`. Observer selects a text handler when `LOG_LEVEL=debug`, and in that mode it emits audit records no collector can parse — silently, since nothing downstream exists yet to notice.

## Approach
> Summarize the solution and implementation details.

`NewLogger` takes an `io.Writer` and builds its own `slog.NewJSONHandler`. Both producers pass `os.Stdout`, the same file the application logger writes to.

`forceLevelHandler` is deleted. It existed to stop a borrowed handler's level filter suppressing audit records; an owned handler sets its own level, so `audit.enabled` remains the only kill switch with nothing left to compensate for.

Test helpers that previously passed a `*slog.Logger` now take the sink writer directly, which is usually the buffer the test was already asserting on.

## Behavioural changes
> Changes visible to callers or operators.

- Audit records are JSON whatever `logging.format` or `LOG_LEVEL` is set to. Application logs are unaffected.
- The `component` slog attr is no longer emitted. It came from `logging.NewWithComponent`, so it appeared on `openchoreo-api` records and never on observer's, and it duplicated `producer`, which is inside the event and consistent across both.

Both are slog envelope fields rather than part of event schema v1, so neither is a schema version bump. The seven golden `want` strings in `record_shape_test.go` are unchanged, so the published event itself is byte-identical.

## Related Issues
> Include any related issues that are resolved by this PR.

Part of #3343. Prerequisite for the collector-side audit routing that follows.

## Checklist
- [x] Tests added or updated (unit, integration, etc.)
- [ ] Samples updated (if applicable)
- [ ] Added `backport/<release-branch>` label if this should be backported (e.g., `backport/release-v1.0`)
- [x] This PR includes AI-generated code or content

## Remarks
> Add any additional context, known issues, or TODOs related to this PR.

`make test` passes and `golangci-lint` reports 0 issues. `make code.gen` produces no drift — no generated file, Helm value or schema is touched.

`TestLogEvent_NotGatedByAppLogLevel` is replaced rather than deleted. With the handler owned there is no application logger in scope to be gated by, so the old assertion could no longer fail; `TestLogEvent_PublishesJSONRegardlessOfAppLogging` pins the property that replaces it — a JSON record carrying the `AUDIT-LOG` marker while the application logger is a text handler at `LevelError`.

Two things recorded in doc comments rather than enforced in code. The `AUDIT-LOG` message is the collector's routing key, so it is stable across schema minor versions. And the two handlers stay non-interleaving because they share one `*os.File`: slog writes each record in a single `Write`, and `os.File` serializes writes per descriptor. Wrapping either side in a buffered or separately-opened writer would quietly lose that, which is why it is stated at `NewLogger`.

The routing contract belongs in the generated audit schema reference, but `docs/audit/schema-reference.md` does not exist yet — it is still an open item from the schema-v1 phase. Until it lands, the contract lives in the `NewLogger` and `LogEvent` doc comments.
