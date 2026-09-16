# 0010 – Resolutions from the pre-audit review (blocking findings)

**Status:** accepted (2026-09-14)

## Context

Three independent reviews of the design notes found nine blocking items:
six contradictions between notes and three gaps an implementer could not
code around. Each was resolved with the smallest rule that keeps the
rest of the design intact.

## Decisions

1. **The global layer is always in the chain.** Its prompt fragments,
   classifier rules, and gates are the policy baseline and need no
   scope; its skills, tools, and memories steps still require the
   matching `use-global-*` scope. (Was: the whole layer dropped without
   a global scope.)
2. **`run-server-tools` and `run-high-risk-tools` are starting
   permissions**, checked by the service scope gate before any rule.
   The starting set is now twelve permissions.
3. **The global `allow` for `load_skill` is unlocked.** Skill loading
   skips the model audit by default, but any team or user `deny` or
   `require_approval` still fires. (Was: locked, which made those rules
   dead.)
4. **`tool_call_proposed` is emitted twice for client tools**: first
   `state=reviewing`, then `state=execute` only after the audit allows.
   Clients execute only on `state=execute`. Now in the API event table.
5. **Restrictive mining rules run once, before routing**, from every
   layer; widening rules run in their own layer's
   `on_memory_candidate` step. The pipeline note now says the same as
   the walkthrough.
6. **Rule ordering wording unified**: restrictive rules are evaluated in
   layer order and accumulate across layers; `locked` matters only for
   `allow` (a locked `allow` shields the call from later layers, an
   unlocked one yields to any restriction).
7. **Context-window overflow**: the core checks `count_tokens` before
   every model call; with provider compaction available it enables it,
   otherwise it emits `context_exhausted`, ends the session (mining and
   summary memory run), and the client creates a successor with
   `resume_from`. Further messages to the exhausted session return 409.
8. **Late background-job results are user-role marker messages**, never
   `tool_result` blocks, so they are valid in any append-only history
   including a different session's.
9. **User-layer sources are keyed by subject**: every user-layer path
   is templated on `${user_root}` = `<users_root>/<token.sub>`; the
   `~/.agent` form is single-user dev mode only. Capability ceilings use
   the same root.
10. **The user's message is appended to the conversation** at the start
    of the turn; the model always receives `conversation.messages`.

## Consequences

- `chain_for` takes the session as well as the token; the session layer
  and the global layer are unconditional.
- Deployments must set `users_root`; single-user mode sets it to a home
  directory with a fixed subject.
- The API gains a `context_exhausted` event and a 409 on exhausted
  sessions.
- Remaining review findings (important and minor) are not covered here.
