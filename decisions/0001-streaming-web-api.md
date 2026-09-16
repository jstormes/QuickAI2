# 0001 – The framework presents itself as a streaming web API

**Status:** accepted (2026-09-14). **Amended by 0010/0011**: WebSocket is
resolved as an optional transport adapter beside SSE (`08-walkthrough.md` §17e).

## Context

The framework must support arbitrary front ends (CLI, web, native app,
possibly all at once against one running service) and background work such
as memory mining. The earlier open question was whether the
core should be a library or a long-running service.

## Decision

The framework runs as a service and exposes an HTTP web API with streaming
responses (SSE by default, WebSocket under consideration). All front ends
are clients of that API. Internally the core keeps an event-bus boundary so
an in-process adapter remains possible for tests.

## Consequences

- One protocol and event schema to design and version; see
  `design/03-web-api.md`.
- Front ends can be written in any language and run on any device.
- Background tasks have a natural process to live in.
- Adds operational surface: auth, session persistence, deployment.
- Pure embedded/library use becomes a secondary path, not the primary one.
