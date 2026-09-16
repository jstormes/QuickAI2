# 0016 – Step kinds and the mandatory global layer

**Status:** accepted (2026-09-15). Refines 0009 and 0011.

## Context

0011 declared routing steps "contributing", but every routing step must
stop its hook by consuming a candidate, which the contributing contract
forbids. 0010 made the global layer unconditional while three notes
still described a solo deployment as user plus session.

## Decision

- Steps have three kinds: `gating` (may stop the whole hook; fail
  closed), `contributing` (only append; fail open; `required: true`
  fails the turn), and `consuming` (only in `on_memory_candidate`; may
  return `Stop(Accepted | Discarded | RequireApproval)`; fail open by
  logging, emitting `memory_discarded(reason=store_error)`, queuing the
  candidate on `session.unmined_candidates`, and continuing).
- "Only gating and consuming steps stop the chain"; a consuming stop
  ends only that candidate's hook.
- The `global` and `session` layers are always present. A solo
  deployment lists `global` (builtin tool source and default prompt
  fragments), `user`, `session`. Startup refuses a config without
  `global`.

## Consequences

- `RouteToTeamStep`, `RouteToPersonalStep`, and `ScratchAcceptStep` are
  consuming; the object model and glossary say so.
- The "solo = user + session" sentences in 02, 05, and 07 are replaced.
