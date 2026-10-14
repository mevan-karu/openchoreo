# Audit Coverage — Classify Read Operations Explicitly

**Status:** ✅ Shipped as [#4703](https://github.com/openchoreo/openchoreo/pull/4703) (`7140fa08f`, 2026-09-10)
**Repo:** `openchoreo` · **Date:** 2026-09-09, revised 2026-09-11 to match what landed

> This was a build plan. It is kept as the record of why the audit exemption tables look the way
> they do, because two of its decisions were reversed during review and the reasoning is worth
> having. Anything describing a separate `ReadOperations` table or a coverage matrix is superseded
> by §3.

---

## 1. The problem

The audit scope is *"every state-modifying operation"*, and HTTP method was the proxy for it,
hardcoded in three places:

```
tools/internal/auditgen/generator.go                        ← generation
internal/observer/audit/audit_coverage_test.go              ← observer's gate
internal/openchoreo-api/mcphandlers/audit_coverage_test.go  ← openchoreo-api's gate
```

Each said `if method == "GET" { continue }`.

**The proxy was corrected in one direction only.** A POST that is really a read got an entry in
`RESTExemptions`. A GET that ought to be audited had no mechanism at all, and in the generator the
skip ran *before* the `Overrides` lookup, so not even an explicit override could reach one.

Concretely: add `GET /secrets/{name}/value` and it is silently, permanently unaudited. Nothing
fails, nothing warns, no reviewer is prompted.

Two smaller defects came from the same design. The exclusion list was duplicated between
`tools/auditgen/config_*.go` and `internal/*/audit/exemptions.go` with no test tying them together.
And `RESTExemptions` conflated *"this mutation is deliberately unaudited"* with *"this is a read"*.

## 2. The fix

Stop inferring the classification; declare it. Every operation in every spec is now either

- **audited** — a definition generated from the spec, or
- **exempt** — an entry in `RESTExemptions` carrying a reason.

The generator walks every operation, and a GET that no table excludes **fails generation** with a
message naming it. An `Overrides` entry is how a read that *should* be audited gets a definition,
since `deriveDefinition` has no verb for GET. Both coverage gates drop the GET skip, so the universe
they check is the whole served surface.

`tools/auditgen`'s configs now read `RESTExemptions` from `internal/*/audit` instead of keeping
their own copy.

## 3. Two decisions reversed during review

**Reads live in `RESTExemptions`, not a separate table.** The plan proposed a `ReadOperations`
set on the grounds that 101 reads would bury the handful of real judgement calls, and that a read
needs no bespoke reason. Reversed: one mechanism is simpler than two, and shared reason constants
(`reasonRead`, `reasonReadAsPOST`) keep the source readable without a second concept. The argument
that lost was mostly about how the generated coverage matrix would render — and the matrix was
deleted in the same change, so it no longer applied.

**The coverage matrix and `tools/auditcoverage` were deleted, not extended.** ~530 lines of tool
and 290 lines of generated Markdown. Nothing referenced them outside audit planning docs, they were
never published to the docs site, and the exemption reasons they rendered now sit next to the
entries themselves. DO-2 (*"a coverage matrix stating which actions are audited, and explicitly
which are not"*) is consequently **unmet**, recorded here rather than quietly dropped — see §6.

An `x-audit` spec annotation was considered and rejected on cost: no `x-` convention exists in these
specs, and it would mean annotating all 214 operations. Still the better end state if the question
is revisited.

## 4. What shipped

| Area | Change |
|---|---|
| `tools/internal/auditgen/generator.go` | GET skip removed; unclassified GET fails with a message naming the operation; `Overrides` reachable for a GET |
| `tools/auditgen/config_*.go`, `exclusions.go` | exclusions derived from each service's `RESTExemptions`; duplicated copies deleted |
| `internal/openchoreo-api/audit/exemptions.go` | 100 GETs + `Evaluates` added with shared reason constants |
| `internal/observer/audit/exemptions.go` | 7 GETs + 8 `Query*` POSTs added; the 4 internal-port entries keep their own reason |
| both `audit_coverage_test.go` | universe is every operation, not just non-GET; pinned counts updated |
| `tools/auditcoverage/`, `docs/audit/`, `make audit-coverage-matrix` | deleted |

Three new generator tests: an unclassified GET errors, an exempted GET is skipped, an overridden GET
produces a definition. The last was impossible before.

**`definitions.gen.go` is byte-identical for both services.** That was the review contract — a
~120-line diff of declarations and provably no change to what is audited.

## 5. What implementation turned up

**The duplication had already drifted.** `tools/auditgen/config_observer.go` was still excluding
`QuerySpanDetailsForTrace`, an operation reverted out of the spec. Dead config nothing caught, which
is the failure the single-source change removes.

**`gocyclo` rejected a three-way classification check** at complexity 34 against a limit of 30. Moot
once the design collapsed to two-way.

**The matrices had a freshness gate** (`TestCoverageMatrix_IsFresh`), so `make audit-coverage-matrix`
was not optional after touching prose, despite sitting outside `code.gen`. Deleted with the tool.

## 6. Open item

The **Known non-events** prose was deleted with `tools/auditcoverage`. It documented real gaps that
are not derivable from code:

- A malformed path parameter never reaches audit on REST; oapi-codegen binds and returns 400 before
  `HandlerMiddlewares` run.
- `source_ip` is unobtainable for a direct MCP client, and both `source_ip` and `X-Request-ID` are
  trusted from the client with no proxy check.
- `?filterByAuthz=false` records a service-layer denial as `failure`, not `denied`, and it cannot be
  recovered in `mcpaudit.classifyResult`.
- `pkg/mcp.NewSTDIO` registers no middleware, so whatever first wires MCP-over-stdio gets an
  unaudited mutating surface.
- An authentication rejection carries no `action`, `category` or `operation_id`.

Recoverable from `git show 7140fa08f^:tools/auditcoverage/service_openchoreo_api.go`. It belongs in
the audit schema reference's limitations section when that is written — the delivery plan §3.1
already plans the page. **Not filed as an issue yet.**
