# Design patterns

The top-level pattern is a **chain of chains over a shared turn context**
(`07-turn-pipeline.md`): an outer chain of layers, each exposing hook
points, each hook running that layer's inner chain of steps. Every
layered resource (skills, memories, prompt fragments, classifier rules,
tools) is contributed by a step. This note records which classic pattern
does which job so the code has a shared vocabulary.

## Chain of Responsibility, twice: layers and steps

```
chain_for(token) -> [Layer]                    # outer: global, team:T (per group), user, session
layer.hooks[hook_name] -> [Step]               # inner: this layer's steps for that hook
step.run(ctx, payload) -> Continue | Stop(Action)
```

- **Outer chain** = layers, most trusted first. Walked by the core at
  four hook points: `on_message`, `on_tool_call`, `on_memory_candidate`,
  `on_turn_end`.
- **Inner chain** = the steps a layer runs at that hook, in configured
  order. Gating steps may `Stop`; contributing steps only append. A
  `Stop` anywhere stops everything.
- Precedence is chain order: later steps overwrite earlier unlocked
  values in the accumulators (`ctx.prompt`, `ctx.skills`, `ctx.tools`),
  so user beats team beats global. `locked` marks a value final.
- Permissions decide which layers are in the outer chain and which steps
  survive in each inner chain. A team layer yields one outer entry per
  team in the token's `groups`.
- The model-based classifier is a step the service appends after the last
  layer's `on_tool_call` chain, so it cannot be shadowed.
- Chain order is merge order, not execution order: contributing steps
  fetch concurrently and apply in order (`07-turn-pipeline.md`,
  Execution model).

Where one answer must win (loading a skill by name, resolving a tool by
name) the chain's last-write-wins on a keyed accumulator *is* the
resolution; nothing else resolves it. Where every layer contributes
(memories, prompt fragments, rules) the accumulator collects everything
and a merge strategy runs afterwards.

Two inner chains deserve naming because their steps make decisions:

- **Tool-call audit**: each layer's `StaticRulesStep` evaluates that
  layer's rules. `deny` and `require_approval` are restrictive and
  accumulate across layers; `allow` pre-empts the model audit but yields
  to any restriction unless locked, in which case it shields the call
  from later layers. The model-based auditor is the last step.
- **Memory write routing**: a candidate is offered to each team layer's
  `RouteToTeamStep` (accepts only with a positive signal: active team,
  auto-accept rule, user confirmation, and no other team in the
  candidate's provenance), then to the user layer's `RouteToPersonalStep`
  (accepts anything the token allows, recording any declined team
  suggestion for promotion). Global has no accepting step.

## Factory: building layers, steps, and sources

```
SourceFactory.create(kind, adapter_name, config) -> SkillStore | MemoryStore | PromptSource | RuleSource | ToolSource
StepFactory.create(step_name, source, config) -> Step
LayerFactory.for_entry(layer_config, token?) -> Layer     # builds every hook's inner chain
ToolRunnerFactory.for(impl) -> ToolRunner
ClassifierEngineFactory.create(config) -> ClassifierEngine
```

- One adapter registry keyed by adapter name (`git`, `directory`,
  `sqlite_vec`, `http`, `file`, `inline`, `builtin`) and one step registry
  keyed by step name (`prompts`, `skills`, `tools`, `memories`, `rules`,
  `gate`, `static_rules`, `route_to_team`, ...). Both are extension
  points; the core imports neither.
- `LayerFactory` reads one entry of the layer list, builds each step's
  source through `SourceFactory`, each step through `StepFactory`, wraps
  them in decorators, and returns a `Layer`. The team entry is built
  lazily per team id the first time a token with that group appears,
  then cached.
- Abstract-factory flavour: a `Backend` can supply several sources from
  one location (a git repo holding skills, prompts, and rules).

## Decorator: cross-cutting behaviour

On sources (inside a step):

- `CachingSource(ttl)` for remote adapters.
- `PermissionFilteredSource(required_scope)` returns nothing when the
  token lacks the scope; this is how a resource is "silently absent".
- `AuditedSource(log)` records reads and writes with subject and layer.
- `ReadOnlySource` enforces `writable: false` regardless of adapter.

On steps:

- `TimedStep` records latency per step; rolled up per layer into the
  `context_ready` event.
- `FailOpenStep` / `FailClosedStep` decide what happens when a step's
  backend is unreachable (default: fail closed for gating steps, fail
  open for contributing steps; configurable per step).
- `DryRunStep` logs what a gating step would have stopped, stops nothing.
  Used to calibrate new global rules.

On tool runners: `SandboxedRunner`, `CapabilityCheckedRunner`,
`TimeoutRunner`, `RateLimitedRunner`, `AuditedRunner`. On the classifier
engine: `DryRunClassifier`, `AuditedClassifier`. Wrap order is set by
the factory and can be mandatory for less trusted layers.

## Strategy: the merge steps

Merge steps run after `on_message`, on what every layer contributed.
They have no layer, so they are strategies rather than steps.

- `MergeStrategy` for `ctx.memories`: reciprocal rank fusion (default),
  layer-boosted rank, cross-encoder rerank.
- `RenderStrategy` for the assembled prompt: plain text, tagged sections,
  markdown headers; chosen per model or deployment.
- (No verdict merge strategy is needed: restrictive rules from any layer
  accumulate and widening rules yield, so peer-team rules never conflict.
  See `08-walkthrough.md` §8b.)
- `ArgumentValidationStrategy` for tool calls: strict reject vs. coerce and
  warn.

## Pipeline: budgets and recall

Small ordered steps applied to a list, each pluggable:

- Recall: `ctx.memories` → filter (age, kind, pinned) → token budget → cite.
- Prompt budget: drop `optional` fragments first, never drop `locked`.
- Memory mining: miner candidates → layered `memory_mining` rules (locked
  first) → routing.

## Observer: the event stream

Steps, the classifier, and tool runners publish events on the
session's bus (`memory_created`, `skill_loaded`, `skill_shadowed`,
`tool_call_proposed`, `approval_requested`, `memory_confirm`). The API
layer forwards them to the one owning client; internal listeners
(metrics, audit log) subscribe to the same bus.

## Summary table

| Concern                                  | Pattern                          | Where precedence lives                 |
|------------------------------------------|----------------------------------|----------------------------------------|
| Processing a turn                        | Chain of chains (layers, then steps) | outer: global → team → user → session; inner: configured |
| Who wins a same-name skill/tool/fragment | last write wins in the chain     | later step; `locked` makes earlier final |
| Gating a message or tool call            | `Stop(Action)` from any step     | most trusted layer runs first          |
| Tool-call audit                          | `on_tool_call` inner chains      | restrictions accumulate across layers; locked allow shields; model last |
| Memory write routing                     | `on_memory_candidate` inner chains | team with signal, else user          |
| Building layers, steps, and sources      | Factory / Abstract Factory       | layer list in config                   |
| Caching, permissions, audit, fail modes  | Decorator                        | wrap order set by factory              |
| Merging memories, rendering prompt       | Strategy                         | per deployment / model                 |
| Budgets, recall, mining                  | Pipeline                         | step order                             |
| Running a tool                           | Factory + Decorator              | wrap order                             |
| Delivering events                        | Observer                         | n/a                                    |

## What this buys

- Adding a layer is adding an entry to the layer list; the factory builds
  its steps. Removing one (solo deployment) is deleting the entry.
- Adding a source to a layer is adding a step. Adding a resource kind is
  adding a step class and an accumulator on `TurnContext`.
- Every layer can use different adapters, and one adapter can back
  several layers.
- The precedence, locking, and permission rules are stated once, for the
  chain, instead of once per resource.
