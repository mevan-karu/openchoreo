# refactor: serve the observer API from its generated OpenAPI server

## Purpose

The observer generated a complete OpenAPI server and served none of it. All 18 operations the spec declared were registered by hand on two `ServeMux` instances, so `openapi/observer-api.yaml` was documentation that nothing enforced and `make code.gen` kept regenerating dead code.

That hid a real problem. The spec said the four alert-rule CRUD operations required a Bearer token. In production they were served with no authentication at all on internal port 8081, and nothing could catch the mismatch, because the generated server that would have enforced it was unused.

This PR puts routing, parameter binding and authentication under the specs, and makes each spec state the truth about its port.

## Approach

The observer serves two ports with different exposure and different auth, so I split the spec one per port. `HandlerWithOptions` registers all of a spec's routes on whichever mux it is handed, so keeping one spec would have meant either exposing alert-rule CRUD on the public port or maintaining a hand-written port-to-operation table. The second option is the same drift problem in a new place.

Two commits:

- `a7c51661` internal port (8081), 5 operations. New `observer-internal-api.yaml`, new `internalgen` package.
- `2ac65c21` public port, 14 operations. `ObserverMiddlewares` with `auth.OpenAPIAuth`, the five alert schemas moved into the internal spec, and `/.well-known/oauth-protected-resource` brought into the spec (which retires `internal/server/oauth`).

Internal went first on purpose. It is the smaller half with no auth question attached, so every mechanism was proven there before touching the 12 authenticated operations.

Auth is now decided by the spec. `auth.OpenAPIAuth` wraps every public route and reads the scopes key the generated wrapper sets, so `/health` is public because it carries `security: []`, not because someone registered it outside the JWT group. Two of the 14 public operations are public that way and the other 12 require a token. The internal spec declares no security scheme, which is what port 8081 actually enforces, so `internalgen` contains zero `BearerAuthScopes` references and needs no auth wrapper at all.

`/.well-known/oauth-protected-resource` moved into the spec as part of this. openchoreo-api already declared it and served it from a strict handler, so the observer was the odd one out. That leaves `internal/server/oauth` with no consumers, so it is deleted: under generated routing each service has to return its own generated response type, and a shared helper returning a neutral struct cannot satisfy either strict interface. That is why openchoreo-api stopped using it in the first place.

Responses are unchanged on the wire. One `apiResponse` type implements all 14 `Visit*Response` methods and writes the service's own value through the same `WriteJSON` as before. The generated typed responses were not usable here: three services return bare `any`, and `MetricsQueryResponse` is a `oneOf` union.

55 files, +4,962/-3,861. `cmd/observer/main.go` has exactly one hand-written route pattern left, `/mcp`, and it stays out of the spec on purpose: it is streaming JSON-RPC rather than a request/response operation, and it could not go through the strict chain anyway, since the wrapped `ResponseWriter` breaks the `http.Hijacker` it needs. openchoreo-api keeps `exec` off its generated router for the same reason.

## Behavioural changes

None of these affect the `ObservabilityAlertRule` controller, the only caller of the four alert-rule CRUD operations. It branches purely on status code, and the one body field it reads on error is `message`, used as log text.

1. `updateAlertRule` now declares a 404. The handler has always returned it on `ErrAlertRuleNotFound`; the old spec declared only 200/400/500. This is the sort of gap a generated server surfaces and a hand-written mux hides.
2. An empty `ruleName` returns 404 instead of 400. `{ruleName}` only matches a non-empty segment, so `/rules/` never reaches the handler now.
3. Missing or unparseable FinOps `startTime`/`endTime` return `INVALID_PARAMETER` from parameter binding rather than a message from `ValidateCostQueryRequest`. Same status, same JSON shape.
4. A malformed body returns `INVALID_REQUEST_BODY` from the strict layer instead of each handler's own message.
5. When the path and body are both invalid, `INVALID_REQUEST_BODY` now wins over `INVALID_SOURCE_TYPE`. The strict layer decodes before the handler runs. Either one alone behaves as before.
6. Content-Type is enforced on all 10 body-bearing public operations. Seven already refused non-JSON via `httputil.BindJSON`; the other three used `json.NewDecoder` and never checked. All ten declare `content: application/json`, so enforcing it uniformly seemed better than restoring a seven-of-ten rule.
7. Parameter-binding 400s bypass auth and the access log. `ServerInterfaceWrapper` calls `ErrorHandlerFunc` before applying middleware, so an unauthenticated request to `/costs/...` with no `startTime` gets a 400 instead of a 401. Scope is exactly the two FinOps operations, since no other parameter can fail to bind. I left this alone: oapi-codegen has no hook between routing and `ErrorHandlerFunc`, and re-checking auth inside the error handler would rebuild the per-route auth this PR removes.
8. `gen.SpanStatusCodeError/Ok/Unset` are now `gen.Error/Ok/Unset`. oapi-codegen only prefixes enum constants on collision, and moving the alert schemas removed the one on `"error"`. `gen` already had bare constants like `Created` and `Synced`, and there was a single usage to update, so I took the rename over renaming ~40 constants with `always-prefix-enum-values`.
9. `openchoreo_security_enabled` in the protected-resource metadata now reports the truth. It was hardcoded `false`, so the observer advertised that authentication was disabled even with JWT enforced. The observer has no `cfg.Security.Enabled` equivalent, so `JWT_DISABLED` is the source of truth, read through a `jwtEnabled()` helper that `initJWTMiddleware` also uses so the two cannot drift. No consumer reads this today: it is an OpenChoreo extension rather than part of RFC 9728, so generic MCP clients ignore it, and the one reader in the repo (`occ`) only ever fetches the control plane's metadata. The value is in not publishing a false claim about our own auth posture, and in not leaving a trap for whoever adds a consumer later.
10. JSON key order changed in that same metadata response. The generated struct orders fields alphabetically rather than in declaration order. No conforming client should care, but it is a diff if anyone golden-tests the body.

## Breaking changes

None for any caller, in-cluster or external.

Authentication is identical on every route: 12 public operations still require a Bearer token, `/health` and `/.well-known/oauth-protected-resource` are still open, the internal port still requires nothing. No path or method changed, and no operation moved between ports. The five operations that moved between spec files are the same five already served only on 8081, on a `ClusterIP` service that is not gateway-routed, so nothing externally reachable changed. Observer's generated `client.gen.go` has no consumers, so regenerating it breaks nothing.

The changes above are message and ordering changes inside unchanged status codes, plus items 1 and 2, which are status changes on the cluster-internal API whose only caller ignores response bodies, and item 9, which corrects a field that was previously wrong.

### One thing this PR deliberately does not fix

The four alert-rule CRUD operations are still unauthenticated and state-modifying on port 8081, with network reachability as the only control. This PR does not change that. It makes the spec say so instead of claiming a Bearer token is required.

Closing it means giving the `ObservabilityAlertRule` controller a service-account token, which is real observability-plane work rather than a line change. Putting a security-posture change that breaks an observability-plane controller inside a routing refactor would make both harder to review and harder to revert. `TestInternalSpecDeclaresNoSecurity` fails if anyone adds a security scheme to the internal spec without that work, and says why.

## Related Issues

<!-- Tracking issue for the migration, and the follow-up for enforcing auth on the internal alert-rule API. -->

## Checklist

- [x] Tests added or updated (unit, integration, etc.)
- [ ] Samples updated (if applicable)
- [ ] Added `backport/<release-branch>` label if this should be backported (e.g., `backport/release-v1.0`)
- [x] This PR includes AI-generated code or content

## Remarks

Handler tests now drive the generated router instead of calling handler methods, so routing, parameter binding, decoding and the handler all run together. That is more coverage than before and it is what makes the spec authoritative in tests too.

Two tests took a second attempt, because the obvious assertion did not actually discriminate:

- `TestInternalMiddlewareOrdering`. Asserting that the logger saw the request path passes in both orderings, since `Recovery` logs the path itself. What separates them is the logger's own `"HTTP request"` line, emitted after `next` returns, which survives a panic only when the logger sits outside recovery. Verified by reversing the slice and watching it fail.
- `TestAbsentStartTimeKeepsRequiredMessage`. The generated time fields have no `omitempty`, so an absent `startTime` decodes to the zero time, and formatting that gives `"0001-01-01T00:00:00Z"`, which parses fine. `"startTime is required"` would never fire and callers would see `"query time range cannot exceed 30 days"` instead. Status stays 400 either way, so only the message catches it. Verified by deleting the guard and watching all four cases fail with that exact wrong message.

Clean: `go build ./...`, `go vet`, `make lint`, `go test ./internal/observer/... ./cmd/observer/... -count=1`, and codegen idempotency.

This is also the prerequisite for observer audit logging. On generated routing the audit middleware's pattern map and the mux's patterns come from one source and cannot disagree; on a hand-registered mux they can, and compensating for that would cost a route-table package plus a spec-agreement gate that only exists to detect a hazard generated routing does not have.
