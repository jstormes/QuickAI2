# 0005 – Add a team layer between global and user

**Status:** accepted (2026-09-14)

## Context

The original requirement named two ownership levels: globally shared and
user-specific. Real organizations have groups that share skills and
memories without sharing them with everyone. The layering design already
allowed extra layers; this decision makes one of them concrete.

## Decision

The decided layer order is **global, team, user, session**. The team
layer holds skills, memories, and prompt fragments owned by a team and
backed by that team's own store. Team membership comes from a `teams`
claim on the OAuth2 token; a user may be in several teams, and the
context resolver expands the team layer into one sub-layer per team.

Four permissions are added: `use-team-skill`, `create-team-skill`,
`use-team-memory`, `create-team-memory`. They apply to all teams in the
token's claim; there are no per-team permission strings in the starting
set.

## Consequences

- Store configuration for the team layer is templated by team id, so each
  team can have a different backend.
- The context resolver gains a team-expansion step and must define an
  order among a user's teams.
- Peer conflicts (same skill name in two of a user's teams) and per-team
  permission granularity are open questions in IDEAS.md.
- Other candidate layers (framework, project) remain undecided.
