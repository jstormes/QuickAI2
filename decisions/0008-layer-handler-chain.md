# 0008 – The turn is a chain of layer handlers over a shared turn context

**Status:** accepted (2026-09-14). Supersedes the per-resource composite
framing in the earlier drafts of `05-design-patterns.md`; does not change
the substance of decisions 0005 (team layer), 0006 (skill overrides), or
0007 (memory routing), which are restated as handler behaviour.

## Context

Earlier drafts gave each layered resource (skills, memories, prompts,
classifier rules, tools) its own composite that iterated the layer list at
its own point of use. Five small chains, one data object describing the
caller. The user's mental model was different and simpler: one top-level
chain that an inbound message passes through, where each layer can stop
the chain with an action or enrich a context and pass it on.

## Decision

- A turn is processed by an **inbound chain** and an **outbound chain** of
  `LayerHandler`s, both in the same order: global, then each team in the
  token's groups, then user, then session (most trusted first).
- Handlers share a `TurnContext` carrying the token, the session, the
  message, and accumulators for every layered resource. Handlers append
  to the accumulators; later handlers overwrite earlier unlocked values
  (last write wins = user precedence); `locked` values are final.
- Only gating returns `Stop`. Contributing a resource never stops the
  chain.
- The model call sits between the two chains and is the only model call
  the core owns. The shared classifier engine is appended to the outbound
  chain by the service as the final gate on tool calls.
- Merge steps (prompt assembly, memory rank fusion, recall budget) are
  strategies run after the inbound chain, not handlers.
- Permissions decide which handlers are in the chain and which resources
  each handler may contribute.
- Configuration is the layer list; each entry names the sources backing
  that layer's handler.

## Consequences

- `05-design-patterns.md` is reframed: chain of responsibility is the
  top-level pattern; factory, decorator, strategy, pipeline, and observer
  are how handlers and merge steps are built.
- The generic "layered resource" shape from IDEAS.md is resolved: the
  layer is the handler.
- Skills loaded mid-turn contribute through the handler of their origin
  layer, preserving shadowing and locking.
- Store interfaces (`SkillStore`, `MemoryStore`, ...) survive unchanged as
  the backends a handler delegates to.
- Open: handler granularity (one class per layer vs. one per
  layer-resource pair), error handling when a handler's backend is down,
  and whether the session handler should be user-extensible.
