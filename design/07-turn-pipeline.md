# The turn pipeline: a chain of chains

This note is the spine of the design. Everything layered (skills,
memories, prompt fragments, classifier rules, tools) is delivered by the
same mechanism: **an inbound message passes through the layers like
middleware; each layer exposes the same hook points, and at each hook it
runs its own inner chain of steps.** Any step may stop the whole thing
with an action or enrich a shared turn context and pass it on. See
`decisions/0008-layer-handler-chain.md` and
`decisions/0009-chain-of-chains.md`.

## The shape: a chain of chains

```
client ──HTTP/SSE──▶ API ──▶ ┌────────── outer chain: layers (middleware order) ──────────┐
                             │  global ▶ team:A ▶ team:B ▶ user ▶ session                 │
                             │     │                                                      │
                             │     └─ each layer = hook points, each hook = inner chain   │
                             │          on_message:         [gate, prompts, skills, tools, memories, rules]
                             │          on_tool_call:       [static rules]   (service prepends risk escalation + scope gate, appends model audit)
                             │          on_memory_candidate:[retag, accept/decline]   (restrictive mining rules run once, before routing)
                             │          on_turn_end:        [audit log]
                             └────────────────────────────────────────────────────────────┘
                                          ▼ model call sits between on_message and the outbound hooks
```

**Outer chain.** The layers, in trust order. The inbound message is passed
through them like middleware. This is the only chain the core walks
directly.

**Inner chains.** Each layer exposes the same **hook points**, and at
each hook point the layer runs its own ordered chain of **steps**. A step
is the unit of work: contribute a resource from one source, apply a
gate, transform the context. A layer with two skill repositories simply
has two skill steps in its `on_message` chain. A layer with no memory
store has no memory step.

**Hook points** (same for every layer, called by the core in outer-chain
order):

| Hook                  | When the core calls it                                  | Typical steps                                     |
|-----------------------|---------------------------------------------------------|---------------------------------------------------|
| `on_message`          | once, before the model runs                             | gates (rate limit, content policy), prompt fragments, skill discovery, tool definitions, memory search, rule loading |
| `on_tool_call`        | once per tool call, as the model emits it               | static audit rules, capability check; the service appends the model-based auditor after the last layer |
| `on_memory_candidate` | once per candidate, after the miner runs (off critical path) | mining rules, accept into this layer's store or decline |
| `on_turn_end`         | once, after the final assistant message                 | audit log, metrics, post-hoc flags                 |

There is no "outbound chain" as a return path. The last three hooks are
the outbound side; they run in the same outer order as `on_message`.

```
TurnContext {
  token: AccessToken              # sub, scopes, groups (04-auth-and-permissions.md)
  session: Session                # id, active_team, conversation, event bus
  inbound: UserMessage            # the message that started this turn
  # accumulators, start empty, filled by on_message steps in order:
  prompt:   {fragment_id -> PromptFragment}     # locked ids are final
  skills:   {name -> Skill}                     # locked names are final
  tools:    {name -> ToolDefinition}            # locked names are final
  rules:    [ClassifierRule]                    # in layer order; each layer's step reads its own
  memories: [ScoredMemory]                      # from every layer, merged later
  # outbound side:
  model_output: AssistantMessage | ToolCall[]
  verdicts:  {tool_call_id -> Verdict}
  candidates: [MemoryCandidate]
}

Step {
  kind: gating | contributing                   # declared by the class; drives the default fail mode
  run(ctx, hook_payload) -> Continue | Stop(Action)
}
ContributingStep(Step) {                        # the common case
  fetch(ctx, payload) -> result                 # I/O; may run concurrently across layers
  apply(ctx, result)                            # pure; applied in chain order; uses ctx.put
}
# gating -> fail closed; contributing -> fail open; `required: true` on a contributing step fails the turn instead
Layer {
  name: text
  hooks: {hook_name -> [Step]}                  # the inner chains
  run(hook_name, ctx, payload):
      for step in hooks[hook_name]:
          if (r := step.run(ctx, payload)) is Stop: return r
      return Continue
}

Action = Deny(reason) | Respond(text) | RequireApproval(request)
       | Accepted | Discarded                   # on_memory_candidate only: the candidate was consumed / dropped by this step
```

The core's turn loop:

```
run_turn(session, message):
  ctx = TurnContext(token, session, message)
  layers = chain_for(token)                                  # outer chain
  if (r := run_hook("on_message", layers, ctx)) is Stop: return emit(r.action)
  loop:                                                      # agent loop, may iterate
      out = model.complete(render(ctx))                      # streams deltas to client
      for call in out.tool_calls:                            # may run concurrently
          if (r := run_hook("on_tool_call", layers, ctx, call)) is Stop:
              handle(r.action)                               # deny / approval
          else: execute(call)
      if out.is_final: break
  run_hook("on_turn_end", layers, ctx)
  schedule_background: for c in miner.observe(ctx):
                           run_hook("on_memory_candidate", layers, ctx, c)

run_hook(name, layers, ctx, payload=None):
  for layer in layers:                                       # outer order
      if (r := layer.run(name, ctx, payload)) is Stop: return r
  return Continue
```

A `Stop` from an inner step stops the outer chain too. A gate is a gate
regardless of nesting; there is no "catch" at the layer boundary.

## Ordering and precedence

The outer chain runs **most trusted first**: global, then each team in
the token's `groups`, then user, then session. Inner chains run in their
configured order within the layer. This single order serves both jobs
the chain has:

- **Gating** (stop the chain): the most trusted layer runs first, so
  global policy can deny or demand approval before anyone lower gets a
  say.
- **Precedence** (who wins an override): later handlers overwrite earlier
  values in the accumulators, so *last write wins* gives user precedence
  over team over global. That is the shadowing rule from
  `02-layering-and-composition.md` expressed as pipeline order.
- **Locking**: an earlier handler marks a value `locked`; later handlers
  may not overwrite it. A locked global skill, prompt fragment, or tool
  therefore survives a same-name user copy. For classifier rules,
  "locked" means a locked `allow` shields the call from later layers'
  restrictions; restrictions themselves (`deny`, `require_approval`)
  accumulate across layers and need no lock (`08-walkthrough.md` §8b).
  Locking is the only way an earlier layer constrains a later one's
  contributions.

Two rules keep the chain honest:

1. **Only gating stops the chain.** A contributing step never returns
   `Stop`. A `Stop` in `on_message` prevents the model from running at
   all; in `on_tool_call` it prevents that call; in `on_memory_candidate`
   it discards that candidate. Steps must not stop merely because they
   have nothing to add.
2. **Steps never talk to each other.** They communicate only through the
   context. A step may read what earlier layers or earlier steps
   contributed but holds no reference to other steps or layers.
3. **The outbound hooks gate actions, never text.** Assistant text streams
   to the client at model speed and no step sees it before the client
   does. Text policy belongs in `on_message` (prompt fragments) or in
   `on_turn_end` (post-hoc flag events). A deployment that must filter
   text before release can add a buffering step and accept the latency
   knowingly.

## Standard steps, per resource

Steps are reusable classes configured with a source; the same
`SkillDiscoveryStep` class serves global with a git store and user with a
directory store. The standard set:

| Resource         | `on_message` step                                                | outbound-hook step                                      |
|------------------|------------------------------------------------------------------|---------------------------------------------------------|
| Prompt fragments | put fragments into `ctx.prompt` by id; skip ids already locked   | –                                                       |
| Skills           | put discovered skills into `ctx.skills` by name; skip locked     | –                                                       |
| Tools            | put tool definitions into `ctx.tools` by name; skip locked; loaded skills add `<skill>.<tool>` | –                                     |
| Classifier rules | append this layer's rules to `ctx.rules`                         | apply this layer's `tool_audit` rules to each tool call (restrictions accumulate, locked allow shields). Restrictive `memory_mining` rules (drop/threshold/redact/disable) from every layer run once, before routing; widening ones (retag, auto-accept) run in this layer's `on_memory_candidate` step |
| Memories         | append search results to `ctx.memories`                          | accept or decline `ctx.candidates` for this layer's store (routing policy) |
| Gating           | static deny/allow/require-approval on the *message* (rate limit, content policy, session frozen) | static deny/allow/require-approval on *tool calls*; final say on memory writes |

Notes:

- **Permissions build the outer chain and prune the inner ones.**
  `chain_for(token, session)` includes a team or user layer only if the
  token's scopes permit it (scope-to-layer table in
  `04-auth-and-permissions.md`). **The global layer is always in the
  chain**: its prompt fragments, rules, and gates are the policy baseline
  and need no scope; its skills, tools, and memories steps are skipped
  without the matching `use-global-*` scope. Within any layer, a step
  for a resource the token cannot use is skipped. A team layer produces
  one outer-chain entry per team in `groups`.
- **The model-based classifier is the last `on_tool_call` step.** After
  every layer's static rules have run, the shared `ClassifierEngine`
  reviews any call no rule decided, using the rules' prompt fragments as
  instructions. The service appends it after the last layer, so it
  cannot be shadowed.
- **The memory miner runs after `on_turn_end`**, off the critical path,
  and each candidate then passes through `on_memory_candidate`
  (`decisions/0007-memory-routing.md`). Team steps accept a candidate
  only with a positive signal (active team, auto-accept rule, user
  confirmation); the user step is the fallback; global has no accepting
  step for agent-driven writes.
- **Loaded skills act at their layer.** When a skill is loaded during a
  turn, its tools and prompt fragments are put into the context by the
  layer the skill came from, so a global skill's tool cannot outrank a
  user tool of the same name, and a user's fork of a global skill wins
  as usual.
- **The session layer is last.** Its steps contribute client-offered
  tools, ad-hoc instructions, temporary memories, and `active_team`.
  Being last it has the final override on unlocked values and the last
  chance to gate.

## Merging the accumulators before the model call

Handlers append; the core merges just before rendering:

- `ctx.prompt` → `PromptAssembler` groups by section, orders by priority,
  applies the token budget (drop `optional` first, never drop `locked`),
  renders with the deployment's `RenderStrategy`.
- `ctx.memories` → `MergeStrategy` (reciprocal rank fusion by default,
  layer boost optional) then `RecallPolicy` for count and budget.
- `ctx.skills` and `ctx.tools` → the tool list sent to the model:
  summaries of every skill plus every tool definition, already deduped by
  name through last-write-wins and locking.

These merge steps are strategies, not handlers, because they have no
layer: they operate on what all layers contributed.

## Execution model: chain order is merge order, not execution order

Steps never talk to each other, so their store lookups are independent.
The core therefore runs `on_message` in two phases:

1. **Gates first, sequentially.** Every layer's gating steps run in outer
   order. They are static checks (microseconds) and may `Stop` before any
   store is touched.
2. **Fetch concurrently, apply in order.** Every contributing step's
   fetch (memory search, skill discovery, prompt fragments, rules, tools)
   runs at once. The results are then applied to the accumulators in
   outer-chain order, then inner-chain order, so last-write-wins and
   locking behave exactly as if the steps had run sequentially.

Time-to-first-token is therefore the slowest single source, not the sum
of all sources. Per-layer timings are reported in the `context_ready`
event so slow sources are visible.

`on_tool_call` follows the same rule: several calls from one model
response are audited concurrently, and the audit for a call may start as
soon as its name and partial arguments are available, so it often
finishes before the model does.

## Streaming

- `on_message` runs once per user message and contains no model calls.
- The model call streams `assistant_delta` events as it goes. No step
  sees text before the client does.
- `on_tool_call` runs per call as the model emits it, so a denied call is
  reported before the model finishes. `tool_call_proposed` is emitted
  immediately with state `reviewing`; `tool_call_started` follows once
  the hook completes.
- A `Stop(RequireApproval)` emits `approval_requested`, parks that call,
  and resumes `on_tool_call` from the step after the asker when
  `approval_response` arrives. Full flow in `08-walkthrough.md` §5.
- `on_memory_candidate` runs after `turn_complete`. Clients must keep the
  event stream open for late `memory_created` / `memory_confirm` events
  or fetch them on the next turn.

What the client sees for a typical turn:

```
turn_started
context_ready            on_message done; per-layer timings
assistant_delta ...      model speed
tool_call_proposed       immediately, state=reviewing (+ static verdict if any)
tool_call_started        after on_tool_call
tool_call_finished
assistant_delta ...
turn_complete
memory_created ...       later, off the critical path
```

## Configuration view

The layer list is the outer chain. Under each layer, each hook lists its
steps in inner-chain order. Standard steps are referenced by name and
given a source; omitting a hook or a step means that layer contributes
nothing there.

```yaml
layers:                                  # order = outer chain = trust order
  - name: global
    on_message:
      - gate:     { adapter: rate_limit, per_subject: 60/min }
      - prompts:  { adapter: file, path: /etc/agent/global-prompt.md }
      - skills:   { adapter: git, url: git@github.com:org/shared-skills.git }
      - skills:   { adapter: git, url: git@github.com:org/legacy-skills.git }   # two sources, two steps
      - tools:    { adapter: builtin }
      - memories: { adapter: http, url: https://memory.example.internal, writable: false }
      - rules:    { adapter: file, path: /etc/agent/classifier-rules.yaml }
    on_tool_call:
      - static_rules: {}                 # evaluates rules this layer loaded
    on_turn_end:
      - audit_log: { adapter: http, url: https://audit.example.internal }
  - name: team                           # one outer entry per team id in the token
    on_message:
      - skills:   { adapter: git, url: git@github.com:org/team-{team_id}-skills.git }
      - memories: { adapter: http, url: https://memory.example.internal/teams/{team_id}, writable: true }
      - rules:    { adapter: http, url: https://policy.example.internal/teams/{team_id}/classifier }
    on_tool_call:
      - static_rules: {}
    on_memory_candidate:
      - route_to_team: {}                # accepts with positive signal only
  - name: user                           # templated per subject: ${user_root} = <users_root>/<token.sub>
    on_message:                          # (single-user dev mode may set users_root to ~/.agent and a fixed subject)
      - prompts:  { adapter: file, path: "${user_root}/prompt.md" }
      - skills:   { adapter: directory, path: "${user_root}/skills" }
      - tools:    { adapter: directory, path: "${user_root}/tools" }
      - memories: { adapter: sqlite_vec, path: "${user_root}/memory.db", writable: true }
      - rules:    { adapter: file, path: "${user_root}/classifier-rules.yaml" }
    on_tool_call:
      - static_rules: {}
    on_memory_candidate:
      - route_to_personal: {}            # fallback; accepts anything the token allows
  - name: session                        # built in: client tools, active_team, ad-hoc instructions
```

A solo deployment can list only `user` and `session`.

## Why this shape

- One mechanism to explain: "a message goes through the layers like
  middleware; each layer runs its own little chain at each hook; earlier
  layers can lock things." The per-resource precedence rules are
  consequences, not separate designs.
- Adding a layer is adding an entry. Adding a source to a layer is adding
  a step. Adding a resource kind is adding a step class and an
  accumulator on the context.
- Steps are small and reusable: the same step class runs in every layer
  with a different source, and a step is tested with a fake source and a
  hand-built context.
- Gating and contribution share one ordering, so there is no separate
  question of "which order do the composites consult layers in".
- Testing a layer is instantiating one handler with fake sources and a
  hand-built context.
