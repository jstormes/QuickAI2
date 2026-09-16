# Glossary

One meaning per term. Where a word was used two ways in earlier drafts,
the preferred term is given and the other is retired.

| Term | Definition | Defined in |
|---|---|---|
| AccessToken | The validated OAuth2 token (`sub`, `scopes`, `groups`) passed through the system as the caller's identity; has pure helpers `allows`, `allows_any`, `with_team`. | `01` §1, `08` §2 |
| accumulator | A keyed or list-valued field on the turn context that steps append to (`prompt`, `skills`, `tools`, `rules`, `memories`). Precedence is settled at `put` time. | `07` |
| active team | A session attribute naming one of the token's groups; the main positive signal for routing a memory to that team. | `08` §11a, §6d |
| adapter | The concrete implementation behind a source or store (git, directory, sqlite, http, builtin). Registered by name; the core never imports one. | `05` Factory |
| asset | A binary or large text blob (truncated tool output, attachment, job result) held in the asset store under an opaque owner-bound ref. | `01` §9 |
| audience | The suggested destination of a memory candidate: `personal`, `team:<id>`, or `session` (scratch). Never a store. | `08` §6b, §14d |
| AuditStore | The single service-level port every audit record goes through (`write`, `query`, `retention`); `AuditLogListener` and direct `audit.write` calls are its only writers. Retired: per-layer `AuditLogStep`. | `01` §8, `04` |
| audit (model audit) | The classifier's model-based review of a tool call, the last `on_tool_call` step. Preferred term: **model audit**. | `08` §8d |
| audit (static rules) | Per-layer evaluation of `deny` / `require_approval` / `allow` rules. Preferred term: **static rules**. | `08` §8b |
| audit log / audit record | The append-only store of security-relevant records, read with `read-audit`. Preferred term: **audit record**. | `04` |
| background job | A tool call that returned a handle and keeps running after the turn; owned by the user, delivered back through the pipeline. | `08` §13 |
| capability | What a tool needs or a skill declares (`fs_read`, `network`, `secrets`, `client(kind)`…), clipped to the layer ceiling into `granted`. Not a permission. | `08` §12a |
| classifier | The shared support engine with three roles: model audit of tool calls, memory mining, session summary. Implementations: model, rules-only, remote. | `01` §6, `08` §16e |
| consuming step | A step in `on_memory_candidate` only; may return `Stop(Accepted)`, `Stop(Discarded)`, or `Stop(RequireApproval)` to consume the candidate; fails open (logs, queues the candidate, continues). | `07`, `08` §1 |
| ceiling | The most capability a tool from a given layer may hold; configured per layer. | `08` §12a |
| chain builder | `chain_for(token, session)`: builds the outer chain for a token (global always, team per group, user if scoped, session last). | `08` §2 |
| contributing step | A step that fetches from its source and applies to an accumulator; never stops the chain; fails open by default. | `07` |
| decided | Per-call audit state: some rule or the model audit has produced a verdict; the model audit is skipped. | `08` §8b |
| derived fragment | A prompt fragment the assembler generates rather than loads: the recalled memories block and the skill index. | `08` §9c |
| earlier layer / later layer | Position in the outer chain. Earlier = more trusted, runs first, may lock. Later = less trusted, runs later, overrides unlocked values. Retired: "lower/higher layer". | `07` |
| effective risk | A tool's declared risk after per-layer escalation; what rules and scope gates match on. | `08` §8c |
| fork | A copy of a shared skill placed in a later layer under the same name, recording `derived_from`; shadows the base for that layer. | `08` §7 |
| gate | Three named things: the `GateStep` on messages (rate limit, content policy, frozen), the service `ScopeGateStep` on tool calls, and client gates (declarative restrictive rules in the session layer). Say which. | `08` §3a, §8c, §14c |
| gating step | A step that may return `Stop(Deny \| Respond \| RequireApproval)`; fails closed by default. Gating and consuming steps are the only ones that stop a hook. | `07` |
| group / team id | An entry in the token's `groups` claim; each yields one team-layer entry in the outer chain. | `04` |
| hook point | One of four places the core walks the outer chain: `on_message`, `on_tool_call`, `on_memory_candidate`, `on_turn_end`. | `07` |
| inner chain | The ordered steps a layer runs at one hook point. | `07` |
| layer | One entry in the outer chain (`global`, `team:<id>`, `user`, `session`), built from one config entry; owns an inner chain per hook. | `07` |
| locked | On a skill, tool, fragment, or rule: an earlier layer marked it final; later layers cannot override it. For rules, only a locked `allow` has effect (it shields). | `07`, `08` §8b |
| memory candidate | What the miner proposes: kind, body, rationale, confidence, audience, provenance; routed by the `on_memory_candidate` steps. | `08` §6b |
| MemoryMerge | The strategy (not a step) the core invokes after `on_message` to rank-fuse `ctx.memories` across layers and dedupe; default reciprocal rank fusion. | `08` §10c |
| miner / mining | The classifier's memory role: proposing candidates after a turn, off the critical path. | `08` §6 |
| outer chain | The layers in trust order, walked by the core at each hook point. | `07` |
| permission / scope | Synonyms: an OAuth2 scope string on the token (`use-team-skill`, `run-server-tools`). Never a capability. The only other "scope"-like word is `remember_match` on approvals. | `04` |
| post-model hooks | `on_tool_call`, `on_memory_candidate`, `on_turn_end`: the hooks the core walks after the model call, in the same outer order as `on_message`. Retired: "outbound hook / outbound chain". | `07` |
| personal (scope word) | The word used in scope strings for the **user layer** (`use-personal-memory`). The layer itself is always called `user`. | `04` |
| PrincipalId / subject | The token's `sub`; owner of sessions, jobs, assets, and the user layer. | `01` §1 |
| precedence | Last-write-wins in an accumulator: a later layer's unlocked value replaces an earlier one. | `07` |
| provenance | Which layers and sources a piece of context came from (on memories, skills, tool results, candidates); used for cross-team routing checks and prompt framing. | `08` §6d, `09` |
| prompt fragment | One addressable piece of the system prompt with id, section, priority, locked/optional/shrinkable flags. | `01` §4 |
| recall | Model-free memory search across layers, rank-fused and budgeted into the prompt. | `08` §10 |
| resource | Two senses, say which: a **layered resource kind** (skills, memories, prompt fragments, rules, tools; what a step contributes) and a **skill resource file** (a file under a skill's `resources/`, read confined to the skill folder). | `07`, `08` §15g |
| rule | A `ClassifierRule` from a layer's rule source with a role (`tool_audit`, `memory_mining`, `memory_recall`), a kind, and fixed match fields. Client gates are rules too. | `08` §8a |
| restrictive / widening rule | Restrictive kinds (`deny`, `require_approval`, `drop`, `threshold`, `redact`, `disable`) accumulate across layers; widening kinds (`allow`, `retag`, `auto_accept`) act only in their own layer and yield unless locked. | `08` §8b, §6c |
| risk escalation | Service-level bump of a tool's risk by its origin or layer, applied before any rule. | `08` §8c |
| scope | An OAuth2 scope string on the token (e.g. `use-team-skill`); "permission" is a synonym in these notes. Not to be confused with a capability (tool grant) or `remember_match` (which args an approval covers). | `04` |
| session | One client's private conversation: token, conversation, event log, approvals, client tools and gates; never shared. | `08` §11 |
| shielded | Per-call audit state set by a locked `allow`: later layers' restrictions are skipped for that call. | `08` §8b |
| skill | A mini-plugin: instructions plus optional tools and prompt fragments, from a skill store, loaded by a tool call. | `01` §3 |
| skill summary | The cheap discovery record for a skill (name, description, tool names, flags) shown in the skill index. | `08` §15b |
| source / store | **Source**: the port a step reads (skills, memories, prompts, rules, tools). **Store**: a source that also has a writer. | `01` |
| stale | A fork whose recorded base version no longer matches the base's current version. | `08` §7d |
| step | The unit of work in an inner chain: one source's contribution, one gate, or one transform. Has `kind: gating \| contributing \| consuming`. | `07` |
| team layer | The layer for one group in the token; one outer-chain entry per group. | `02` |
| tool origin | Where a tool definition came from: a tool source, a skill (with its layer), or the client. Rules may match on it. | `01` §3 |
| turn context | The per-turn object all steps read and append to; carries the token, session, the user message, accumulators, and post-model state. | `07` |
| user layer | The subject's own layer, paths templated on `${user_root}`; scoped by `*-personal-*` permissions. | `07`, `04` |
| verdict | The decision for one tool call (`ALLOW` / `DENY` / `ASK_USER`) with its source (rule id or model) and confidence. | `08` §8 |
| x-arg-kind | JSON-schema annotation (`path \| host \| url`) on a tool's input properties; the only way an argument is recognised as a path or host by capability checks and client gates. No heuristics. | `08` §12e, §14c |
