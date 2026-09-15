# 0009 – Chain of chains: layers are middleware, each layer runs inner chains at hook points

**Status:** accepted (2026-09-14). Refines 0008.

## Context

Decision 0008 made the turn a chain of layer handlers with an inbound and
an outbound method. Two things were unclear: whether "outbound" was the
return path of "inbound" (it was not), and how a single handler should be
structured when a layer has several sources for one resource or wants to
omit a resource entirely. The intended model was a **chain of chains**:
the inbound message passes through the layers like middleware, and each
layer has hooks and chains of its own.

## Decision

- The **outer chain** is the layers in trust order (global, team per
  group, user, session). The core walks it at four **hook points**:
  `on_message` (once, before the model), `on_tool_call` (per call),
  `on_memory_candidate` (per candidate, off critical path),
  `on_turn_end` (once).
- Each layer runs an **inner chain of steps** at each hook. A step is the
  unit of work: one source's contribution, one gate, one transform. Two
  sources for a resource are two steps. No source means no step.
- `Stop` from any step stops the whole turn's hook. There is no return
  path; the outbound hooks run in the same outer order as `on_message`.
- Chain order is **merge order, not execution order**: gating steps run
  first sequentially; contributing steps fetch concurrently and apply in
  chain order.
- The outbound hooks gate **actions only** (tool calls, memory writes),
  never assistant text, which streams to the client unseen by any step.
- Configuration lists layers, then hooks, then steps.

## Consequences

- `LayerHandler` is replaced by `Layer` + `InnerChain` + `Step`; step
  classes are reusable across layers with different sources.
- Handler granularity and multi-source layers are resolved.
- Time-to-first-token is bounded by the slowest single source; a
  `context_ready` event reports per-layer timings.
- Memory events may arrive after `turn_complete`; clients must handle
  that.
- The model-based audit is a step the service appends after the last
  layer's `on_tool_call` chain.
