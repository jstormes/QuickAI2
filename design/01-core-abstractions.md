# Core abstractions

This note names the main interfaces ("ports") the core depends on and the
adapters expected behind each one. Names are placeholders; pseudocode is
language-neutral.

**How they fit together** is in `07-turn-pipeline.md`: a turn passes
through the layers (global, team, user, session) like middleware. Each
layer runs its own inner chain of steps at four hook points, and every
step either contributes its source's skills, memories, prompt fragments,
classifier rules, or tools to a shared `TurnContext`, or stops the turn
with an action. The store and source interfaces in this note are the
backends a step reads from.

```
                +---------------------------------------------+
  Front ends    |  CLI   |   Web   |   Desktop/Mobile app     |
                +----------------------+----------------------+
                                       |  HTTP + streaming (SSE / WebSocket)
                +----------------------v----------------------+
                |              Web API layer                  |
                |  sessions (one per client) · auth · stream  |
                +----------------------+----------------------+
                                       |  event bus
                +----------------------v----------------------+
                |                  Agent Core                 |
                |  on_message ▶ model ▶ on_tool_call ▶ on_turn_end │
                +----------------------+----------------------+
                                       |  TurnContext (token, session, accumulators)
                +----------------------v----------------------+
                |  outer chain: global ▶ team(s) ▶ user ▶ session   |
                |  each layer: inner chain of steps per hook, |
                |  each step backed by one source:            |
                |  SkillStore · MemoryStore · PromptSource ·  |
                |  ClassifierRuleSource · ToolSource          |
                +----------------------+----------------------+
                                       |
                    adapters: git · directory · sqlite_vec · http · file · builtin
                                   +---------------------------+
                                   |  Support: ClassifierAgent |
                                   |  · tool-call audit        |
                                   |  · background memory gen  |
                                   +---------------------------+
```

## 1. Identity: the access token

There is no separate identity abstraction. The validated OAuth2 access
token *is* the caller's identity, and it is passed to every store,
step, and policy that needs to know who is asking.

```
AccessToken {
  subject: text               # `sub` claim: the user or service account
  scopes: {text}              # `scope` claim: permissions, e.g. use-global-skill
  groups: [text]              # `groups` claim: team ids (claim name configurable)
  expires_at, issuer, raw     # standard JWT fields, kept for refresh/audit
}
```

- The API layer validates the bearer token (`04-auth-and-permissions.md`)
  and hands the resulting `AccessToken` to the core. Nothing downstream
  re-validates it or sees the raw header.
- **Which layers apply is a rule, not state.** The layer list lives in
  config. A (store, layer) pair participates in a request when the token
  holds the matching scope: the global skill store is queried only if
  `use-global-skill` is present; a team store for team T is queried only
  if `use-team-skill` is present *and* T is in `groups`.
- Stores receive the `AccessToken` and decide which of their contents
  apply. A global skills repo ignores the subject; a user memory store keys
  on it.
- Naming: OAuth2 "scopes" are called *permissions* in prose and `scopes`
  in code. (Earlier drafts had a resolved `Scope` / `LayerContext` object;
  it was removed because it only restated the token's claims.)
- Layers give us precedence for free: later layers override earlier ones
  when two sources supply the same key.

## 2. Memory

```
Memory {
  id: MemoryId              # includes source id, stable across reads
  kind: enum                # user | feedback | project | reference | episodic | ...
  title, body: text
  tags: [text]
  links: [MemoryId]         # graph edges for "organized"
  layer: Layer              # which layer it belongs to
  owner: PrincipalId        # the subject it was written for (user layer) or on behalf of (team layer)
  confidence: float         # from the candidate; raised on reinforcement
  created_at, updated_at, used_at, source: SourceId
  pinned: bool              # always recalled; last to be dropped
  supersedes?: MemoryId     # set when the miner said this replaces an earlier memory
  superseded_by?: MemoryId  # excluded from recall; kept for history
  suggested_audience?: Audience   # a declined team suggestion, kept for promotion
  promoted_from?: MemoryId  # set on the team copy made by promotion
  embedding?: vector        # optional, store-specific
}

MemoryStore (read side)
  search(query, token, opts) -> [ScoredMemory]   # semantic + filter
  get(id) -> Memory
  list(token, filter) -> [Memory]
  touch(id, used_at)                              # citation reinforcement

MemoryWriter (write side, optional per store)
  put(memory) -> MemoryId
  update(id, patch)
  delete(id)

Each layer's MemorySearchStep calls its own MemoryStore during on_message
and appends results to ctx.memories; a MergeStrategy fuses them afterwards.
Writes go through the `on_memory_candidate` routing steps (below).
```

Key decisions to keep the storage loosely coupled:

- **Embedding is the store's problem, not the core's.** The core hands the
  store text; a store may embed with any model, do keyword search, or
  delegate to a remote service. A store may also accept a pre-computed
  embedding to allow sharing one embedder across stores.
- **Organization is expressed as data** (kind, tags, links, layer), not as
  directory structure. A file-based store may map these to folders, but that
  is an adapter detail.
- **Recall is a pipeline stage**, not a store feature: each layer's
  `MemorySearchStep` fills `ctx.memories`, `MemoryMergeStep` fuses by
  rank (RRF) and dedupes across layers, then `RecallPolicy` decides how
  many and which go into the context window. Full flow in
  `08-walkthrough.md` §10.

### Memories are personal or team, never global, when written by the agent

Memories are personal by nature (`decisions/0007-memory-routing.md`).
Agent-driven writes, whether from the classifier's miner or from an
explicit user request, target only the **personal** or a **team** layer.
A global memory layer may exist but is read-only from the agent's point
of view and curated by an admin outside the agent loop.

### Memory routing policy

Routing is the `on_memory_candidate` hook: each team layer runs a
`RouteToTeamStep`, the user layer runs `RouteToPersonalStep`, and the
candidate walks them in chain order (`08-walkthrough.md` §6d). A team
step accepts only if the token allows team writes for its team and a
positive signal exists (the session's active team, an auto-accept rule
of that team, or user confirmation) and the candidate's provenance does
not include another team. The user step accepts anything the token
allows and records a declined team suggestion. A candidate no step
accepts is dropped.

- **Private by default.** The user step is the last, fallback step.
- **`session.active_team`** is set by the client (CLI flag, web dropdown,
  app setting) and tells the router "team-relevant things go here". With
  no active team, everything is personal unless a team rule or the user
  says otherwise.
- **Team classifier rules** (§6) let a team declare what it auto-accepts,
  e.g. "candidates tagged for us that mention project X". Absent such a
  rule, and absent an active team, a team-tagged candidate needs user
  confirmation or is downgraded.
- **Downgraded candidates keep their suggestion** as `suggested_audience`
  on the personal memory. They can be promoted later (§ API: promote),
  which copies the memory into the team layer with provenance and leaves
  the personal copy in place.

## 3. Skills

```
Skill {
  id: SkillId
  name, description: text        # description is what the agent matches on
  instructions: text             # loaded lazily
  resources: [ResourceRef]       # scripts, templates, reference files
  tools: [ToolDefinition]        # tools this skill brings with it (may be empty)
  prompt_fragments?: [PromptFragment]  # optional additions to the system prompt
  triggers?: [text]              # optional hints for discovery; a match marks the skill "suggested"
  auto: bool                     # load automatically when a trigger matches (still rule-gated)
  pinned: bool                   # always listed in the skill index
  layer, source
  version: text                  # content hash or explicit version from the store
  derived_from?: SkillRef        # {id, layer, version} of the skill this was forked from
  locked: bool                   # if true, later (less trusted) layers may not override it
  auto, pinned: bool             # see §3 discovery
  capabilities: [Capability]     # what the skill may need (tools, network); must cover its tools' `requires`
}

SkillSummary {                   # what discovery returns; cheap
  id, name, description, tool_names: [text], layer, locked, version, derived_from?, triggers, auto, pinned
  shadows?: SkillId, stale?: bool, suggested: bool, loaded: bool     # filled per turn
}

ToolDefinition {
  name: text                     # namespaced on registration: <skill>.<name>
  description: text
  input_schema: JSONSchema
  risk: enum                     # none | low | medium | high
  execute_on: enum               # server | client
  mode: enum                     # foreground | background | auto  (long-running tools: 08-walkthrough.md §13)
  on_session_end?: enum          # continue | cancel  (background only)
  origin: { kind: source | skill | client, skill_name?, skill_layer? }   # where it came from; rules may match on it
  requires: [Capability]         # declared needs; clipped to the layer ceiling into `granted` at registration
  granted: [Capability]
  effective_risk: enum           # risk after per-layer escalation (08-walkthrough.md §8c)
  implementation: ToolImpl       # see below
}

ToolImpl (one of)
  script   { runtime: text, entry: ResourceRef }   # e.g. a script in resources/
  http     { url, method, auth? }                   # call an external service
  builtin  { ref: text }                            # provided by the host; only allowed for origin.kind == source in a
                                                    # layer listed in config.tools.builtin_refs_allowed_from (default [global])
  client   { }                                      # client-side; see web API

SkillStore
  discover(query, token) -> [SkillSummary]   # cheap; store-side index over name+description+triggers
  load(id) -> Skill                          # full body; version-pinned per session once loaded
  resource(id, path) -> bytes                # size-capped; path confined to the skill folder
  load_version(id, version) -> Skill?        # optional; git stores can
  has(name) -> bool
  summary(id) -> SkillSummary
  version(id) -> text

SkillWriter (optional per store)
  put(Skill) -> SkillId, update(id, patch), delete(id)     # forking is API-layer logic on top of put

Each layer's SkillDiscoveryStep calls its own SkillStore during on_message
and puts summaries into ctx.skills by name; later layers overwrite unlocked
names (shadowing), locked names are final.
```

- Two-phase (discover then load) keeps the context window small: only the
  summary of every skill is visible, the body is loaded on use.
- Skills are read-only from the agent's perspective by default. Authoring
  is done by whoever maintains that store (a git repo, a CMS, a user's
  local folder). A `SkillWriter` can exist but is not required.
- Skill format: likely Markdown with front matter, because it is
  human-editable and git-friendly. The store interface does not care.

### Personal overrides of shared skills

A user may take a global (or team) skill, modify it, and use their copy
in place of the original. This is the shadowing rule made into a
workflow (`decisions/0006-personal-skill-overrides.md`).

- **Fork, not patch.** The override is a full copy of the skill placed in
  the user's layer under the *same name*, with `derived_from` recording
  the base skill's id, layer, and version. Because the same name in a
  later layer wins, the copy shadows the base everywhere: discovery, load, its tools,
  and its prompt fragments. The base remains loadable by full id.
- **Permissions.** Forking needs `use-<base layer>-skill` to read the base
  and `create-personal-skill` to write the copy. A team override
  (`create-team-skill`) works the same way one layer earlier.
- **Locked skills cannot be overridden.** A global skill with
  `locked: true` is excluded from shadowing: a same-name skill in a later,
  less trusted layer is ignored with a warning, and the fork endpoint refuses.
  This is how policy-bearing skills stay authoritative.
- **Staleness.** When the base skill's version changes, the fork's
  `derived_from.version` no longer matches. Discovery marks the fork
  `stale`; the API exposes a diff between the fork's base version and the
  current base so the user can re-merge or re-fork. The fork keeps
  working meanwhile.
- **Tools travel with the fork.** The copy carries the base's tool
  definitions; the user may edit, add, or drop them. Capability limits
  for the user layer still apply, so a fork cannot grant itself more than
  a user-authored skill could have.
- **Unfork.** Deleting the personal copy restores the base. No special
  operation is needed.

### Skills define their own tools

A skill is a **mini-plugin**: instructions plus, optionally, the tools
those instructions rely on and prompt fragments to accompany them.

- **Registration.** When a skill is loaded into a session, its tools
  are put into `ctx.tools` (§7) by the skill's own layer under a
  namespaced name (`<skill>.<tool>`) and are included in the tool
  definitions sent to the model. When the skill is unloaded, the tools go
  away. Tools are therefore per-session in visibility even though the
  skill itself is a shared resource.
- **Dispatch.** The core's tool dispatcher does not care whether a tool
  came from a layer's tool source or from a skill; it resolves by name
  through the registry and routes to the resulting `ToolImpl`.
  Server-side impls run inside the service (script, http, builtin).
  Client-side impls are sent to the owning client as `tool_call_proposed`
  with `execute_on: client`.
- **Classifier audit.** Skill-defined tools go through the same tool-call
  audit as built-in tools. The declared `risk` feeds the approval policy.
  Because skills can come from less trusted maintainers (a global repo maintained
  by someone else), the runtime may raise the effective risk of a tool
  based on its skill's layer.
- **Capabilities.** A skill's `capabilities` must cover what its tools need
  (network, filesystem, subprocess). The service enforces this at the
  `ToolImpl` boundary. A skill store, or the service config, can restrict
  which capabilities a layer is allowed to grant.
- **Name collisions.** Because tool names are namespaced by skill, two
  skills may both define `search`. Shadowing rules for skills (the later
  layer wins on the same skill name) apply to the whole skill, tools included.
- **Discovery stays cheap.** `SkillSummary` carries tool names and flags
  only; full schemas and instructions are loaded with the skill body.

## 4. System prompt sources

```
PromptFragment {
  id, source, layer
  priority: int
  section: enum   # identity | policy | project | skills | tools | user_prefs | memories | session
  body: text
  locked: bool    # the earlier (more trusted) layer wins for this id if true
  optional: bool  # may be dropped first under token budget
  shrinkable: bool # assembler may render a smaller version before dropping (derived fragments)
  condition?: predicate over (token, session)   # fixed fields: active_team, client_kind, groups, scopes, time window
}

PromptSource.fragments(token, session) -> [PromptFragment]

PromptAssembler (merge step, after on_message)
  assemble(ctx, budget) -> ordered text grouped by section, budgeted, rendered
```

- Each layer's PromptFragmentsStep contributes fragments to `ctx.prompt`
  during on_message (`07-turn-pipeline.md`); `PromptAssembler` is the
  merge step that runs afterwards.
- Every layer contributes fragments. A fragment id supplied by a later
  layer overrides the same id from an earlier layer, unless the earlier one
  is `locked`, in which case the earlier (more trusted) layer wins. Distinct
  ids accumulate. Output is grouped by section; within a section, priority
  descending, then the earlier layer first, then id.
- Skills may contribute fragments while loaded (`Skill.prompt_fragments`).
- Rendering (plain text vs. tagged sections) is a strategy chosen per
  deployment or model.
- Recalled memories and the skill index enter as *derived* fragments
  generated by the assembler, so one token budget covers everything.
  Full flow in `08-walkthrough.md` §9.

## 5. Front ends via the streaming web API

The core does not expose a UI. It exposes a **web API with streaming**
(details in `03-web-api.md`), and every front end is a client of it.

```
Client -> API : user_message, tool_result, approval_response, cancel
API -> client : assistant_delta, assistant_message,
                           tool_call_proposed, approval_requested,
                           memory_created, error, turn_complete
```

- CLI, web page, and desktop/mobile app all speak the same protocol. They
  differ only in rendering.
- Every client owns exactly one session; sessions are not shared between
  clients. What *is* shared across sessions is the memory stores, the
  skill stores, and the classifier. Running CLI, web, and app together
  means three private sessions drawing on the same shared memories,
  skills, and classifier.
- Approval flows (see classifier) are modeled as events so any client can
  render an "allow this?" prompt in its own way and answer it over the API.
- Internally the core still has a `FrontendAdapter`-shaped boundary (an
  event bus). The web API is the one adapter shipped by the framework; an
  in-process adapter can exist for tests and embedding.

## 6. Classifier support agent

A second, smaller model call that runs alongside the primary agent. The
classifier is a **shared, service-level component**, not a per-session
object: one classifier (configuration, model, prompt scaffolding, audit
log) serves every session. It is invoked with a session's conversation
window as input but holds no per-session state of its own. This lets it
be tuned, replaced, or scaled independently of sessions, and lets its
verdict log span all sessions for calibration.

### Role A: tool-call audit

```
ClassifierEngine.audit(AuditInput) -> AuditOutput
  AuditInput  = { window, call, definition, provenance, prior_calls, instructions }
  AuditOutput = { p_requested, effect_summary, reason, mismatch? }
```

- Runs between "model proposed a tool call" and "tool executes", as the
  last `on_tool_call` step; the step applies the configured thresholds to
  turn `p_requested` into a Verdict (allow / deny / ask_user) and then
  executes, denies, or emits `approval_requested`.
- Inputs: the recent conversation window, the proposed call, its
  definition and provenance, and the layered instruction fragments.
- Must be cheap and fast; it sits on the critical path. Layered static
  rules run first and can decide without a model call; a deployment may
  run rules-only.

### Role B: background memory generation

```
ClassifierEngine.mine(turns, hints, instructions) -> [MemoryCandidate]
ClassifierEngine.summarise(session) -> MemoryCandidate        # kind=episodic, at session end

MemoryCandidate {
  kind, body, rationale, confidence, tags, links
  audience: personal | team:<id>     # the miner's *suggestion*, not a decision
  supersedes?: [MemoryId]            # only the miner may declare a contradiction
  provenance: { teams: [team_id] }   # which teams' skills/memories the turn drew on
  hash, source_turn                  # idempotency and attribution
}
```

- Runs **off** the critical path after each turn (or batched).
- Produces candidates with kind, body, a "why this is worth keeping", and
  a suggested audience. To suggest an audience the miner sees: the token's
  `groups`, the session's `active_team` (if set), which team skills were
  loaded this turn, which recalled memories came from a team layer, and
  explicit mentions in the conversation.
- **The miner never picks a store.** Candidates go through the layered
  mining rules and then the routing steps (§2), which decide the
  layer with personal as the fallback. Private by default: a wrong guess
  can only keep something personal, never leak it to a team.
- Emits `memory_created` events (with the layer chosen) so the front end
  can show what was saved and where.

### Rules are layered; the engine is shared

The engine (model, prompt scaffolding, verdict log) is one shared
component. What it enforces comes from a `ClassifierRuleSource` per layer,
following the same layered pattern as skills, memories, and prompts
(`05-design-patterns.md`). Static rules are evaluated per layer in chain
order: `deny` and `require_approval` accumulate across layers, an
`allow` pre-empts the model audit but yields to any restriction unless
it is locked, and the model-based judgment is the final step
(`08-walkthrough.md` §8b). Memory-mining policy is a
pipeline of layered rules applied to the miner's candidates before they
reach the memory write chain. The classifier's own system prompt is
assembled by the same `PromptAssembler` as the agent's.

### Why one "classifier" for both roles

Both jobs are "look at the conversation and make a structured judgment".
Sharing prompt scaffolding, model config, and observability keeps this
simple. They should still be separately enable-able.

## 7. Tools

Tools follow the same layered pattern as everything else
(`05-design-patterns.md`). There is no fixed set of "core tools".

```
ToolSource.tools(token) -> [ToolDefinition]      # one per layer

ctx.tools (accumulator on the TurnContext)
  filled by each layer's ToolDefinitionsStep from its ToolSource plus tools of skills
  loaded at that layer; later layers overwrite unlocked names, locked
  names are final. What the model sees is ctx.tools after the chain.

ToolRunnerFactory.for(impl) -> ToolRunner      # script | http | builtin | client
ToolRunner.run(definition, args) -> result     # wrapped by decorators:
                                               # sandbox, capability check,
                                               # timeout, rate limit, audit
```

- Built-in tools are simply the global layer's tool source.
- A layer may lock a tool definition so later, less trusted layers cannot
  shadow it.
- A client may contribute a session-layer tool source at session creation
  (tools it can execute locally).
- `ToolDefinition` is the type introduced in §3; skills and tool sources
  share it.

## 8. Write side of sources

Every source kind has an optional writer port. A store without one is
read-only through the API and is maintained outside the framework.

```
SkillWriter   { put(Skill) -> SkillId,  update(id, patch), delete(id) }
ToolWriter    { put(ToolDefinition), update, delete }
PromptWriter  { put(PromptFragment), update, delete }
RuleWriter    { put(ClassifierRule), update, delete }
MemoryWriter  { put(Memory), update, delete }             # §2
```

Git-backed writers: each write is one commit on the configured branch
with author `<subject> via agent` and a message naming the API
operation; `push` happens per write (default) or batched per
`config.sources.git.push_interval_s`. A non-fast-forward on push is
retried once after `fetch`; a merge conflict returns 409 `conflict` and
leaves the working tree clean. Directory and file writers write
atomically (temp file + rename). HTTP writers `PUT` and surface the
service's status code.

## 9. Asset store

```
AssetStore
  put(owner, session_id, bytes, media_type, ttl) -> AssetRef    # ref = opaque 128-bit random id, never derived from content
  get(ref, token) -> bytes | NotFound                          # owner must match token.subject; no listing across owners
  delete(ref, token); sweep()                                  # TTL enforced by a sweeper
```

Used for truncated tool output (§12g of the walkthrough), attachments,
and job results. Refs are meaningful only to their owner; a ref from
another owner is `NotFound`, never `Forbidden`, so refs do not leak
existence.

## 10. Identifiers

| Id            | Format                                              | Uniqueness                 |
|---------------|-----------------------------------------------------|----------------------------|
| session, turn, approval, job, asset, tool_call | opaque random (≥ 96 bits), prefixed `s_`, `t_`, `ap_`/`mc_`, `j_`, `a_`, `c_` | global |
| MemoryId      | `<layer>/<store-local id>`; layer is `global`, `team:<team_id>`, `user`, `session` | per store; the layer prefix makes it global |
| SkillId       | `<layer>:<name>` where `<layer>` is `global`, `team:<team_id>`, `personal`, `session`; `<name>` is `[a-z0-9-]+` | per layer |
| tool name     | `<name>` from a tool source, `<skill>.<name>` from a skill, client tools as declared | per session accumulator |
| fragment id, rule id | `[a-z0-9-]+`, unique within a source; the same id in two layers is an override | per layer |
| team_id       | from the IdP; must not contain `:` or `/`           | per IdP                    |
| subject       | the token's `sub`; used verbatim in `${user_root}` after path-safe encoding | per IdP |

## 11. Model client

```
ModelClient
  complete(ModelRequest) -> Stream<ModelEvent>     # always streaming
  count_tokens(ModelRequest) -> int
  capabilities() -> {context_window, max_output, structured_output, tool_use, parallel_tools, caching, thinking, forced_tool_choice}
```

One provider behind one interface, kept thin so the framework is not
tied to a vendor. The core renders a `ModelRequest` from the turn context
with stable-first system blocks (so adapters can place cache
breakpoints), a deterministic tool list, and an append-only history in
which assistant content is stored verbatim. Adapters: Anthropic (the
reference), OpenAI-compatible, local, and a scripted fake for tests. The
classifier engine uses its own `ModelClient` instance, usually with a
smaller model. Full mapping in `08-walkthrough.md` §16.
