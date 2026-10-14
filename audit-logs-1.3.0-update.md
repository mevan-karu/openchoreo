## Audit logs update

Where we've got to on audit logging for 1.3.0.

**Walkthrough:** recording to follow

### Working now

- Audit events are being emitted again, through a shared policy-driven framework used by both the REST API and MCP. Taking audit out of a middleware chain now fails a test rather than going unnoticed.
- Coverage of state-modifying operations in openchoreo-api is complete. Each one is audited, or it sits on a short exclusion list with a reason next to it, and CI fails if someone adds or renames one without accounting for it.
- Observer-api is in the same shape. We had to move it onto its generated OpenAPI server to get there.
- The event schema is v1 and frozen. Environment made it in.
- Audit records go out through their own JSON handler, so flipping the app to debug logging can't quietly reformat them.

### Still to come this milestone

- A query endpoint on the Observer, gated behind its own permission.
- The module side for OpenSearch and OpenObserve: audit records landing in a store of their own with retention set independently, the query behind that endpoint, and the configuration an operator needs to set it up documented alongside it.
- The portal view, sitting on top of that API.
- Docs and e2e.

The thing I'm watching is the collection side. It rides on the platform log collection work, which isn't done, and in a log-based setup the collector is what gives us buffering and retry. If that slips we've got no fallback.

### UX

The recording covers it properly, so just the shape:

- Lives under Platform, in a new Compliance tab.
- You query by typing attribute tokens against the v1 event fields. Repeat an attribute and it ORs, mix attributes and it ANDs.
- Time range, columns you can turn on and off, and a chart of events over time you can click into to zoom the window.
- Click a row for the full event. Anything in that panel can be clicked to filter by it.
- Results are success, failed, denied and unauthenticated. The last two are counted separately on purpose. "Policy said no" and "you never got in the door" aren't the same finding.

### Pushed out

- Declared, typed per-object attributes. There's no way yet for a resource to contribute attributes of its own to an event, so anything specific to one kind of resource has to go into free-form metadata where nothing downstream can rely on it.
- Webhook publishing. Log-based only for now, so anyone already running an audit system reads from the log backend instead of receiving pushes. The sink plumbing exists, it just has the one sink.
- Blocking mode, and the durable queue on the producer side. An action can still finish without its audit event landing.
- Pre-action and post-action events, which only really pay off once blocking exists.
- Client-side auditing, so whatever the portal does on its own stays invisible.
- Query support for the other log modules. They return a straight "not supported" until someone writes the handler.
- Forwarding and strictness knobs for operators. Filtering already works, through the policy selectors.

Nothing has changed on the out-of-scope list.

Let me know if anyone wants more detail on any of it.
