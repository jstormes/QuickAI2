# 0006 – Users can override shared skills with personal forks

**Status:** accepted (2026-09-14)

## Context

Global and team skills are maintained by other people. A user often wants
a shared skill with small changes (different defaults, extra steps,
a tool swapped out) without affecting anyone else. Layer shadowing
already makes a same-name personal skill win over a global one; this
decision turns that into a supported workflow.

## Decision

- A user holding `create-personal-skill` (and `use-*` for the base layer)
  can fork any non-locked global or team skill into their personal layer
  under the same name. The fork is a full copy with `derived_from`
  recording the base id, layer, and version.
- The fork shadows the base for that user in discovery, loading, tools,
  and prompt fragments. The base stays loadable by full id. Deleting the
  fork restores the base.
- A skill marked `locked` in a more trusted layer cannot be shadowed or
  forked.
- The service tracks staleness (base version moved) and exposes a diff;
  it does not auto-merge.

## Consequences

- `Skill` gains `version`, `derived_from`, and `locked`.
- New API operations: fork, base, diff; a `skill_shadowed` event.
- Skill stores must expose a stable version per skill (content hash is
  acceptable).
- Team-level overrides of global skills work identically with
  `create-team-skill`.
- Overlay-style forks and auto-rebase are deferred (IDEAS.md).
