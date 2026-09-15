# 0007 – Memories are personal or team; the classifier suggests, a router decides

**Status:** accepted (2026-09-14)

## Context

Memories are personal by nature. Users (and the agent acting for them)
should create memories only for themselves or their team, never for
everyone. Memories are meant to be created automatically by the
classifier's miner, but the miner cannot reliably know which store a
memory belongs in, especially for users in several teams.

## Decision

- Agent-driven memory writes target only the **personal** layer or a
  **team** layer. There is no `create-global-memory`; a global memory
  store, if configured, is admin-curated and read-only to agents.
- The miner attaches a **suggested audience** (personal or team:T) to each
  candidate. It never chooses a store.
- A **memory router** decides the layer. A team layer is chosen only if
  the token holds `create-team-memory`, T is in the token's groups, and a
  positive signal exists: the session's `active_team` is T, a team
  classifier rule auto-accepts the candidate, or the user confirms.
  Otherwise the memory is written to personal with the team suggestion
  recorded, or dropped if the token cannot write personal either.
- Sessions gain an `active_team` attribute set by the client.
- Personal memories carrying a team suggestion can be **promoted** later
  via the API; promotion copies with provenance.

## Consequences

- Private by default: a wrong guess by the miner keeps a memory personal;
  it can never leak into a team store without a positive signal.
- Clients need a way to set the active team (flag, dropdown, setting) and
  to answer `memory_confirm` events.
- Team classifier rules gain a practical use: declaring what the team
  auto-accepts.
- Review-before-visibility for team memories is an open question.
