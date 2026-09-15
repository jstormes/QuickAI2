# 0002 – Every client has its own session

**Status:** accepted (2026-09-14)

## Context

With the framework exposed as a web API, an earlier draft allowed several
clients (e.g. CLI and web) to attach to one session and receive the same
event stream. That raised questions about who answers approval requests
and how to fan out events.

## Decision

A session belongs to exactly one client. Clients never share a session.
Running several front ends at once means several independent sessions
against the same service.

What is shared across all sessions: **skills, memories, and the
classifier**. Each is a service-level resource; sessions read from (and,
for memories, write to) them according to the session's token scopes.

## Consequences

- No event fan-out, no multi-responder approval rules. Each session has one
  event stream and one responder.
- Cross-client continuity is the job of the memory layers, not the session.
  A session-summary memory written at session end is a candidate mechanism
  (see IDEAS.md).
- Session state can be simpler: it is private to one client and can be
  discarded or persisted per client policy.
- The classifier must be stateless with respect to sessions, or keep any
  per-session state keyed by session id inside the service.
- Memory writes from one session are visible to other sessions of the
  layer and scopes as soon as the store commits them.
