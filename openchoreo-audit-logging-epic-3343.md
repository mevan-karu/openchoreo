### Please select the area issue is related to

Area/SystemAuthorization (Access Control of the OpenChoreo System)

### Description

OpenChoreo has an audit logging framework today, introduced in PR [#1465](https://github.com/openchoreo/openchoreo/pull/1465). It records the core of an audit event — who performed an action, what action it was, which resource was affected, when, and whether it succeeded, failed or was denied — for a set of state-changing operations in `openchoreo-api`.

What we do not have yet is a complete audit logging *system*:

- **Coverage is partial.** Only `openchoreo-api` is instrumented, and nothing stops a new operation from being added without auditing. A later refactor also removed the middleware wiring, so events are not currently being emitted at all.
- **Audit logs are mixed with application logs** — same output, retention and access rules, though audit records normally need to be kept longer and seen by fewer people.
- **There is only one way to publish them.** Users who already run an audit or SIEM system have no supported way to receive OpenChoreo's audit events.
- **Nobody can read them back.** There is no query API and no UI, so answering "who deleted this component?" means going to the raw log backend.
- **Clients cannot record their own actions.** The Backstage portal does things that never reach an OpenChoreo API.
- **Object-specific details are ad hoc.** A promotion needs source and target environments, a role mapping needs role and effect, a workflow run needs the commit. Each handler adds these as free-form metadata in its own way, so consumers cannot rely on them being present or consistently named.

The goal of this epic is to close these gaps: one canonical event format with declared per-object attributes, complete and enforced coverage of state-changing operations, audit logs on their own stream, pluggable publishing, storage and analysis through OpenChoreo's observability modules, and viewing through OpenChoreo clients behind a dedicated permission.

The design is set out in [Discussion #4396](https://github.com/openchoreo/openchoreo/discussions/4396).

#### Initial delivery

The first delivery is **log-based**. Audit events are emitted as structured records on their own stream, collected by the existing log-collection agent, and routed to a dedicated audit index in the observability plane where retention can be set independently of application logs. They are then queryable through the Observer API and viewable in the Backstage portal.

Getting there means restoring emission and making the coverage regression impossible to repeat, extending coverage to every state-changing operation with a check that catches new ones, freezing the event schema, and declaring per-object attributes.

The following remain in scope for the epic but are not part of the initial delivery:

- **Webhook publishing** to an external audit or SIEM system.
- **Producer-side durable queueing** — in the log-based delivery, buffering and retry come from the collecting agent.
- **Client-side auditing**, so the portal can record actions that never reach an OpenChoreo API.
- **Blocking mode**, where an action fails rather than succeeding unaudited.

### Feature Request Issue

N/A

### Feature Proposal Discussion

https://github.com/openchoreo/openchoreo/discussions/4396

### Sub Tasks

1. We need to re-evaluate the overall auditability of OpenChoreo's operations
2. Produce tutorials for extracting and publishing all these audit logs to an external system (if applicable)
3. Assess a design for an optional Audit Logs module (UI + audit event storage system)

## Task Breakdown

- [ ] Restore audit event emission and prevent silent removal
- [ ] Audit every state-modifying operation, and enforce it in CI
- [ ] Define and freeze the canonical audit event schema (v1)
- [ ] Declared, typed and validated per-object audit attributes
- [ ] Publish control-plane telemetry to a configured observability plane
- [ ] Store and query audit logs through the observability module
- [ ] View the system audit trail in the Backstage portal
- [ ] Per-category pre-action and post-action audit events
