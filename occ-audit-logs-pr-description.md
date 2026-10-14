## Purpose
Adds CLI support for querying audit logs.

## Approach
- New `occ auditlogs` command. It reads `/api/v1alpha1/metadata` to check that audit logs are enabled and find the observer, then queries `POST /api/v1alpha1/audit-logs/query`.
- Supports every audit log filter, `--since`/`--start`/`--end` windows (up to 366 days), `--limit`, `--sort`, and text or JSON output.
- When more records match than were shown, prints the `--start`/`--end` for the next page.
- Adds `GetMetadata` to the occ API client.
- Moves the observer API client into `internal/occ/observerclient`, shared with `platformlogs`.

## Related Issues
Resolves #4731

## Checklist
- [x] Tests added or updated (unit, integration, etc.)
- [ ] Samples updated (if applicable)
- [ ] Added `backport/<release-branch>` label if this should be backported (e.g., `backport/release-v1.0`)
- [x] This PR includes AI-generated code or content

## Remarks
Paging across sub-second boundaries also needs the OpenSearch adapter to keep sub-second precision in the query window. That fix is in openchoreo/community-modules, a separate repo.
