# Ideas and open questions

Raw list. Promote items into `design/` when they mature; record outcomes in
`decisions/`.

## Open questions

- **Model config as a layered resource?** Model choice, effort, and
  max output are service config today (`08-walkthrough.md` §16f). A team
  wanting a different agent model, or a user wanting lower effort, would
  make model config a sixth layered resource. Defer until asked for.
- **Language/runtime.** Undecided. Python has the richest AI ecosystem;
  TypeScript makes web + CLI + desktop (Electron/Tauri) sharing easier.
  Could also be language-neutral via a wire protocol between core and
  front ends.
- ~~Is the core a library or a daemon?~~ Decided: it runs as a service
  behind a streaming web API (`decisions/0001-streaming-web-api.md`).
  Remaining sub-question: should a pure in-process mode also be supported
  for tests and embedding, or is "run the service on localhost" enough?
- **SSE vs. WebSocket.** SSE is simpler and enough for server-to-client
  streaming with plain POSTs upstream. WebSocket allows true bidirectional
  streaming (e.g. streaming tool results back). Could offer both over one
  event schema.
- ~~API auth.~~ Decided: OAuth2-style login, permission scopes on the
  token (`decisions/0004-oauth2-scope-permissions.md`). Local mode uses a
  dev IdP rather than "no auth" so the code path is identical.
- ~~Rename `Scope` to avoid clashing with OAuth2 scopes.~~ Superseded:
  the resolved context object was removed entirely; the validated
  `AccessToken` (sub, scopes, groups) is passed through instead.
- **`token.allows(layer, verb, resource)` helper.** The scope-to-layer
  table is small; a single helper on the token object keeps every
  handler from re-implementing it. Where should it live so the token
  type stays a plain claims holder?
- ~~Permission naming pattern.~~ Done: `<verb>-<layer>-<resource>`;
  `allowed-personal-memory` split into `use-personal-memory` and
  `create-personal-memory`.
- **Implicit grants.** Should `create-global-skill` imply
  `use-global-skill`? Current proposal is no implicit grants.
- **On-behalf-of tokens for background work.** Current answer
  (`08-walkthrough.md` §6a): mine with the session token immediately after
  the turn; if it is near expiry, queue the turn and mine it with the
  next fresh token. End-of-session mines what is left if the token is
  still valid (`08-walkthrough.md` §11e); otherwise the turns are logged
  as dropped.
- ~~Session persistence across API restarts.~~ Resolved in
  `08-walkthrough.md` §11f: a pluggable `SessionStore`; running turns at
  crash time become `turn_interrupted`; the chain is rebuilt on load.
- **Raw token at rest.** Mining after a restart needs the user's token,
  so the session store keeps it encrypted. Alternative: drop post-restart
  mining and never persist raw tokens. Which does the deployment prefer?
- ~~Embedding ownership.~~ Resolved in `08-walkthrough.md` §10g: stores
  own embedding; a shared embedder is optional config; cross-store
  ranking uses RRF so scores never need to be comparable. Cross-encoder
  rerank remains an optional extra merge pass.
- **Query rewriting for recall.** Recall is model-free by design (no
  model call in `on_message`). Would a cheap rewrite ("what is this
  message really asking about?") improve hit rate enough to justify the
  latency, perhaps only when the message is short?
- **Memory schema stability.** How much structure to standardize (kinds,
  tags, links) vs. leave to adapters. Too little and "organized" is hollow;
  too much and adapters fight the schema.
- **Skill trust.** A skill from a global repo can instruct the agent to do
  things and now can also ship executable tools. Skills declare
  capabilities and the runtime enforces them at the tool boundary
  (`01-core-abstractions.md`). Skill loading is itself a tool call
  (`load_skill`, `08-walkthrough.md` §15d), so it is audited and
  rule-gated by construction. Decided (`decisions/0010`): the global
  allow for `load_skill` is *unlocked*, so the model audit is skipped by
  default but any team or user restriction still fires.
- ~~Sandboxing skill tools.~~ v1 decided in `08-walkthrough.md` §12d: a
  restricted subprocess (user/mount/network namespaces, read-only binds,
  egress proxy with host allowlist, rlimits, seccomp), mandatory for
  script impls below the user layer; container and wasm as adapters.
  Open: is the egress proxy acceptable operationally, or should team
  script tools be `http`-only in v1?
- ~~Tool implementation portability.~~ Resolved: a config allowlist of
  runtimes (`tools.runtimes`); a script names one; anything else must be
  `http` (§12c).
- **Secrets via egress proxy.** Attaching secrets at the proxy keeps them
  out of the sandbox entirely, but couples the secret to a host pattern.
  Is per-host attachment enough, or do some tools need the raw value?
- ~~Tool schema versioning.~~ Resolved in §15h: a loaded skill is pinned
  to its version for the session; refresh emits `skill_outdated`; reload
  re-puts tools at the new version.
- **Classifier latency budget.** Tool-call audit sits on the critical path.
  What is acceptable? Can it run speculatively in parallel with the primary
  model's generation?
- ~~Multi-subscriber sessions.~~ Decided: one client, one session
  (`decisions/0002-one-session-per-client.md`). Session-summary memory at
  end of session is now a config option (`08-walkthrough.md` §11e),
  routed through the normal memory chain.
- **Memory confirmation UX.** Personal writes: silent with a
  `memory_created` event. Team writes without an active team or
  auto-accept rule: `memory_confirm` event. Still open: should a client
  be able to opt out of confirmations entirely (always downgrade)?
- **Fork as overlay instead of copy.** A full copy goes stale as the base
  evolves. An overlay (store only the user's changes, re-apply on load)
  tracks upstream automatically but makes "what does this skill actually
  say" harder to answer and conflicts likely. Copy-with-diff first; revisit
  if staleness becomes a real complaint.
- **Auto-rebase forks.** When the base changes and the diff applies
  cleanly, offer (or silently do) a rebase of the personal fork.
- **Same-name skills across two of a user's teams.** Shadowing by layer
  order works between global/team/user, but two team sub-layers are peers.
  Options: IdP claim order wins; always require the full id; surface both
  to the model with team prefix.
- **Per-team permission granularity.** Starting set grants `create-team-*`
  for all of a user's teams at once. If "publish to team A, read-only in
  team B" is needed, options are per-team permission strings
  (`create-team-skill:teamA`) or a role claim per team from the IdP.
- ~~Team memory write routing.~~ Decided: miner suggests an audience,
  router decides with personal as fallback; `active_team` on the session
  is the main positive signal (`decisions/0007-memory-routing.md`).
- **Promotion review for teams.** Should a team be able to require that
  promoted or auto-accepted memories go through a team reviewer before
  other members recall them? A `pending` flag on team memories would do
  it.
- **Prompt permissions.** Should prompt layers be gated by dedicated
  `use-<layer>-prompt` permissions, or should a layer's fragments ride
  along with any `use-<layer>-*` permission the token already holds?
  Dedicated is cleaner; ride-along means fewer permissions to manage.
- **Locked fragment semantics with several teams.** If two of a user's
  teams both lock the same fragment id, which wins? Same peer-conflict
  problem as skills.
- ~~Name the shape.~~ Resolved by `decisions/0008-layer-handler-chain.md`:
  the layer is the handler; resources are accumulators on the turn
  context. No generic `LayeredResource<T>` needed.
- ~~Handler granularity.~~ Resolved by `decisions/0009-chain-of-chains.md`:
  a layer is an inner chain of small steps per hook; one step per source.
- **Step failure modes.** Default is fail closed for gating steps and fail
  open for contributing steps. Should a layer be able to declare a
  contributing step `required` (e.g. org policy prompt) so its outage
  fails the turn?
- ~~Client-supplied session steps.~~ Resolved in `08-walkthrough.md`
  §14c: clients register declarative *gates* (rules with fixed match
  fields, restrictive only, unlocked), never steps or code.
- **Speculative audit cancellation.** If the model-based audit started on
  partial arguments and the final arguments differ materially, restart or
  accept? Proposal: hash the arguments; restart on change.
- **Inner-chain onion.** The inner chains are flat. If a step ever needs
  "before and after the rest of my layer" semantics, an onion-style inner
  chain could be allowed per layer without changing the outer chain.
- **Tool name collisions with skill namespacing.** Skill tools are
  `<skill>.<tool>`; layer tools are bare names. Should layer tools be
  namespaced too (`team-x.search`) so shadowing is always explicit?
- ~~Client-offered tools and trust.~~ Resolved in §14b: client tools get
  client-only capabilities, a risk floor plus session escalation, forced
  `execute_on: client`, and untrusted results. Open: should an
  authenticated *trusted app* client (its own OAuth client id) be allowed
  a lower escalation than an anonymous CLI?
- **Rule expressiveness.** How rich should `ClassifierRule.match` be?
  Options: fixed fields (tool name glob, risk, skill layer); a small
  expression language; or "rules are just prompt fragments and the model
  decides". Static fields first; expression language only if needed.
- **Speculative audit.** Static rules are instant; the model handler is
  not. Can the model-based audit run in parallel with the main model's
  generation of the tool call arguments, and be cancelled if a static rule
  decides first?
- **Conflict between memories.** Two stores disagree (old preference vs.
  new). Timestamp wins? Layer wins? Surface both to the model?

## Ideas parking lot

- Memory writes as a chain of responsibility: should a layer be able to
  *claim* a candidate (e.g. team store says "this mentions our project,
  I'll take it") or is routing purely by an explicit tag from the miner?
- Should a store be able to declare it participates in the chain for
  some skill names only (prefix-based routing)?

- Memory "decay" or confidence that lowers with age unless reinforced.
- ~~Skills that bundle their own prompt fragments and tools (a skill as a
  mini-plugin).~~ Promoted to design; see `01-core-abstractions.md` §3.
- An audit log of every classifier verdict for later tuning.
- A `dry-run` mode where the classifier only logs what it would have
  blocked, for calibration before enforcement.
- Provenance on every context item so the model can cite "from your user
  memory" vs. "from team policy".
- A reference implementation with only file-based adapters, to prove the
  interfaces before any database or service adapters exist.
