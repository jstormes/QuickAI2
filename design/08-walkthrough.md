# Walkthrough: startup, then a chat turn with a tool call

Language-neutral pseudocode using the names from `06-object-model.md`
and `07-turn-pipeline.md`. Error handling and most events are omitted
unless they matter to the flow.

## 1. Service startup

```
main(config_path):
  config = load_yaml(config_path)

  # --- registries: the extension points. Adapters and steps register
  #     themselves; the core imports none of them.
  sources = SourceFactory()
  sources.register("git",        GitSkillStore, GitPromptSource, GitRuleSource)
  sources.register("directory",  DirectorySkillStore, DirectoryToolSource)
  sources.register("file",       FilePromptSource, FileRuleSource)
  sources.register("http",       HttpMemoryStore, HttpSkillStore, HttpRuleSource, HttpToolSource)
  sources.register("sqlite_vec", SqliteVecMemoryStore)
  sources.register("builtin",    BuiltinToolSource)
  for plugin in config.plugins: plugin.register(sources)     # optional extra adapters

  steps = StepFactory(sources)
  steps.register("gate",              GateStep)
  steps.register("prompts",           PromptFragmentsStep)
  steps.register("skills",            SkillDiscoveryStep)
  steps.register("tools",             ToolDefinitionsStep)
  steps.register("memories",          MemorySearchStep)
  steps.register("rules",             RuleLoadStep)
  steps.register("static_rules",      StaticRulesStep)
  steps.register("route_to_team",     RouteToTeamStep)
  steps.register("route_to_personal", RouteToPersonalStep)
  steps.register("audit_log",         AuditLogStep)

  # --- shared, service-level singletons
  model      = ModelClientFactory.create(config.model)              # e.g. anthropic adapter
  classifier = ClassifierEngineFactory.create(config.classifier, model)
  runners    = ToolRunnerFactory(config.tools.runners)             # script/http/builtin/client
  assembler  = PromptAssembler(RenderStrategy.for(config.model))
  merger     = MemoryMergeStep(MergeStrategy.by_name(config.memory.merge or "rrf"))
  recall     = RecallPolicy(config.memory.recall)
  miner      = classifier.miner()                                   # same engine, mining role
  events     = EventBusFactory()

  # --- layers. Non-templated layers (global, user) are built once and
  #     cached. Templated layers (team:{team_id}) are built lazily per team
  #     the first time a token carrying that group arrives.
  layers = LayerFactory(config.layers, steps)
  layers.prebuild(["global"])            # fail fast if org policy sources are down
  # "user" is templated on token.subject, so it is also lazy + cached.

  # --- API
  validator = TokenValidatorFactory.create(config.api.auth)          # oauth2_jwt or dev_idp
  sessions  = SessionStoreFactory.create(config.api.sessions)        # in-memory | sqlite
  core      = AgentCore(layers, model, classifier, runners, assembler, merger, recall, miner)
  api       = ApiLayer(config.api, validator, sessions, core, events)
  api.listen()                                                       # HTTP + SSE
```

`LayerFactory.for_entry`, called by `prebuild` and lazily later:

```
LayerFactory.for_entry(entry, token=None) -> Layer:
  vars  = { team_id: token?.current_team, subject: token?.subject,
            user_root: f"{config.users_root}/{token?.subject}" }      # every user-layer path MUST be under user_root (G37)
  layer = Layer(name=entry.name, hooks={})
  for hook_name, step_configs in entry.hooks:
      chain = InnerChain()
      for sc in step_configs:
          source = sources.create(kind_for(sc.step), sc.adapter, expand(sc, vars)) if sc.adapter
          step   = steps.create(sc.step, source, sc)
          step   = TimedStep(step)
          step   = FailClosedStep(step) if sc.gating else FailOpenStep(step)
          if sc.required: step = RequiredStep(step)
          chain.append(step)
      layer.hooks[hook_name] = chain
  return layer
```

## 2. Client logs in and opens a session

```
# CLI: device flow against the IdP, gets an access token
POST /sessions  Authorization: Bearer <jwt>  { "active_team": "team-a" }

ApiLayer.create_session(request):
  token = validator.validate(request.bearer)          # sub, scopes, groups, exp
  if request.body.active_team not in token.groups: return 403
  session = Session(id=new_id(), owner=token.subject, token=token,
                    active_team=request.body.active_team,
                    conversation=Conversation(), bus=events.new())
  sessions.save(session)
  return 201 { session_id }

# CLI then opens the stream
GET /sessions/{id}/events   -> text/event-stream (one stream per session)
```

Building the outer chain for this token (done once per session, cached
until the token changes):

```
LayerFactory.chain_for(token, session) -> OuterChain:
  chain = []
  for entry in config.layers:                            # config order = trust order
      if entry.name == "global":
          chain.append(cached(entry) or for_entry(entry))          # ALWAYS present: policy baseline (04-auth-and-permissions.md)
          continue                                                 # scope gating happens per step inside the layer (below)
      if not token.allows_any(entry.name):               # no use-* scope for this layer at all
          continue
      if entry.name == "team":
          for team_id in token.groups:                   # one outer entry per team
              chain.append(cached(entry, team_id) or for_entry(entry, token.with_team(team_id)))
      else:
          chain.append(cached(entry, token) or for_entry(entry, token))
  chain.append(SessionLayer(session))                    # built in, always last
  return OuterChain(chain)

# Inside any layer, a step is skipped when the token lacks the scope for that
# step's resource (07-turn-pipeline.md). Exceptions that never need a scope:
#   global prompts, global rules, global gates              -> the policy baseline
#   the session layer's own steps                           -> the client's own contributions
# So a personal-only token still gets global locked rules and org policy
# fragments, but no global skills, tools, or memories.

Suppose the token has scopes
`use-global-skill use-team-skill use-personal-skill use-personal-memory
create-personal-memory use-team-memory create-team-memory
run-server-tools run-high-risk-tools` and groups `[team-a]`. The outer
chain is:

```
[ global, team:team-a, user, session ]
```

## 3. One chat turn with a tool call

The user types: *"Check whether the nightly build passed and summarise
the failures."* Suppose the global layer provides a `ci_status` tool and
`team-a` has a skill `nightly-triage` that adds a prompt fragment and a
tool `nightly-triage.fetch_log`.

```
POST /sessions/{id}/messages { "text": "Check whether the nightly build passed..." }
-> 202 { turn_id }
```

### 3a. `on_message`

```
AgentCore.run_turn(session, message):
  ctx   = TurnContext(token=session.token, session=session, inbound=message)
  chain = session.outer_chain                            # from chain_for(token, session)
  session.conversation.append(user_message(message))     # append-only history: the user turn goes in first (§16b)
  emit(turn_started)

  # phase 1: gates, sequential, outer order
  for layer in chain:
      for step in layer.hooks.on_message.gating():
          if (r := step.run(ctx)) is Stop: emit(r.action); return
  # e.g. global GateStep(rate_limit): 12/60 this minute -> Continue

  # phase 2: fetch concurrently, apply in order
  fetches = []
  for layer in chain:
      for step in layer.hooks.on_message.contributing():
          fetches.append((layer, step, async step.fetch(ctx)))   # store lookups only
  results = await_all(fetches)                                # slowest single source
  for (layer, step, result) in results in chain order:        # outer, then inner
      step.apply(ctx, result)                                  # ctx.put(...) honours locks
```

What `apply` does for each layer (illustrative contents):

```
global:
  prompts   -> ctx.prompt["org-policy"]        (locked)
               ctx.prompt["assistant-identity"]
  skills    -> ctx.skills["code-review"], ctx.skills["release-notes"]
  tools     -> ctx.tools["ci_status"], ctx.tools["shell"] (locked)
  memories  -> (global store read-only, returns 0 hits for this query)
  rules     -> ctx.rules += [deny shell from layers below user (locked), require_approval risk>=high]

team:team-a:
  skills    -> ctx.skills["nightly-triage"]    (summary + tool names; body loaded lazily)
  memories  -> ctx.memories += [ "nightly job name is build-nightly-linux" (score .81),
                                 "flaky test: test_upload_retry" (score .66) ]
  rules     -> ctx.rules += [allow nightly-triage.* without audit]

user:
  prompts   -> ctx.prompt["user-prefs"]        ("terse answers, UK spelling")
  skills    -> ctx.skills["code-review"]       (user's fork: overwrites team/global copy, not locked)
  memories  -> ctx.memories += [ "user prefers failures grouped by test file" (.74) ]
  rules     -> (none)

session:
  tools     -> ctx.tools["open_in_editor"]     (client-offered, execute_on=client)
  active_team = team-a                          (already on session)
```

Merge steps and the model call:

```
  ctx.memories  = recall.select(merger.merge(ctx.memories), budget=1500 tokens)
  system_prompt = assembler.assemble(ctx.prompt, budget=config.prompt_budget)
  tool_defs     = [t.definition for t in ctx.tools.values()] + skill_summaries(ctx.skills)
  emit(context_ready, timings=per_layer(results), counts=...)
```

### 3b. Model call, tool call emitted

```
  loop:
      stream = model.complete(system=system_prompt,
                              messages=session.conversation.messages,     # already includes this turn's user message
                              tools=tool_defs)
      for chunk in stream:
          match chunk:
            text_delta(t):      emit(assistant_delta, t)          # unseen by any step
            tool_call_start(c): emit(tool_call_proposed, c, state="reviewing")
                                audits[c.id] = async run_tool_call(ctx, chain, c)   # speculative
            tool_call_args(c):  c.args += ...; if audits[c.id].args_hash != hash(c.args): restart audit
            tool_call_end(c):   pending.append(c)
      if not pending: break                                        # final answer streamed
      results = await_all(audits[c.id] for c in pending)           # all calls gated concurrently
      session.conversation.append(assistant_turn_with(pending), tool_results(results))
      pending.clear()
```

Here the model first calls `ci_status(job="build-nightly-linux")`, using
the team memory to pick the job name.

### 3c. `on_tool_call` for `ci_status`

```
run_tool_call(ctx, chain, call) -> ToolResult:
  definition = ctx.tools.resolve(call.name)                    # from the accumulator, already precedence-resolved
  if not definition: return error("unknown tool")

  for layer in chain:                                          # outer order
      if (r := layer.run("on_tool_call", ctx, call)) is Stop:
          return handle_stop(r, call)
  # global StaticRulesStep: evaluates the rules the global layer loaded (§8b: this layer's rules only):
  #   deny shell from lower layers?      no match (ci_status)
  #   require_approval risk>=high?       ci_status.risk == low -> no match
  # team-a StaticRulesStep: allow nightly-triage.*?  no match
  # user: no rules
  # -> Continue from every layer, so the service-appended step runs:

  if (r := ModelAuditStep(classifier).run(ctx, call)) is Stop:
      return handle_stop(r, call)
  # classifier.audit(conversation_window, call, instructions=prompt_fragment_rules)
  # -> ALLOW (confidence .93, "user asked for build status")

  ctx.verdicts[call.id] = ALLOW
  emit(tool_call_started, call.id)
  runner = runners.for(definition.impl)                        # http impl -> HttpRunner
  runner = CapabilityCheckedRunner(TimeoutRunner(AuditedRunner(runner)))
  result = runner.run(definition, call.args)                   # GET https://ci.example/api/jobs/...
  emit(tool_call_finished, call.id, summary=result.summary)
  return result
```

Second iteration of the loop: the model sees the CI result (failed, 3
tests), decides it needs the log, and loads the team skill.

```
  # model returns tool_call load_skill("nightly-triage")   (load_skill is a builtin global tool)
  # on_tool_call -> ALLOW (static rule from team-a: allow nightly-triage.*)
  # executing it: SkillStore(team-a).load("nightly-triage") -> Skill
  #   ctx.tools.put("nightly-triage.fetch_log", locked=False)   at layer team-a
  #   ctx.prompt.put("nightly-triage", fragment)                 at layer team-a
  #   tool_defs and system_prompt are re-derived for the next model call
  emit(skill_loaded, "nightly-triage", tools=["nightly-triage.fetch_log"])

  # model then calls nightly-triage.fetch_log(job=..., run=1234)
  # on_tool_call: team-a static rule "allow nightly-triage.*" matches -> Stop? No:
  #   an `allow` rule returns Continue with ctx.verdicts[call.id] = ALLOW(rule) and
  #   marks the call as decided, so ModelAuditStep is skipped.
  # runner: script impl from a team layer -> SandboxedRunner is mandatory
  #   (config: sandbox required below layer user)
  # result: log excerpt with the 3 failing tests
```

Third iteration: no tool calls, the model streams the summary grouped by
test file (user preference memory), in UK spelling (user prompt
fragment), terse (same). `assistant_message` is emitted with citations
to the two memories and the skill.

### 3d. `on_turn_end`, then background mining

```
  session.conversation.append(final_message)
  for layer in chain: layer.run("on_turn_end", ctx)            # global AuditLogStep posts the verdict log
  emit(turn_complete, usage=...)

  schedule_background:
      candidates = miner.observe(ctx)
      # e.g. { kind: project, body: "nightly run 1234 failed on test_upload_retry (flaky) and 2 others",
      #        audience: team:team-a, confidence: .7 }
      #      { kind: feedback, body: "user wants failures grouped by test file", audience: personal, .9 }
      candidates = prefilter(ctx, candidates)            # restrictive mining rules from ALL layers, before routing (§6c)
      for c in candidates:
          for layer in chain:
              if (r := layer.run("on_memory_candidate", ctx, c)) is Stop: break
          # global: no accepting step (its restrictive rules already ran in prefilter)
          # team-a RouteToTeamStep: audience == team-a, token has create-team-memory,
          #        team-a in groups, session.active_team == team-a  -> ACCEPT, write, Stop
          #        (second candidate: audience personal -> decline, Continue)
          # user RouteToPersonalStep: second candidate -> dedupe against existing
          #        "prefers failures grouped by test file" (.74 hit earlier) -> update, not insert
          emit(memory_created, ...)       # arrives after turn_complete on the stream
```

## 4. Event timeline the CLI renders

```
turn_started
context_ready            global 38ms · team-a 210ms (git fetch) · user 6ms · session 0ms
assistant_delta ...      "Checking the nightly build…"
tool_call_proposed       ci_status  state=reviewing
tool_call_started        ci_status                        (after model audit, ~400ms)
tool_call_finished       ci_status  "failed: 3 tests"
tool_call_proposed       load_skill nightly-triage  state=reviewing
tool_call_started        load_skill                       (static allow, ~0ms)
tool_call_finished       load_skill
skill_loaded             nightly-triage  tools=[nightly-triage.fetch_log]
tool_call_proposed       nightly-triage.fetch_log  state=reviewing
tool_call_started        nightly-triage.fetch_log         (static allow)
tool_call_finished       nightly-triage.fetch_log
assistant_delta ...      the summary
assistant_message        + citations
turn_complete            usage
memory_created           team:team-a  "nightly run 1234 failed on …"
memory_created           personal     (updated) "prefers failures grouped by test file"
```

## 5. The approval flow

Approval is a `Stop(RequireApproval)` outcome from an `on_tool_call`
step, answered by the session's one client. The same mechanism serves
team-memory confirmation (`memory_confirm`), with a different payload.

### 5a. Who can ask

```
# (i) a static rule in any layer's StaticRulesStep
rule { kind: require_approval, match: risk >= high, locked: true }    # global
rule { kind: require_approval, match: tool == "shell", locked: false } # user's own caution

# (ii) the service-appended ModelAuditStep, when the classifier is unsure
Verdict { decision: ASK_USER, confidence: .55, reason: "user asked to *check* the file, tool would *delete* it" }
```

Both produce the same outcome from the step. The full step logic,
including how locked allows shield a call and how restrictions from
several layers accumulate, is in §8b; the shape is:

```
StaticRulesStep(layer).run(ctx, call):        # this layer's rules only
  ... deny            -> Stop(Deny)                       unless shielded by an earlier locked allow
  ... require_approval-> Stop(RequireApproval(...))       unless shielded or already decided
  ... allow           -> record ALLOW, mark decided (and shielded if locked); Continue
  return Continue

ModelAuditStep.run(ctx, call):                # appended by the service after the last layer
  if decided: return Continue
  verdict = classifier.audit(window, call, instructions=layered prompt fragments)
  ALLOW -> Continue | DENY -> Stop(Deny) | ASK_USER -> Stop(RequireApproval(...))
```

```
ApprovalRequest {
  approval_id, session_id, turn_id
  tool_call: { id, name, args, risk, layer, execute_on }
  asked_by: { kind: rule | classifier, id, layer, locked: bool }
  reason: text
  options: [allow, deny]
  remember_options: [once, session]          # "always" only if asked_by is unlocked; see 5e
  timeout_s: 300                             # from rule or config
  created_at
}
```

### 5b. Parking the call

`run_tool_call` (section 3c) handles the stop:

```
handle_stop(r, call, ctx, chain, resume_from):
  match r.action:
    Deny(reason):
        ctx.verdicts[call.id] = DENY(reason)
        emit(tool_call_denied, call.id, reason)
        return ToolResult.error(f"tool call denied: {reason}")     # model sees this and can explain

    RequireApproval(req):
        ctx.pending_approvals[req.approval_id] = Pending(req, call, resume_from, future=Future())
        session.approvals.save(req)                                 # survives reconnect
        emit(approval_requested, req)                               # replayable via Last-Event-ID
        decision = await ctx.pending_approvals[req.approval_id].future
                         .with_timeout(req.timeout_s, default=Deny("approval timed out"))
        return resume_after_approval(decision, call, ctx, chain, resume_from)
```

Nothing else waits. The model stream keeps producing; other tool calls
from the same response are audited concurrently; the client keeps
receiving deltas. Only this one call is parked, and the pending loop in
3b awaits all results before the next model call.

### 5c. The client round trip

```
# stream
event: approval_requested
data: { approval_id: "ap_91", tool_call: { name: "shell", args: { cmd: "rm -rf build/" }, risk: "high" },
        asked_by: { kind: "rule", id: "global.high-risk", layer: "global", locked: true },
        reason: "risk >= high requires approval", options: ["allow","deny"],
        remember_options: ["once"], timeout_s: 300 }

# CLI renders:
#   ⚠ shell: rm -rf build/    (high risk, org policy requires approval)
#   [a]llow  [d]eny  (times out in 5:00)

# client answers
POST /sessions/{id}/approvals/ap_91  { "decision": "allow", "remember": "once", "note": "" }

ApiLayer.answer_approval(request):
  token   = validator.validate(request.bearer)
  session = sessions.load(request.session_id)
  if session.owner != token.subject: return 403
  req = session.approvals.get(request.approval_id)
  if not req or req.answered: return 409
  if request.remember not in req.remember_options: return 400
  req.answered = Decision(request.decision, request.remember, request.note, by=token.subject, at=now())
  session.approvals.save(req)
  core.resolve_approval(session, req.approval_id, req.answered)    # completes the future
  return 200
```

The client may also list what is outstanding after a reconnect:

```
GET /sessions/{id}/approvals?state=pending -> [ApprovalRequest]
```

### 5d. Resuming the chain

```
resume_after_approval(decision, call, ctx, chain, resume_from):
  emit(approval_resolved, call.id, decision.decision, by=decision.by)

  if decision.decision == deny:
      ctx.verdicts[call.id] = DENY("user declined" + decision.note)
      return ToolResult.error("user declined this tool call" + note)

  # allowed: the user is the authority for *this* call
  ctx.verdicts[call.id] = ALLOW(approval=decision)
  ctx.decided.add(call.id)                        # ModelAuditStep will skip it

  # continue the outer chain from the step AFTER the one that asked.
  # Later layers still get their turn: a session-layer gate (e.g. a CLI
  # refusing paths outside the cwd) can still deny. A later layer cannot
  # re-ask for the same call: a second RequireApproval for a decided call
  # is treated as Continue.
  for (layer, step) in chain.steps_after("on_tool_call", resume_from):
      r = step.run(ctx, call)
      if r is Stop and r.action is RequireApproval: continue
      if r is Stop: return handle_stop(r, call, ctx, chain, (layer, step))

  return execute(call, ctx)                        # 3c: runner factory + decorators
```

### 5e. Remembering a decision

```
apply_remember(decision, req, ctx):
  match decision.remember:
    once:    nothing
    session: # add an unlocked allow rule to the session layer for the rest of this session
             ctx.session.rules.append(rule { kind: allow, match: tool == req.tool_call.name
                                             and args_hash == hash(req.tool_call.args)?  # exact or by name, client's choice
                                             layer: session, locked: false })
```

Constraint: a session-level allow rule is unlocked and lives in the last
layer, so it can only pre-empt an *unlocked* asker. If the ask came from
a locked global rule (the `rm -rf` case above), `remember_options` does
not include `session`, and the user is asked every time. This is why the
options are computed by the asker, not the client.

Nothing is ever remembered across sessions by this path. A user who wants
a permanent allow writes a rule into their user-layer rule source
(`create-personal-classifier-rule`), which is a deliberate act outside
the chat.

### 5f. Cancel, timeout, disconnect

```
on cancel(turn_id):                  # POST /sessions/{id}/cancel
  for p in ctx.pending_approvals.values(): p.future.set(Deny("turn cancelled"))
  model stream aborted; parked calls return error results; turn ends with turn_complete(cancelled=true)

on timeout:                          # per request, from rule or config
  future defaults to Deny("approval timed out"); model is told; turn continues

on client disconnect:
  nothing changes server-side; the request stays pending until timeout.
  On reconnect the client resumes the stream with Last-Event-ID (replays
  approval_requested) or calls GET /approvals?state=pending.
```

### 5g. Memory confirmation uses the same path

`RouteToTeamStep` may return `Stop(RequireApproval(req))` with
`req.kind = memory` when a candidate is tagged for a team but there is no
active-team match and no auto-accept rule. The client sees
`memory_confirm`, answers with `memory_decision` (accept team / keep
personal / discard), and the candidate resumes through the remaining
`on_memory_candidate` steps. Because this runs off the critical path,
the timeout default is longer and the fallback on timeout is "keep
personal", not "discard".

### 5h. Event timeline for an approved call

```
assistant_delta ...
tool_call_proposed       shell  state=reviewing
approval_requested       ap_91  shell rm -rf build/   asked_by global.high-risk (locked)
   ... user thinks; deltas for any parallel text keep flowing ...
approval_resolved        ap_91  allow  by=<subject>
tool_call_started        shell
tool_call_finished       shell
assistant_delta ...
turn_complete
```

## 6. Memory mining and confirmation

Mining runs after `turn_complete`, off the critical path, as a background
job keyed by `turn_id`. It has three stages: the miner proposes
candidates, restrictive mining rules from every layer filter them, and
each surviving candidate walks the `on_memory_candidate` hook through the
outer chain, where layers retag, accept, decline, or ask.

### 6a. Scheduling and identity

```
AgentCore.after_turn(ctx):
  if not config.classifier.memory_mining.enabled: return
  if ctx.token.expires_at < now() + config.mining.min_token_ttl:
      session.unmined_turns.append(ctx.turn_id)          # retry next turn with a fresh token
      return
  jobs.submit(MiningJob(session_id, turn_id, token=ctx.token, snapshot=ctx.snapshot()))
  # snapshot: inbound message, assistant message, tool calls + result summaries,
  #           recalled memories (ids + layers), loaded skills (ids + layers),
  #           active_team, rules (mining role), timings

MiningJob.run():
  if store.mined(turn_id): return                          # at-most-once per turn
  ctx = TurnContext.from_snapshot(snapshot)                # no model call, no stores yet
  turns = [snapshot] + session.take_unmined_turns()        # catch up if earlier turns were skipped
  candidates = mine(ctx, turns)
  candidates = prefilter(ctx, candidates)
  for c in candidates: route(ctx, c)
  store.mark_mined(turn_id)
```

Mining acts **as the user**: it writes with the session's token, so
scopes and groups apply exactly as they would for a foreground write.

### 6b. The miner proposes

```
mine(ctx, turns) -> [MemoryCandidate]:
  instructions = assembler.assemble(ctx.rules.prompt_fragments(role="memory_mining"))
  # layered: global says what kinds exist and what never to keep;
  #          team says what is team-relevant; user says their preferences
  raw = classifier.mine(turns,
                        existing=ctx.memories,               # what was recalled: avoid restating it
                        groups=ctx.token.groups,
                        active_team=ctx.session.active_team,
                        team_context=[s.layer for s in ctx.skills_loaded] +
                                     [m.layer for m in ctx.memories],
                        instructions=instructions)
  return [MemoryCandidate(kind=r.kind, body=r.body, rationale=r.why,
                          confidence=r.confidence, audience=r.audience,   # personal | team:<id>
                          tags=r.tags, links=r.related_memory_ids,
                          source_turn=turn_id, hash=hash(r.kind, normalise(r.body)))
          for r in raw]
```

The miner is asked for a **suggested audience**, never a store. It picks
`team:T` only when the turn carried team signal: the active team, a team
skill was loaded, a team memory was recalled, or the conversation named
the team or its project. Otherwise it says `personal`. When unsure it
says `personal`.

Example output for the turn in section 3:

```
c1 { kind: project,  body: "Nightly run 1234 (build-nightly-linux) failed: test_upload_retry (flaky) + 2 in upload/",
     audience: team:team-a, confidence: .72, links: [mem_team_a_17], tags: [ci, nightly] }
c2 { kind: feedback, body: "Prefers CI failures grouped by test file",
     audience: personal, confidence: .91, links: [mem_user_04] }
c3 { kind: reference, body: "CI API token is ci_live_8f3…",                       # from a tool result
     audience: personal, confidence: .60 }
```

### 6c. Restrictive rules apply globally, before routing

Rules that *narrow* mining (drop, threshold, redact) come from any layer
and apply to every candidate regardless of which layer loaded them.
Rules that *widen* it (retag to a team, auto-accept) apply only at their
own layer, in 6d. This is why a user can switch mining off for themselves
even though the team layer runs before theirs.

```
prefilter(ctx, candidates):
  rules = ctx.rules.for_role("memory_mining").restrictive()      # kind in {drop, threshold, redact, disable}
  ordered locked-first, then layer order
  if any(r.kind == disable for r in rules): return []            # any layer may disable; disabling is restrictive
  out = []
  for c in candidates:
      for r in rules:
          if not r.match(c): continue
          match r.kind:
            drop:      c = None; break
            threshold: if c.confidence < r.value: c = None; break
            redact:    c.body = r.redact(c.body)
      if c: out.append(c)
  return out

# global (locked): drop kind == credential OR body matches secret_patterns   -> c3 dropped
# global (locked): threshold .5
# team-a:          threshold .65 for audience == team:team-a                  -> c1 (.72) survives
# user:            (none; a user could add `disable` here)
```

### 6d. Routing through the outer chain

Each candidate walks `on_memory_candidate` in outer order. Steps:
`MiningRetagStep` (widening rules), `RouteToTeamStep` (team layers),
`RouteToPersonalStep` (user layer). Global has no accepting step.

```
route(ctx, c):
  for layer in chain:
      r = layer.run("on_memory_candidate", ctx, c)
      if r is Stop and r.action is RequireApproval: r = await_confirmation(ctx, c, r.action.request, resume_from=layer)
      if r is Stop: return                                   # accepted, discarded, or denied
  emit(memory_discarded, c.hash, reason="no layer accepted")  # fell off the end (e.g. no write scopes)
```

Team layer, for `team-a`:

```
MiningRetagStep(team-a).run(ctx, c):
  for r in this_layer.rules.widening():                   # e.g. "mentions project nightly -> audience team-a"
      if r.match(c) and c.audience == personal: c.audience = team:team-a; c.retagged_by = r.id
  return Continue

RouteToTeamStep(team-a).run(ctx, c):
  T = this_layer.team_id
  if c.audience != team:T: return Continue                # not for us
  if not ctx.token.allows("team", "create", "memory") or T not in ctx.token.groups:
      c.declined.append((T, "no scope")); return Continue  # falls through to personal
  signal = (ctx.session.active_team == T) or any(r.auto_accepts(c) for r in this_layer.rules.auto_accept())
  if not signal:
      return Stop(RequireApproval(ApprovalRequest.memory(c, team=T, options=[accept_team, keep_personal, discard],
                                                          timeout_s=config.mining.confirm_timeout_s,
                                                          on_timeout=keep_personal)))
  return write_to(this_layer.store, ctx, c, layer=team:T)
```

User layer:

```
RouteToPersonalStep.run(ctx, c):
  if not ctx.token.allows("personal", "create", "memory"): return Continue
  if c.declined: c.suggested_audience = c.declined[0].team    # keep the hint for promotion
  return write_to(this_layer.store, ctx, c, layer=user)
```

Writing, with dedupe against the target store:

```
write_to(store, ctx, c, layer) -> Stop:
  near = store.search(c.body, top_k=3, filter={kind: c.kind})
  match classify_overlap(c, near):                          # cheap: embedding sim + normalised text; no model call
    duplicate(m):     store.update(m.id, touch=now(), links+=c.links, confidence=max(...))
                      emit(memory_updated, m.id, layer); return Stop(Accepted)
    contradiction(m): id = store.put(Memory.from(c, layer, supersedes=m.id))
                      store.update(m.id, superseded_by=id)
                      emit(memory_created, id, layer, supersedes=m.id); return Stop(Accepted)
    new:              id = store.put(Memory.from(c, layer))
                      emit(memory_created, id, layer, suggested_audience=c.suggested_audience)
                      return Stop(Accepted)
```

For the example: `c1` is tagged `team:team-a`, the session's active team
is `team-a`, so it is accepted into the team store, linked to the
existing "nightly job name" memory. `c2` is personal; the user store
finds `mem_user_04` ("prefers failures grouped by test file", recalled
earlier) as a duplicate and updates it instead of inserting. `c3` never
reached routing.

### 6e. Confirmation

If `c1` had been mined in a session with no active team and no team
auto-accept rule, `RouteToTeamStep` would have asked:

```
await_confirmation(ctx, c, req, resume_from):
  session.approvals.save(req)                                # same store as tool approvals, kind=memory
  emit(memory_confirm, req)                                  # may arrive after turn_complete
  decision = await future(req).with_timeout(req.timeout_s, default=req.on_timeout)   # keep_personal
  emit(memory_decision_resolved, req.approval_id, decision)
  match decision:
    accept_team:   return write_to(team_store(req.team), ctx, c, layer=team:req.team)
    keep_personal: c.declined.append((req.team, "user kept personal"))
                   return Continue                           # resume at the next layer -> RouteToPersonalStep
    discard:       emit(memory_discarded, c.hash, "user"); return Stop(Discarded)
```

```
# stream
event: memory_confirm
data: { approval_id: "mc_12", kind: "memory",
        candidate: { kind: "project", body: "Nightly run 1234 … failed …", confidence: .72 },
        team: "team-a", asked_by: { kind: "step", id: "route_to_team", layer: "team:team-a" },
        options: ["accept_team", "keep_personal", "discard"], timeout_s: 86400, on_timeout: "keep_personal" }

# CLI renders (non-blocking, since the turn is already complete):
#   💾 Save to team-a?  "Nightly run 1234 … failed …"   [t]eam  [p]ersonal  [x] discard

POST /sessions/{id}/approvals/mc_12  { "decision": "keep_personal" }
```

Timeout policy differs from tool approval on purpose: nothing is blocked,
so the window is long and the fallback is the safe one (`keep_personal`),
never `discard`. A client that does not implement memory prompts simply
never answers, and every such candidate lands in personal with a
suggestion attached.

### 6f. Promotion later

```
GET  /memories?suggested_for=team:team-a          -> personal memories with suggested_audience == team-a
POST /memories/{id}/promote?layer=team:team-a

ApiLayer.promote(request):
  token, session = ...
  m = user_store(token.subject).get(request.id)
  if m.layer != user or m.owner != token.subject: return 404
  if not token.allows("team", "create", "memory") or T not in token.groups: return 403
  c = MemoryCandidate.from_memory(m, audience=team:T, confidence=1.0, promoted_from=m.id)
  c = prefilter(ctx_for(token, session), [c])              # locked restrictive rules still apply
  if not c: return 422 "blocked by policy"
  r = RouteToTeamStep(T).run(ctx, c[0], force_signal=True)  # explicit user act is the positive signal
  return 201 { memory_id, promoted_from: m.id }             # personal copy stays; new memory links back
```

### 6g. Event timeline (for the turn in section 3)

```
turn_complete
memory_created           team:team-a  mem_team_a_23  "Nightly run 1234 … failed …"  links=[mem_team_a_17]
memory_updated           personal     mem_user_04    "Prefers CI failures grouped by test file"
# (c3 dropped silently by the locked global credential rule; visible only in the audit log)
```

And in a session with no active team:

```
turn_complete
memory_confirm           mc_12  team-a  "Nightly run 1234 … failed …"
memory_updated           personal     mem_user_04
   ... user answers keep_personal, or the window lapses ...
memory_decision_resolved mc_12  keep_personal
memory_created           personal     mem_user_31  suggested_audience=team-a
```

## 7. Skill fork and override

A user with `create-personal-skill` copies a shared skill into their own
layer under the same name. From then on the chain's last-write-wins
makes the copy shadow the base for that user
(`decisions/0006-personal-skill-overrides.md`).

### 7a. Forking through the API

```
POST /skills/global:code-review/fork?layer=personal
Authorization: Bearer <jwt>

ApiLayer.fork_skill(request):
  token  = validator.validate(request.bearer)
  base_layer, base_id = parse(request.skill_id)                # "global:code-review"
  target = request.layer                                       # personal | team:T

  # read side: must be able to use the base
  if not token.allows(base_layer, "use", "skill"): return 403
  base = layers.get(base_layer, token).skill_store.load(base_id)
  if not base: return 404
  if base.locked: return 409 { error: "locked", by: base.layer }   # cannot be overridden anywhere below

  # write side: must be able to create in the target
  if not token.allows(target, "create", "skill"): return 403
  if target is team:T and T not in token.groups: return 403
  if precedence(target) <= precedence(base_layer): return 400 "target must be a less trusted layer than the base"

  target_store = layers.get(target, token).skill_store
  if target_store.has(base.name) and not request.overwrite: return 409 { error: "exists" }

  fork = Skill.copy_of(base)
  fork.id           = f"{target}:{base.name}"
  fork.layer        = target
  fork.source       = target_store.id
  fork.locked       = False                                    # a fork can never lock
  fork.derived_from = SkillRef(id=base.id, layer=base.layer, version=base.version)

  # tools travel with the fork, but the target layer's capability ceiling applies
  excess = [t for t in fork.tools if not capability_ceiling(target).allows(t.required_capabilities)]
  if excess and not request.strip_excess: return 422 { error: "tools exceed layer capabilities", tools: [t.name for t in excess] }
  fork.tools = [t for t in fork.tools if t not in excess]
  for t in fork.tools: t.layer = target                         # so runners sandbox by the fork's layer, not the base's

  fork.version = content_hash(fork)
  target_store.writer.put(fork)
  emit_global(skill_forked, fork.id, from=base.id, by=token.subject)
  return 201 { skill_id: fork.id, derived_from: fork.derived_from, stripped_tools: [..] }
```

Editing the copy is an ordinary write; `derived_from` is untouched and
`version` is recomputed:

```
PUT /skills/personal:code-review   { instructions: "...", tools: [...] }
  -> same scope checks; capability ceiling re-applied; version = content_hash
```

### 7b. How the override takes effect in a turn

Nothing special happens. The next `on_message` runs the normal chain:

```
# global SkillDiscoveryStep applies first
ctx.skills.put("code-review", global_summary, locked=False)
#   -> stored; ctx.shadow_log["code-review"] = [global]

# team-a has no code-review; nothing

# user SkillDiscoveryStep applies later
ctx.skills.put("code-review", user_fork_summary, locked=False)
#   -> ctx.skills["code-review"] replaced; shadow_log += [user]
#   -> summary.shadows = "global:code-review"
```

`TurnContext.put` is the whole precedence rule:

```
TurnContext.put(acc, key, value, locked):
  existing = acc.get(key)
  if existing and existing.locked:
      emit(shadow_refused, acc.name, key, attempted_by=value.layer, locked_by=existing.layer)
      return False                                        # a same-name lower-trust copy is ignored
  value.locked = locked
  if existing: value.shadows = existing.id
  acc[key] = value
  return True
```

When the model later loads `code-review`, `SkillStore(user).load` returns
the fork; its tools enter `ctx.tools` as `code-review.<tool>` at the
**user** layer and its prompt fragments at the user layer. The runner
factory sandboxes script tools by *that* layer's rules, which is why the
fork endpoint re-labels tool layers.

### 7c. Locked base

`release-notes` is global and `locked: true` because it embeds the
release policy text.

```
POST /skills/global:release-notes/fork?layer=personal   -> 409 { error: "locked", by: "global" }
```

A user who bypasses the API and drops a `release-notes` skill into their
own directory does not get an override either. At discovery:

```
ctx.skills.put("release-notes", global_summary, locked=True)    # global step
ctx.skills.put("release-notes", user_copy_summary, locked=False) # user step -> False
emit(shadow_refused, "skills", "release-notes", attempted_by=user, locked_by=global)
```

The client can surface this once per session so the user learns why
their copy is silent. The user's copy remains loadable by full id
(`personal:release-notes`) if the model or user asks for it explicitly;
it just never takes the bare name.

### 7d. Staleness, detected for free by chain order

Because the base's layer applies before the fork's layer, the user step
can compare versions at discovery without an extra lookup:

```
SkillDiscoveryStep(user).apply(ctx, summaries):
  for s in summaries:
      if s.derived_from:
          base_now = ctx.skills.get(s.name)               # what the base layer just put, if any
          if base_now and base_now.id == s.derived_from.id:
              s.stale = (base_now.version != s.derived_from.version)
              s.base_version_now = base_now.version
          else:
              s.stale = None                              # base layer not in chain (no scope) or base deleted
      ctx.put("skills", s.name, s, locked=False)
```

`stale` travels in the summary the model sees and in the `skill_shadowed`
event, so both the model and the client know the copy is behind. Nothing
stops working.

```
event: skill_shadowed
data: { skill: "personal:code-review", shadows: "global:code-review", stale: true,
        base_version: "a91c…", fork_base_version: "7e02…" }
```

### 7e. Diff and rebase

```
GET /skills/personal:code-review/diff

ApiLayer.diff_skill(request):
  fork = user_store.load(...)
  base_store = layers.get(fork.derived_from.layer, token).skill_store
  base_now   = base_store.load(fork.derived_from.id)
  base_then  = base_store.load_version(fork.derived_from.id, fork.derived_from.version)   # git-backed stores can; http stores may not
  if base_then is None:
      return 200 { mode: "two_way", base_now_vs_fork: diff(base_now, fork) }
  return 200 { mode: "three_way",
               upstream_changes: diff(base_then, base_now),     # what the maintainers changed
               local_changes:    diff(base_then, fork),         # what the user changed
               conflicts:        merge3(base_then, base_now, fork).conflicts }
```

```
POST /skills/personal:code-review/rebase

ApiLayer.rebase_skill(request):
  ... same loads ...
  if base_now.locked: return 409 "base is now locked; fork can no longer override"   # policy changed underneath
  m = merge3(base_then, base_now, fork)                       # per field: instructions, tools by name, fragments by id
  if m.conflicts and not request.resolutions: return 409 { conflicts: m.hunks }
  merged = m.apply(request.resolutions)
  merged.derived_from.version = base_now.version
  merged = enforce_capability_ceiling(merged, target)
  merged.version = content_hash(merged)
  user_store.writer.put(merged)
  return 200 { skill_id, derived_from }
```

Diff and merge are structural, per field: instructions as text, tools
matched by name, prompt fragments by id. A tool the maintainers added
upstream appears in the fork after rebase; a tool the user deleted stays
deleted unless upstream changed it, which is a conflict.

### 7f. Unfork

```
DELETE /skills/personal:code-review
  -> user_store.writer.delete(...)
  -> next on_message: only global puts "code-review"; no shadow; base is back
```

### 7g. Team-level fork, and a fork of a fork

```
POST /skills/global:code-review/fork?layer=team:team-a       # needs create-team-skill, team-a in groups
  -> team:team-a:code-review  derived_from=global:code-review
  -> every team-a member's chain now applies it after global, so it shadows global for the whole team

POST /skills/team:team-a:code-review/fork?layer=personal      # a member forks the team version
  -> personal:code-review  derived_from=team:team-a:code-review
  -> chain: global puts, team-a overwrites, user overwrites; shadow_log = [global, team-a, user]
  -> staleness compares against the *team* version, because that is derived_from
```

A team fork can be marked `locked` by someone with `create-team-skill`
only if the team's store adapter supports it; a locked team fork then
blocks personal forks below it while itself still yielding to a locked
global (which would have prevented the team fork in the first place).

### 7h. Event timeline

```
# forking
skill_forked             personal:code-review  from global:code-review

# next turn, on_message
context_ready
skill_shadowed           personal:code-review shadows global:code-review  stale=false

# weeks later, maintainers update global:code-review
context_ready
skill_shadowed           personal:code-review shadows global:code-review  stale=true

# user runs rebase via the client; next turn
skill_shadowed           personal:code-review shadows global:code-review  stale=false
```

## 8. Tool audit and classifier rules

The audit of one tool call is the `on_tool_call` hook: each layer's
`StaticRulesStep` runs that layer's rules, then the service-appended
`ModelAuditStep` reviews anything still undecided. Sections 3c and 5a
showed the outline; this section fills in the rules themselves.

### 8a. Rule format and loading

Rules are data in each layer's rule source. A file-backed source:

```yaml
# /etc/agent/classifier-rules.yaml   (global layer)
rules:
  - id: no-shell-from-shared-skills
    role: tool_audit
    kind: deny
    locked: true
    match: { tool: "shell", tool_layer_in: [global, team] }       # shell defined by a shared skill? never
    reason: "shell may only come from the built-in tool source"

  - id: high-risk-needs-approval
    role: tool_audit
    kind: require_approval
    locked: true
    match: { effective_risk_gte: high }

  - id: read-only-tools-are-fine
    role: tool_audit
    kind: allow
    locked: true                                                 # shields from lower-layer nagging
    match: { tool_in: [ci_status, read_file, search_memory] }

  - id: audit-instructions
    role: tool_audit
    kind: prompt_fragment
    body: |
      Approve a call only if the user's most recent request plainly asks for
      the effect the tool will have. Deleting or overwriting is never implied
      by "check", "look at", or "show".

  - id: no-credentials
    role: memory_mining
    kind: drop
    locked: true
    match: { kind: credential }            # or body_matches: secret_patterns
```

```
ClassifierRule {
  id, layer, source, role: tool_audit | memory_mining
  kind:   deny | require_approval | allow | prompt_fragment | threshold | drop | redact | retag | auto_accept | disable
  match:  Match                              # fixed fields, no expression language (v1)
  locked: bool
  body?:  text                               # prompt_fragment
  value?: any                                # threshold
  reason?: text
}

Match (tool_audit) = all of the present fields must hold:
  tool | tool_in | tool_glob        # by registered name, e.g. "nightly-triage.*"
  tool_layer | tool_layer_in         # layer that put the tool into ctx.tools
  risk_gte | effective_risk_gte      # declared risk vs. risk after escalation (8c)
  execute_on                         # server | client
  args_match: { field: glob }        # shallow, string-valued args only
  skill                              # owning skill name, if any
```

Loading is the `RuleLoadStep` in `on_message`; each layer appends its own
rules and the accumulator remembers the layer:

```
RuleLoadStep(layer).apply(ctx, rules):
  for r in rules: r.layer = layer; ctx.rules.append(r)
# ctx.rules is a list in outer-chain order; no global re-sort
```

### 8b. Which rule wins: restrictive rules accumulate, widening rules act at their layer

Static rule kinds split by direction, the same split as mining rules
(§6c):

| Direction   | Kinds                       | Scope of effect                                   |
|-------------|-----------------------------|---------------------------------------------------|
| restrictive | `deny`, `require_approval`  | any layer may add one; **no layer can remove another layer's** |
| widening    | `allow`                     | pre-empts the model audit; yields to any restrictive rule unless **locked** |
| advisory    | `prompt_fragment`           | assembled into the auditor's prompt in layer order |

So:

- A user can add `require_approval` for `shell` even though global allows
  it (unlocked): the user gets asked.
- A user cannot remove global's `high-risk-needs-approval`: it is
  restrictive, and restrictions accumulate.
- Global's `read-only-tools-are-fine` is a **locked allow**: it shields
  those tools from lower-layer restrictions, so a team cannot make
  `ci_status` require approval. Unlock it and the team could.
- Two peer teams with conflicting rules need no merge strategy: their
  restrictions both apply, and their allows both yield. The only true
  conflict is two locked allows for the same call, which is not a
  conflict at all.

The per-layer step, replacing the sketch in §5a:

```
StaticRulesStep(layer).run(ctx, call):
  a = ctx.audit[call.id]                                   # per-call state: decided, shielded, verdict
  for r in ctx.rules.of_layer(layer).for_role("tool_audit").ordered_by(kind: deny, require_approval, allow):
      if not r.match(call, ctx): continue
      match r.kind:
        deny:
            if a.shielded: continue                        # locked allow from an earlier layer wins
            a.verdict = DENY(rule=r); a.decided = True
            return Stop(Deny(reason=r.reason or r.id))
        require_approval:
            if a.shielded or a.decided: continue           # already decided (approved earlier, or denied) or shielded
            a.asked_by = r
            return Stop(RequireApproval(ApprovalRequest.from_rule(r, call)))
        allow:
            if a.decided: continue
            a.verdict = ALLOW(rule=r); a.decided = True
            a.shielded = r.locked                          # only a locked allow protects against later layers
            # not a Stop: later layers still get to add restrictions unless shielded
  return Continue
```

Within one layer, `deny` is checked before `require_approval` before
`allow`, so a layer's own most restrictive matching rule wins.

### 8c. Service-level gates that run before any layer

The service prepends two steps to every `on_tool_call` chain; they are
not rules and cannot be configured away by a layer:

```
RiskEscalationStep.run(ctx, call):
  d = ctx.tools.resolve(call.name)
  bump = config.tools.risk_escalation.get(d.layer, 0)      # e.g. team: +1, global-from-skill: +1, session/client: +1
  d.effective_risk = clamp(d.risk + bump)                  # what `effective_risk_gte` matches on
  return Continue

ScopeGateStep.run(ctx, call):
  d = ctx.tools.resolve(call.name)
  if d.execute_on == server and not ctx.token.scopes.has("run-server-tools"):     return Stop(Deny("no run-server-tools scope"))
  if d.effective_risk >= high and not ctx.token.scopes.has("run-high-risk-tools"): return Stop(Deny("no run-high-risk-tools scope"))
  return Continue
```

`bypass-tool-audit` (service accounts) skips only the `ModelAuditStep`;
static rules and scope gates still apply.

### 8d. The model auditor

```
ModelAuditStep.run(ctx, call):
  a = ctx.audit[call.id]
  if a.decided: return Continue
  if ctx.token.scopes.has("bypass-tool-audit"): a.verdict = ALLOW(reason="bypass scope"); return Continue
  if engine.kind == rules_only:
      return default_for(config.classifier.undecided_default, call)    # allow | ask | deny, may depend on effective_risk

  key = (hash(ctx.session.conversation.window(n=config.classifier.window)), call.name, hash(call.args))
  v = cache.get(key) or engine.audit(AuditInput(
        window        = ctx.session.conversation.window(n),           # recent user + assistant turns, tool results summarised
        call          = call, definition = ctx.tools.resolve(call.name),
        provenance    = { tool_layer, owning_skill, skill_layer, skill_forked_from },
        prior_calls   = [c for c in ctx.calls_this_turn if c.id != call.id],
        instructions  = assembler.assemble(ctx.rules.prompt_fragments(role="tool_audit"))   # layered, in layer order
      ))
  cache.put(key, v)

  # engine returns a structured verdict; thresholds turn confidence into a decision
  v = apply_thresholds(v, config.classifier.thresholds)       # e.g. allow if p_requested >= .85, deny if <= .15, else ask
  a.verdict = v; a.decided = True
  audit_log.write(ctx, call, v, rules_considered=ctx.rules.of_role("tool_audit"), static_path=a.trace)
  match v.decision:
    ALLOW:    return Continue
    DENY:     return Stop(Deny(reason=v.reason))
    ASK_USER: return Stop(RequireApproval(ApprovalRequest.from_verdict(v, call)))

AuditOutput {                                       # the engine's structured answer
  p_requested: float                                # "the user's latest request plainly asks for this effect"
  effect_summary: text                              # one line, shown to the user on ask
  reason: text
  mismatch?: { asked_for: text, tool_does: text }   # populated when p_requested is low
}
```

The auditor's prompt is layered like the agent's: global's
`audit-instructions` fragment, then a team's ("calls to
`nightly-triage.*` during incident hours are expected"), then a user's.
Fragments are advisory to the model; they never decide by themselves.

Speculation and cancellation (from §3b): the step may start when the tool
name and partial args are known; the cache key includes the args hash, so
a change in the final args simply misses the cache and re-runs.

### 8e. Dry run and calibration

```
DryRunStep(inner).run(ctx, call):
  r = inner.run(ctx, call)
  if r is Stop:
      audit_log.write(ctx, call, would_have=r.action, dry_run=True)
      emit(audit_dry_run, call.id, would_have=r.action.kind, by=r.action.source)
      ctx.audit[call.id].decided = False                     # let later steps and the model auditor proceed normally
      return Continue
  return r
```

Wrapping a layer's `StaticRulesStep` in `DryRunStep` is how a new global
rule is trialled: verdicts are logged, nothing is blocked, and
`GET /classifier/verdicts?dry_run=true` shows what it would have caught.

### 8f. Worked cases

Chain: global, team-a, user, session. Rules as in 8a plus team-a
`allow nightly-triage.*` (unlocked) and user `require_approval shell`
(unlocked). Token has `run-server-tools` and `run-high-risk-tools`.

| Call                               | Path                                                                                      | Result |
|------------------------------------|-------------------------------------------------------------------------------------------|--------|
| `ci_status(job=…)`                 | risk low → global locked allow → shielded → team/user restrictions skipped → model audit skipped | run    |
| `shell(cmd="ls")` (built-in, global)| risk medium → no global match → user `require_approval shell` → ask                       | ask user |
| `shell(cmd="rm -rf build/")`       | risk high → global `high-risk-needs-approval` (restrictive, locked) → ask; user rule never reached | ask user, `remember: session` not offered |
| `shell` defined by a team skill    | RiskEscalation team +1 → global `no-shell-from-shared-skills` deny                        | denied |
| `nightly-triage.fetch_log(...)`    | risk medium (team +1 → high) → global `high-risk-needs-approval` matches first → ask; team's allow is later and unlocked | ask user |
| same, with `run-high-risk-tools` missing | ScopeGate denies before any rule                                                    | denied |
| `open_in_editor(path)` (session, client) | risk low, escalation session +1 → medium → no static match → model audit → p=.92 → allow | run on client |
| `delete_file(path)` after user said "show me the file" | no static match → model audit → p=.08, mismatch{asked_for:"show", tool_does:"delete"} → deny | denied, reason shown |

The fifth row is worth noticing: a team's unlocked `allow` cannot get a
high-risk tool past global's locked `require_approval`. If the team truly
needs that, the fix is upstream: global lowers the escalation for that
team's tool layer, or the tool's declared risk is reduced by whoever
maintains it.

### 8g. Authoring rules

```
POST /classifier/rules?layer=personal    { id, role, kind, match, locked?: false, reason }
  -> requires create-personal-classifier-rule; `locked` is refused below global unless the layer's config allows it
  -> written to the user's rule source; takes effect at the next on_message (RuleLoadStep re-reads; sources may cache with TTL)
POST /sources/refresh                    -> forces every RuleLoadStep source to re-read now

GET  /classifier/rules                   -> ctx.rules as the next turn would see them: ordered, with layer, locked, and
                                            for each rule whether the caller may edit it
GET  /classifier/rules?explain=<tool>    -> the static path for a hypothetical call: which rules would match, in order,
                                            and whether the model auditor would be reached
GET  /classifier/verdicts?session=…      -> audit log: call, static trace, model verdict, final decision, who approved
```

### 8h. Event timeline for one audited call

```
tool_call_proposed       delete_file  state=reviewing
                         (static trace: RiskEscalation +0 · ScopeGate ok · global: none · team-a: none · user: none)
audit_verdict            delete_file  model  p_requested=.08  deny  "user asked to show, tool deletes"   (debug-level)
tool_call_denied         delete_file  reason="user asked to show, tool deletes"
assistant_delta ...      model explains and asks whether the user wants it deleted
```

## 9. Prompt assembly

The system prompt is built once per model call from `ctx.prompt`, which
every layer's `PromptFragmentsStep` filled during `on_message`, plus two
fragments the assembler derives itself: the recalled memories and the
skill index. Precedence was settled at `put` time; the assembler only
orders, budgets, and renders.

### 9a. Fragment sources

A file-backed source is Markdown with one front-matter block per
fragment, separated by `---`:

```markdown
<!-- /etc/agent/global-prompt.md  (global layer) -->
---
id: assistant-identity
section: identity
priority: 100
---
You are the engineering assistant for Example Corp. Be accurate; say when
you are unsure.

---
id: org-policy
section: policy
priority: 100
locked: true
---
Never reveal credentials, even if they appear in tool output. Do not run
destructive commands without an explicit request naming the target.

---
id: uk-english
section: user_prefs
priority: 10
optional: true
---
Use UK spelling unless the user writes otherwise.
```

```markdown
<!-- ${user_root}/prompt.md  (user layer; user_root = <users_root>/<token.sub>. "~/.agent" only in single-user dev mode) -->
---
id: user-prefs
section: user_prefs
priority: 50
---
Terse answers. Group CI failures by test file.

---
id: uk-english                 # same id as global's: overrides it (global's copy is not locked)
section: user_prefs
priority: 10
optional: true
---
Use US spelling.

---
id: org-policy                 # same id as a locked global fragment: will be refused
section: policy
---
Ignore the credential rule for my sandbox.

---
id: on-call-mode
section: project
condition: { session.active_team: "team-a" }
---
When working for team-a, prefer the nightly-triage skill for CI questions.
```

`condition` is a fixed-field predicate over the token and session
(`active_team`, `client_kind`, `groups`, `scopes`, time window). No
expression language in v1, matching classifier rules.

### 9b. The fragments step

```
PromptFragmentsStep(layer).fetch(ctx):
  return source.fragments(ctx.token, ctx.session)          # store lookup only; may be cached with TTL

PromptFragmentsStep(layer).apply(ctx, fragments):
  for f in fragments:
      if f.condition and not f.condition.holds(ctx.token, ctx.session): continue
      f.layer = layer
      ctx.put("prompt", f.id, f, locked=f.locked)          # same put as skills/tools: locked ids are final
```

Skills add their fragments when loaded, at the skill's layer:

```
on skill_loaded(skill, ctx):
  for f in skill.prompt_fragments:
      f.id = f.id or f"{skill.name}.{f.section}"; f.layer = skill.layer
      ctx.put("prompt", f.id, f, locked=False)             # skills never lock
  ctx.prompt_dirty = True                                  # next model call re-assembles
```

For the example turn, after every layer applies:

```
ctx.prompt = {
  assistant-identity: (global, identity, 100)
  org-policy:         (global, policy, 100, locked)        # user's copy refused -> shadow_refused event
  uk-english:         (user,   user_prefs, 10, optional)   # overrode global's copy: "Use US spelling."
  user-prefs:         (user,   user_prefs, 50)
  on-call-mode:       (user,   project)                    # condition held: active_team == team-a
  session-note:       (session, session)                   # CLI flag: --instruct "we're mid-incident"
}
# after load_skill("nightly-triage"):
  nightly-triage.project: (team:team-a, project)
```

### 9c. Derived fragments

Recalled memories and the skill index are not stored anywhere; the
assembler generates them so they are budgeted like everything else:

```
derive(ctx) -> [PromptFragment]:
  mems = PromptFragment(id="__memories", section="memories", layer=session, priority=0,
                        optional=True, shrinkable=True,
                        body=render_memories(ctx.memories))          # already through MergeStrategy + RecallPolicy
  idx  = PromptFragment(id="__skills", section="skills", layer=session, priority=0,
                        optional=False, shrinkable=True,
                        body=render_skill_index(ctx.skills))         # name + one-line description + tool names
  return [mems, idx]
```

`shrinkable` means the assembler can ask for a smaller rendering before
dropping the fragment: fewer memories, or the skill index reduced to
names only.

### 9d. Ordering

```
SECTION_ORDER = [identity, policy, project, skills, tools, user_prefs, memories, session]
#                ^ stable across turns and users ................. ^ volatile per turn

assemble(ctx, budget) -> AssembledPrompt:
  frags = list(ctx.prompt.values()) + derive(ctx)
  by_section = group(frags, key=section)
  for sec in by_section: by_section[sec].sort(key=(-priority, layer_precedence, id))
  frags = budget_fit(by_section, budget)                    # 9e
  text  = render_strategy.render(SECTION_ORDER, by_section)  # 9f
  return AssembledPrompt(text, fragments_used=frags, hash=hash(text))
```

The section order is deliberate: stable, shared content first (identity,
policy, team project notes, skill index), volatile per-user and per-turn
content last (preferences, memories, session notes). Providers that
cache prompt prefixes then reuse the shared prefix across every user
in the deployment, and across every call within a turn.

### 9e. Budgeting

```
budget_fit(by_section, budget):
  total = tokens(by_section)
  if total <= budget: return by_section

  # 1. shrink before dropping
  for f in shrinkable fragments, least trusted layer first:
      while total > budget and f.can_shrink(): f.shrink(); total = tokens(by_section)
      # __memories: RecallPolicy.fewer(); __skills: names-only

  # 2. drop optional fragments: lowest priority first, then least trusted layer first
  for f in sorted(optional fragments, key=(priority, -layer_precedence)):
      if total <= budget: break
      remove(f); dropped.append(f); total = tokens(by_section)

  # 3. drop non-optional, non-locked fragments the same way (this is a misconfiguration, but never fail the turn)
  ...

  # 4. locked fragments are never dropped. If they alone exceed the budget, the turn fails:
  if tokens(locked) > budget: raise PromptBudgetExceeded(locked_ids)

  emit(prompt_truncated, dropped=[f.id for f in dropped], shrunk=[...])
  return by_section
```

`budget` comes from the model adapter (context window minus room for
conversation and output) and may be lowered per deployment.

### 9f. Rendering

```
TaggedSectionRender.render(order, by_section) -> text:
  out = []
  for sec in order:
      if sec not in by_section: continue
      out.append(f"<{sec}>")
      for f in by_section[sec]:
          out.append(f"<!-- {f.layer}:{f.id} -->" if config.prompt.provenance_comments else "")
          out.append(f.body.strip())
      out.append(f"</{sec}>")
  return "\n".join(out)

PlainTextRender: same, with "## Section" headers and no tags.
```

Rendered for the example (provenance comments on, memories shortened):

```
<identity>
You are the engineering assistant for Example Corp. Be accurate; say when you are unsure.
</identity>
<policy>
Never reveal credentials, even if they appear in tool output. Do not run destructive commands
without an explicit request naming the target.
</policy>
<project>
<!-- user:on-call-mode -->
When working for team-a, prefer the nightly-triage skill for CI questions.
<!-- team:team-a:nightly-triage.project -->
Nightly triage: the job is build-nightly-linux; known-flaky tests are listed in flaky.txt in the repo.
</project>
<skills>
- code-review (personal, overrides global; stale): review a diff against team conventions. tools: code-review.lint
- nightly-triage (team-a, loaded): triage a failed nightly run. tools: nightly-triage.fetch_log
- release-notes (global, locked): draft release notes from merged PRs.
</skills>
<user_prefs>
<!-- user:user-prefs -->
Terse answers. Group CI failures by test file.
<!-- user:uk-english -->
Use US spelling.
</user_prefs>
<memories>
- [team-a] nightly job name is build-nightly-linux
- [team-a] flaky test: test_upload_retry
- [personal] prefers CI failures grouped by test file
</memories>
<session>
we're mid-incident
</session>
```

Provenance comments are off by default for the agent and on for
`GET /prompt`, so a user can see exactly which layer said what.

### 9g. Re-assembly within a turn and caching

```
before each model call in the agent loop:
  if ctx.prompt_dirty or ctx.memories_dirty:
      ctx.assembled = assemble(ctx, budget); ctx.prompt_dirty = ctx.memories_dirty = False
  model.complete(system=ctx.assembled.text, ...)
```

Loading a skill mid-turn is the common reason to re-assemble; the
stable sections do not change, so a provider prefix cache still hits up
to the `<skills>` section. Assembled prompts are also cached across
turns keyed by `(hash of ctx.prompt ids+versions, memories hash)` so a
session that loads nothing new pays no assembly cost.

### 9h. The classifier's prompt

Same assembler, different role filter:

```
audit_prompt = assemble_from(ctx.rules.prompt_fragments(role="tool_audit"),   # kind: prompt_fragment rules, in layer order
                             derived=[], order=[identity, policy, project, user_prefs], budget=config.classifier.prompt_budget)
```

The classifier never sees the agent's memories or skill index; it sees
the conversation window and the tool call (§8d).

### 9i. Inspecting

```
GET /prompt                       -> the text the next turn would send, provenance comments on
GET /prompt/fragments             -> every fragment in ctx.prompt: id, layer, section, priority, locked, optional,
                                     condition (and whether it currently holds), shadows, refused (with locked_by)
GET /prompt?budget=4000           -> what budgeting would drop at that size
```

### 9j. Event timeline

```
context_ready            ... prompt: 6 fragments, 1 refused (org-policy by user), 0 dropped
shadow_refused           prompt  org-policy  attempted_by=user  locked_by=global
skill_loaded             nightly-triage  (+1 prompt fragment; re-assembled before next model call)
prompt_truncated         dropped=[uk-english]  shrunk=[__memories: 5->3]      # only when over budget
```

## 10. Memory recall and search

Recall happens inside `on_message`: every layer's `MemorySearchStep`
queries its own store concurrently, results are appended to
`ctx.memories`, and two merge steps run afterwards: `MemoryMergeStep`
(rank fusion and cross-layer dedupe) and `RecallPolicy` (what actually
enters the prompt). No model call is involved.

### 10a. Building the query

```
build_recall_query(ctx) -> RecallQuery:
  msg  = ctx.inbound.text
  prev = ctx.session.conversation.last_assistant_message()?.text[:400]     # continuity for "and the other one?"
  return RecallQuery(
      texts   = [msg] + ([prev] if prev else []),           # stores may embed each; first is primary
      hints   = { active_team: ctx.session.active_team,
                  loaded_skills: [s.name for s in ctx.skills_loaded_last_turn],
                  client_kind: ctx.session.client_kind },
      kinds   = None,                                        # all kinds unless a rule narrows (below)
      k       = config.memory.k_per_store,                   # e.g. 8
      exclude_superseded = True)
```

Rules can narrow recall the same way they narrow mining: a
`memory_recall` role with `kinds`, `tags`, or `disable` (restrictive,
any layer). A user who does not want feedback memories recalled in a
shared screen session can say so in their rule file.

### 10b. Per-layer search

```
MemorySearchStep(layer).fetch(ctx):
  q = ctx.recall_query
  if ctx.rules.for_role("memory_recall").disables(layer): return []
  hits   = store.search(q, ctx.token, opts={k: q.k, kinds: q.kinds, exclude_superseded: True})
  pinned = store.list(ctx.token, filter={pinned: True})     # always recalled, regardless of query
  return hits + [ScoredMemory(m, score=None, pinned=True) for m in pinned]
  # store.search is whatever the adapter does: vector, keyword, hybrid, remote service.
  # The user store keys on token.subject; the team store is the one built for this team_id.

MemorySearchStep(layer).apply(ctx, results):
  for r in results:
      r.layer = layer; r.rank = r.rank or position_in(results)
      ctx.memories.append(r)                                 # append only; no precedence, no dedupe here
```

Each step is wrapped `FailOpenStep` with a per-step timeout (config,
e.g. 300 ms): a slow team store yields nothing for this turn and a
`context_ready` timing shows it. Scores are **store-local** and not
comparable across stores, which is why the merge uses ranks.

For the example turn:

```
global   (read-only org memory):  0 hits                                     22 ms
team-a:  [ "nightly job name is build-nightly-linux"  rank 1  score .81
           "flaky test: test_upload_retry"              rank 2  score .66
           "build cache lives on nfs://ci-cache"        rank 3  score .41 ]   210 ms
user:    [ "prefers CI failures grouped by test file"   rank 1  score .74
           "works on the upload service"                rank 2  score .52
           (pinned) "timezone: Europe/London" ]                              6 ms
```

### 10c. Merge: rank fusion, then cross-layer dedupe

```
MemoryMergeStep.merge(ctx) -> [ScoredMemory]:
  by_layer = group(ctx.memories, key=layer)
  fused = merge_strategy.merge(by_layer)                     # default: reciprocal rank fusion
  return dedupe_across_layers(fused)

ReciprocalRankFusion.merge(by_layer, k=60, boost=config.memory.layer_boost):
  score = {}
  for layer, results in by_layer:
      for r in results:
          if r.pinned: score[r.id] = +inf; continue
          score[r.id] += (1 / (k + r.rank)) * boost.get(layer, 1.0)   # e.g. user 1.2, team 1.1, global 1.0
  return sorted(all r, key=-score[r.id])

dedupe_across_layers(fused):
  seen = {}
  for r in fused:
      key = r.memory.promoted_from or r.memory.id            # a promoted memory exists in personal and team
      if key in seen: seen[key].also_in.append(r.layer); continue
      if near_duplicate(r, seen.values()): continue          # same normalised body across stores
      seen[key] = r
  return list(seen.values())
```

RRF is the default because it needs no comparable scores. A deployment
that injects one shared embedder into every store (`config.memory.embedder:
shared`) may switch `merge_strategy` to score fusion, and a deployment
with a cross-encoder may add a rerank pass; neither changes the steps.

Fused order for the example (boosts user 1.2, team 1.1):

```
1. (pinned) timezone: Europe/London                       user    +inf
2. prefers CI failures grouped by test file               user    1/(60+1)*1.2 = .0197
3. works on the upload service                            user    1/(60+2)*1.2 = .0194
4. nightly job name is build-nightly-linux                team-a  1/(60+1)*1.1 = .0180
5. flaky test: test_upload_retry                          team-a  1/(60+2)*1.1 = .0177
6. build cache lives on nfs://ci-cache                    team-a  1/(60+3)*1.1 = .0175
```

Note line 3: a rank-2 user hit outranks a rank-1 team hit only because
of the layer boost. With all boosts at 1.0, RRF interleaves strictly by
rank (user 1, team 1, user 2, team 2, ...).

### 10d. Recall policy: what enters the prompt

A pipeline of small steps over the fused list:

```
RecallPolicy.select(fused, ctx, budget_tokens) -> [Memory]:
  xs = fused
  xs = drop_superseded(xs)                                   # belt and braces; stores should already exclude
  xs = drop_below(xs, min_fused_score=config.memory.min_score)          # pinned exempt
  xs = recency_decay(xs, half_life_days=config.memory.half_life)        # optional; kind=feedback exempt
  xs = diversify(xs, max_per_kind=config.memory.max_per_kind)           # avoid 6 "project" notes and no "feedback"
  xs = xs[:config.memory.max_recalled]                                  # e.g. 6
  xs = expand_links(xs, hops=1, max=2, mark=optional)                   # pull directly linked memories, low priority
  xs = fit_tokens(xs, budget_tokens)                                    # drop from the tail; pinned last to go
  ctx.recalled = xs
  return xs
```

`budget_tokens` is what the prompt assembler grants the `__memories`
derived fragment (§9c). When the assembler needs to shrink, it calls
`RecallPolicy.fewer(ctx)` which pops from the tail of `ctx.recalled`,
so shrinking and selection agree.

For the example: `min_score` drops "build cache" (.0175 < .0176),
`max_per_kind` keeps at most 3 `project` memories, the cap keeps 5, and
the token fit keeps all 5. The `<memories>` block in §9f shows three of
them because that rendering was shortened.

### 10e. Citations

Every recalled memory carries its id and layer into the prompt as an
inline marker the model can echo:

```
<memories>
- [m:team-a/17] nightly job name is build-nightly-linux
- [m:user/04] prefers CI failures grouped by test file
</memories>
```

```
on assistant_message(text):
  cited = extract_markers(text, pattern="[m:<layer>/<id>]")
  emit(assistant_message, text=strip_markers(text), citations=[{id, layer, title} for id in cited])
  for id in cited: store_for(id).touch(id, used_at=now())    # reinforces recency; feeds decay
```

The client can render "from team-a memory" links; the audit log records
which memories influenced which answer.

### 10f. Explicit search and writes over the API

```
GET /memories?q=nightly&kinds=project&layers=team-a,personal&k=20&include_superseded=false

ApiLayer.search_memories(request):
  token = validate(...)
  ctx   = TurnContext.lightweight(token, session=None)       # no session: active_team absent, pinned excluded
  chain = layers.chain_for(token)
  results = concurrent(MemorySearchStep(layer).fetch(ctx) for layer in chain if layer in request.layers)
  fused   = MemoryMergeStep.merge(results)
  return fused[:request.k]                                   # no RecallPolicy: this is a browser, not a prompt
```

```
POST /memories { kind, body, tags, audience: personal | team:T, links? }

ApiLayer.create_memory(request):
  c = MemoryCandidate.from_request(request, confidence=1.0, source="explicit")
  c = prefilter(ctx, [c])                                    # locked restrictive mining rules still apply (no credentials)
  if not c: return 422
  route(ctx, c[0], explicit=True)                            # explicit=True is a positive signal for team routing:
                                                             # the user typed it and named the team; no confirm needed
  return 201 { memory_id, layer }
```

### 10g. Embedding ownership

Resolved by the flow above:

- The **store owns embedding**. It embeds on `put` and on `search` with
  whatever it likes; the core hands it text.
- A store **may accept** a shared `Embedder` (config) so several stores
  share one model; this makes scores comparable but is never required.
- Cross-store ranking uses **ranks** (RRF) by default, so nothing depends
  on comparable scores.
- Near-duplicate detection across layers (§10c) and at write time (§6d)
  uses normalised-text hashing plus, where the store exposes it, a
  similarity threshold; it never requires a model call.

### 10h. Event timeline

```
context_ready            memories: global 0 (22ms) · team-a 3 (210ms) · user 2+1 pinned (6ms) · fused 6 · recalled 5
memory_recalled          [m:user/pin-1, m:user/04, m:team-a/17, m:user/09, m:team-a/22]     (debug-level)
assistant_delta ...
assistant_message        citations=[m:team-a/17, m:user/04]
```

## 11. Session lifecycle and reconnect

A session is one client's private conversation with the service. It
owns a conversation, the token that created it, an event log, pending
approvals, and the outer chain built for that token. It has no
subscribers to fan out to; reconnect means *replacing* its one stream.

### 11a. States

```
            POST /sessions                GET /events              stream closes
  (none) ───────────────▶ created ───────────────▶ active ◀──────────────▶ idle
                                                     │  ▲                     │
                                        turn running │  │ reconnect           │ idle_timeout
                                                     ▼  │                     ▼
                                                  (turn states)             ended ──▶ purged (retention)
                                                                              ▲
                                    DELETE /sessions/{id}  or  POST /sessions/{id}/end
Session {
  id, owner (sub), token (claims + encrypted raw), created_at, last_seen_at
  state:       created | active | idle | ended
  turn:        { id, state: idle | running | awaiting_approval | cancelling, started_at }
  active_team, client_kind, client_tools
  conversation: Conversation
  events:      EventLog                  # append-only, seq-numbered, bounded (retain last N or last T)
  approvals:   ApprovalStore             # pending + answered
  unmined_turns: [turn_id]
  chain:       OuterChain?               # rebuilt lazily; not persisted
}
```

Only one stream is expected per session. Everything else is derived.

### 11b. Opening (or re-opening) the stream

```
GET /sessions/{id}/events
Authorization: Bearer <jwt>
Last-Event-ID: 4127                       # optional; omitted on first open

ApiLayer.open_stream(request):
  token   = validator.validate(request.bearer)
  session = sessions.load(request.session_id) or 404
  if session.owner != token.subject: return 403
  refresh_token(session, token)                                   # 11d

  if session.stream:                                              # a previous connection is still open
      session.stream.send(stream_replaced, by=request.connection_id)
      session.stream.close()                                      # the old client stops receiving; no fan-out

  stream = SSEStream(request, heartbeat_s=config.api.heartbeat_s)   # ": keepalive" comments every N s
  since  = int(request.headers["Last-Event-ID"] or 0)
  if since and since < session.events.oldest_seq:
      stream.send(replay_gap, from=since, oldest=session.events.oldest_seq)   # client should refetch state
  for e in session.events.since(since): stream.send(e)             # replay, including approval_requested still pending
  stream.send(session_state, turn=session.turn, pending_approvals=session.approvals.pending_ids(),
              active_team=session.active_team, chain=[l.name for l in session.chain])
  session.stream = stream; session.state = active; session.last_seen_at = now()
  stream.on_close(lambda: (session.stream = None, session.state = idle, session.last_seen_at = now()))
```

`session_state` is sent after replay on every open, so a client can
rebuild its view without inspecting the replayed events at all.

### 11c. Disconnect during a turn

Nothing server-side depends on the stream. Events are appended to the
log first and pushed second:

```
emit(session, event):
  event.seq = session.events.append(event)        # durable before delivery
  if session.stream: session.stream.send(event)   # best-effort delivery

# turn keeps running; tool calls execute; approvals park; mining is scheduled
```

Reconnect replays from `Last-Event-ID`, so the client sees the deltas it
missed, any `approval_requested` still pending, and finally
`session_state` saying `turn.state = awaiting_approval`. Answering the
approval works exactly as in §5c; it never needed the stream.

If the client reconnects with a *new* session instead (it lost its
session id), the old session goes idle and eventually ends (11e); its
pending approvals time out to deny; its unmined turns are mined at end.

### 11d. Token refresh and expiry

```
refresh_token(session, token):
  if token.subject != session.token.subject: raise Forbidden        # a session never changes owner
  if token.raw == session.token.raw: return
  changed = (token.scopes != session.token.scopes) or (token.groups != session.token.groups)
  session.token = token
  if changed:
      session.chain = layers.chain_for(token, session)                 # e.g. joined a team, lost a scope
      if session.active_team and session.active_team not in token.groups: session.active_team = None
      emit(session, chain_rebuilt, layers=[...], active_team=session.active_team)
```

Every request may carry a newer token; the first one that does swaps it
in. A running turn keeps the token it started with (captured in
`ctx.token`), so its verdicts and writes are consistent.

Expiry while the stream is open:

```
stream watchdog:
  if session.token.expires_at - now() < config.api.token_warning_s:  send(token_expiring, at=expires_at)
  if session.token.expires_at <= now():
      send(auth_expired); close()                    # client refreshes with the IdP and reopens with Last-Event-ID
      # no turn may start; a running turn finishes on its captured token
```

### 11e. Idle timeout and end

```
sweeper (every minute):
  for session in sessions.where(state=idle, last_seen_at < now() - config.sessions.idle_timeout):
      end_session(session, reason="idle")

POST /sessions/{id}/end     or     DELETE /sessions/{id}
  -> end_session(session, reason="client")             # DELETE also skips retention: purge immediately after end

end_session(session, reason):
  if session.turn.state in (running, awaiting_approval):
      cancel_turn(session)                                          # pending approvals -> Deny("session ended")
  await drain(session)                                              # running mining jobs finish

  # 1. mine what is left, if the token still works
  if session.unmined_turns and session.token.valid():
      MiningJob(session, turns=session.take_unmined_turns()).run_now()
  else: audit_log.write(session, "unmined turns dropped", count=len(session.unmined_turns))

  # 2. optional session-summary memory (config.sessions.summary: off | personal | routed)
  if config.sessions.summary != off and session.conversation.turns >= config.sessions.summary_min_turns and session.token.valid():
      c = miner.summarise(session)                                    # kind=episodic, body="what was worked on / decided / left open"
      c.audience = team:session.active_team if (config.sessions.summary == routed and session.active_team) else personal
      c.tags += ["session-summary", session.id]
      route(TurnContext.from_session(session), c)                    # normal on_memory_candidate chain; may downgrade to personal

  # 3. retention
  session.state = ended; session.ended_at = now(); session.stream?.send(session_ended, reason); session.stream?.close()
  match config.sessions.retain_transcript:
    none:      session.conversation = None; session.events = None
    duration:  purge_at = now() + config.sessions.retention
  sessions.save(session)
```

Nothing about a session is shared, so ending one affects no other
session. What survives is only what was written to memory stores.

### 11f. Persistence and service restart

```
SessionStore {                                   # adapter: memory (dev) | sqlite (single node) | postgres/redis (multi node)
  save(session), load(id), list(owner, state?), where(...), purge(id)
}
persisted:   id, owner, token claims + raw token encrypted at rest (needed to mine as the user after a restart),
             state, turn, active_team, client_kind, client_tools, conversation, events (bounded), approvals,
             unmined_turns, timestamps
not persisted: chain (rebuilt from token + config on load), stream, ctx of a running turn
```

```
on service start:
  for session in sessions.where(turn.state in (running, awaiting_approval)):     # crashed mid-turn
      session.turn.state = idle
      session.events.append(turn_interrupted, turn_id, reason="service restart")  # client sees it on reconnect
      for a in session.approvals.pending(): a.answer(Deny("service restart"))
      session.unmined_turns.append(turn_id)                                         # the turn's completed part still gets mined
  sessions.save_all()
# sessions load lazily on first request; the chain is rebuilt then.
```

A client that reconnects after a restart replays, sees
`turn_interrupted`, and may resend its last message.

Multi-node: sessions are single-owner objects, so either route by
session id (sticky) or hold a per-session lease in the shared store and
proxy requests to the lease holder. Either way, one node runs a
session's turns at a time; the design needs no cross-node fan-out.

### 11g. Starting a new session with context

```
GET  /sessions?state=ended&limit=20                    -> the caller's own sessions, for a history picker

POST /sessions { active_team, client_kind, resume_from: "<old session id>" }
  -> old session must belong to the same subject and have a retained transcript
  -> new session; new chain from the *current* token; conversation seeded with the old
     transcript tail (config.sessions.resume_tail_turns) marked as prior context
  -> nothing else is carried: approvals, unmined turns, and active_team belong to the old session
```

Without `resume_from`, continuity is the memory layers alone, including
any session-summary memory written at the old session's end. That is
the normal path when switching from CLI to web.

### 11h. Timelines

Disconnect during an approval:

```
client                                         server
  POST /messages ─────────────────────▶        turn running
  ◀──── assistant_delta (seq 40..52)
  ◀──── tool_call_proposed (53)
  ◀──── approval_requested ap_91 (54)
  ✂ connection drops                           approval still pending; turn.state=awaiting_approval
  ...
  GET /events  Last-Event-ID: 52 ─────▶        replay 53, 54; then session_state{awaiting_approval, pending:[ap_91]}
  POST /approvals/ap_91 allow ────────▶        resume from step after asker; execute
  ◀──── approval_resolved (55), tool_call_started (56) ... turn_complete (61)
```

Token refresh that changes groups:

```
  POST /messages  (bearer: new token, groups now [team-a, team-b])
  ◀──── chain_rebuilt  layers=[global, team-a, team-b, user, session]
  ◀──── turn_started ...                        this turn already uses the new chain
```

Idle end with summary:

```
  (stream closed 30 min ago)                    sweeper: end_session(idle)
                                                mining job: 1 unmined turn -> memory_created (logged, no stream)
                                                summary -> on_memory_candidate -> personal (no active_team) -> memory_created
                                                session_ended (logged; delivered on a later replay if the client ever reconnects)
```

## 12. Tool execution and sandbox

Execution starts after `on_tool_call` returned `Continue` (§3c, §8).
From here nothing consults layers or rules; what runs, where, and with
what privileges was fixed when the tool entered `ctx.tools`.

### 12a. Capabilities and ceilings

```
Capability =
  fs_read(paths: [glob]) | fs_write(paths) | network(hosts: [glob]) | subprocess
  | env(vars: [name]) | secrets(names: [name]) | client(kind)      # client(...) only for execute_on=client

ToolDefinition.requires: [Capability]            # declared by the tool (skill author or tool source)
Skill.permissions:       [Capability]            # must cover every tool the skill ships
LayerCeiling[layer]:     [Capability]            # config: the most any tool from this layer may have
```

```yaml
tools:
  ceilings:                                      # applied when a tool is put into ctx.tools, and again at run
    global:   [fs_read: ["**"], fs_write: ["${workspace}/**"], network: ["*"], subprocess, env: ["PATH","HOME"], secrets: ["*"]]
    team:     [fs_read: ["${workspace}/**"], network: ["*.example.internal", "ci.example"], secrets: ["ci_token"]]
    user:     [fs_read: ["${user_root}/**"], fs_write: ["${user_root}/scratch/**"], network: ["*"]]   # user_root = users_root/<sub>
    session:  []                                 # client tools run on the client; nothing server-side
  sandbox:
    required_below_layer: user                   # script impls from global-skill/team layers are always sandboxed
    kind: subprocess_restricted                  # v1; adapter registry: subprocess_restricted | container | wasm
  runtimes: { python: /usr/bin/python3, node: /usr/bin/node, sh: /bin/sh }   # allowlist; a script names one
  limits:   { wall_s: 60, cpu_s: 30, mem_mb: 512, pids: 64, stdout_kb: 256, result_kb: 64 }
```

Ceilings are clipped at registration so the model never sees a tool it
cannot run:

```
ToolDefinitionsStep(layer).apply(ctx, defs):
  for d in defs:
      d.granted = intersect(d.requires, ceiling[layer])
      if d.granted != d.requires:
          if config.tools.on_excess == "hide": emit(tool_hidden, d.name, excess=...); continue
          d.degraded = True                                  # advertised with a note; run may fail on the missing capability
      ctx.put("tools", d.name, d, locked=d.locked)
# a skill's tools go through the same clip at the skill's layer when the skill loads (§7a re-labels forks)
```

### 12b. Dispatch

```
execute(call, ctx) -> ToolResult:
  d = ctx.tools.resolve(call.name)
  args = arg_validation.validate(d.input_schema, call.args)         # strict: reject; lenient: coerce + warn
  if args.error: return ToolResult.error(args.error, kind="invalid_args")

  base   = runners.for(d.impl)                                       # 12c
  runner = AuditedRunner(                                            # fixed wrap order, outermost first
             TimeoutRunner(limits.wall_s,
               RateLimitedRunner(per_tool=config.tools.rate.get(d.name), per_session=config.tools.rate.session,
                 ConcurrencyRunner(session_slots=config.tools.max_concurrent,
                   CapabilityCheckedRunner(d.granted,
                     SandboxedRunner(policy_for(d), base) if needs_sandbox(d) else base)))))

  emit(tool_call_started, call.id)
  handle = runner.start(d, args, ctx)                                # returns a handle so cancel can kill it
  ctx.running_tools[call.id] = handle
  result = await handle
  del ctx.running_tools[call.id]
  result = redact(result, ctx)                                       # 12f
  result = truncate(result, limits.result_kb, ctx)                   # 12g
  emit(tool_call_finished, call.id, ok=result.ok, duration=result.duration, truncated=result.truncated)
  return result

needs_sandbox(d):  d.impl is script and precedence(d.layer) < precedence(config.tools.sandbox.required_below_layer)
                   or d.impl is script and config.tools.sandbox.always
```

### 12c. Runners

```
BuiltinRunner.run(d, args, ctx):
  fn = builtins[d.impl.ref]
  host = HostApi(granted=d.granted, ctx=ctx)          # builtins never touch the OS directly; they call host.*
  return fn(args, host)

HostApi:                                              # every method checks `granted` before acting
  read_file(path)      -> requires fs_read matching path
  write_file(path, b)  -> requires fs_write
  fetch(url, ...)      -> requires network matching host
  run(cmd, ...)        -> requires subprocess; runs through the same SandboxedRunner as scripts
  secret(name)         -> requires secrets(name); value never returned to the model, only used by host.fetch headers
  memory_search(q)     -> the recall path of §10 with the session token
  load_skill(name)     -> §7b mutation of ctx.tools / ctx.prompt

HttpRunner.run(d, args, ctx):
  url = render(d.impl.url, args)
  assert host(url) allowed by d.granted.network
  headers = { **d.impl.headers, **auth_headers(d.impl.auth, secrets) }     # e.g. "Bearer ${secret:ci_token}"
  resp = http.request(d.impl.method, url, json=args, timeout=limits.wall_s, max_bytes=limits.stdout_kb*1024)
  return ToolResult.from_http(resp)

ScriptRunner.run(d, args, ctx):                       # always wrapped by SandboxedRunner when required
  runtime = config.tools.runtimes[d.impl.runtime] or fail("runtime not allowed")
  entry   = materialise(d.impl.entry, ctx.workspace)   # copy the skill resource into the call workspace
  proc = spawn([runtime, entry], stdin=json(args), env=env_for(d), cwd=ctx.workspace)
  return ToolResult.from_process(proc)                 # stdout is the result (json if parseable, else text); stderr captured

ClientRunner.run(d, args, ctx):                       # execute_on = client
  emit(tool_call_proposed, call, execute_on="client")  # already emitted with state=reviewing; now state=execute
  return await ctx.session.await_tool_result(call.id, timeout=limits.client_wall_s)   # POST /sessions/{id}/tool-results
```

### 12d. The sandbox (v1: restricted subprocess)

```
policy_for(d) -> SandboxPolicy:
  return SandboxPolicy(
    kind      = config.tools.sandbox.kind,
    uid       = config.tools.sandbox.uid,                        # unprivileged service account
    workspace = fresh_tmpdir(),                                  # per call; deleted after
    ro_mounts = [p for p in d.granted.fs_read.paths],            # bind read-only into the workspace view
    rw_mounts = [p for p in d.granted.fs_write.paths],
    network   = d.granted.network.hosts or NONE,                 # NONE -> no network namespace route at all
    env       = { v: os.environ[v] for v in d.granted.env.vars } | { "AGENT_TOOL": d.name },
    limits    = config.tools.limits)

SandboxedRunner.start(d, args, ctx):
  pol = self.policy
  cmd = self.inner.command(d, args, ctx)                          # e.g. [python3, entry.py]
  wrapped = sandbox_adapters[pol.kind].wrap(cmd, pol)
  # subprocess_restricted (Linux):
  #   unshare -n (no network) unless pol.network; if allowed hosts: route through a local egress proxy
  #     that enforces the allowlist (the process only ever sees the proxy)
  #   mount namespace: tmpfs workspace + ro/rw binds from pol; nothing else from the host filesystem
  #   setuid to pol.uid; prlimit cpu/mem/pids; seccomp profile "no ptrace, no mount, no raw sockets"
  #   stdin = args json; stdout/stderr captured with byte caps; wall timeout -> SIGKILL the whole cgroup
  handle = spawn(wrapped, stdin=json(args), cwd=pol.workspace, env=pol.env, limits=pol.limits)
  handle.on_finish(lambda: rm_rf(pol.workspace))
  return handle
```

Capability enforcement therefore happens in three places, and a tool
only needs to pass the one that applies to its impl:

| Impl      | Enforced by                                                    |
|-----------|----------------------------------------------------------------|
| builtin   | `HostApi` checks `granted` on every call                       |
| http      | `HttpRunner` checks the host against `granted.network`; secrets by name |
| script    | the sandbox: mounts, network namespace + egress proxy, env, limits |
| client    | the client; server treats results as untrusted (12h)           |

`container` and `wasm` are adapters with the same `wrap(cmd, policy)`
contract; a deployment picks one per layer if it wants stronger
isolation for team-layer scripts than for user-layer ones.

### 12e. Capability check at run time

```
CapabilityCheckedRunner.start(d, args, ctx):
  # static: the definition was clipped at registration; re-check in case the ceiling config changed since
  if not covers(ceiling[d.layer], d.granted): return ToolResult.error("tool exceeds layer ceiling", kind="capability")
  # dynamic: args that name paths or hosts must fall inside the grant
  for p in paths_in(args, d.input_schema):   if not d.granted.fs_read.matches(p) and not d.granted.fs_write.matches(p): return error(f"path not permitted: {p}", kind="capability")
  for h in hosts_in(args, d.input_schema):   if not d.granted.network.matches(h): return error(f"host not permitted: {h}", kind="capability")
  return self.inner.start(d, args, ctx)
```

A `capability` error is returned to the model as a normal tool error so
it can explain or choose another tool; it is also logged with the layer,
so a team learns its ceiling is too tight.

### 12f. Secrets and redaction

```
secrets: SecretsProvider                              # adapter: env | file | vault; keyed by name, allowlisted per layer via ceilings
auth_headers(auth, secrets):  "${secret:ci_token}" -> secrets.get("ci_token")   # resolved at run, in the runner, never in ctx

redact(result, ctx):
  for pattern in config.tools.redact_patterns + secrets.known_values():
      result.content = pattern.sub("[redacted]", result.content)
  return result
```

Secrets never enter `ctx`, the event log, the conversation, or memory
candidates (the locked global `no-credentials` mining rule is the second
line of defence, §6c).

### 12g. Results

```
ToolResult {
  ok: bool, kind?: invalid_args | capability | timeout | rate_limited | cancelled | tool_error
  content: text | json | [ContentPart]                 # ContentPart: text | json | asset_ref (binary saved aside, not inlined)
  truncated: bool, full_ref?: AssetRef                 # over result_kb: head kept inline, full output stored and referenced
  exit_code?, duration_ms, stderr_tail?
}

truncate(result, kb, ctx):
  if size(result.content) <= kb*1024: return result
  ref = assets.put(ctx.session, result.content, ttl=config.tools.result_ttl)
  result.content = head(result.content, kb*1024) + f"\n[truncated; full output: {ref}]"
  result.truncated = True; result.full_ref = ref
  return result
```

The model gets a bounded result every time. A builtin `read_asset(ref,
range)` lets it page through the rest if it needs to.

### 12h. Concurrency, limits, cancel, client results

```
ConcurrencyRunner: per-session slots (config.tools.max_concurrent, e.g. 4); excess calls queue in emit order
RateLimitedRunner: per-tool and per-session token buckets; over limit -> ToolResult.error(kind=rate_limited) immediately, model can retry later
TimeoutRunner:     wall clock; on expiry handle.kill() (sandbox: kill the cgroup) -> ToolResult.error(kind=timeout, stderr_tail)

cancel_turn(session):                                 # POST /sessions/{id}/cancel
  for h in ctx.running_tools.values(): h.kill()       # results become kind=cancelled; workspaces removed
  ...

# client-executed tools
POST /sessions/{id}/tool-results { tool_call_id, ok, content }
  -> must match a call awaiting a client result; else 409
  -> content is untrusted input: size-capped, redacted, and never treated as instructions
  -> client_wall_s timeout (longer than server tools; the user may be involved) -> kind=timeout
```

Clients declare what their tools can do at session creation
(`client_tools[].requires`), and the session ceiling for `client(...)`
capabilities is all a client tool can claim; server-side capabilities
are never granted to client tools.

### 12i. Worked example: `nightly-triage.fetch_log`

```
definition (from team-a skill, after clip at registration):
  impl:     script { runtime: python, entry: resources/fetch_log.py }
  requires: [network: ["ci.example"], secrets: ["ci_token"], fs_write: ["${workspace}/**"]]
  granted:  same (team ceiling allows ci.example and ci_token; workspace writes always allowed)
  layer:    team:team-a   risk: medium -> effective high (team +1)

on_tool_call: global high-risk require_approval -> user approved (§5) -> decided

execute:
  args ok (job, run: int)
  needs_sandbox: script from team layer, required_below_layer=user -> yes
  policy: uid=agent-sbx, workspace=/tmp/agt-7f2/, ro_mounts=[], rw_mounts=[workspace],
          network=["ci.example"] via egress proxy, env={AGENT_TOOL}, limits default
  runner chain: Audited(Timeout 60s(RateLimited(Concurrency(CapabilityChecked(Sandboxed(Script)))))
  spawn: unshare -n … python3 /tmp/agt-7f2/fetch_log.py   stdin={"job":"build-nightly-linux","run":1234}
     script reads CI_TOKEN? no: secrets are not env; it calls http://proxy/ with header injected by the proxy for ci.example
     (secrets(ci_token) grant = the egress proxy attaches the token to requests to ci.example; the script never sees it)
  stdout: 2.1 MB of log
  redact: none matched; truncate: 64 KB kept, full stored as asset a_91
  result: ok, truncated=true, full_ref=a_91, duration=3.8s
  workspace removed

events:
  tool_call_started        nightly-triage.fetch_log
  tool_call_finished       nightly-triage.fetch_log  ok  3.8s  truncated -> a_91
```

## 13. Long-running tools: background jobs, progress, and notification

A foreground tool call blocks the agent loop and lives inside one turn
(§12). A **background job** returns immediately with a handle, keeps
running after the turn ends, reports progress on the stream, and
delivers its result back through the pipeline when it finishes.

### 13a. Declaring or promoting

```
ToolDefinition.mode: foreground | background | auto     # default foreground
  background: always returns a job handle
  auto:       runs foreground; if still running after config.jobs.promote_after_s (e.g. 10 s), promoted to a job

Job {
  id, owner (sub), origin: { session_id, turn_id, tool_call_id }, tool: name, args_hash
  state:     queued | running | finished | failed | cancelled | timed_out
  progress:  { pct?, message?, updated_at }
  result?:   ToolResult                                  # redacted + truncated exactly as foreground
  delivery:  { mode: agent_turn | notify, delivered: bool, delivered_to_session?: id }
  limits:    { wall_s: config.jobs.wall_s (e.g. 3600), ... }
  on_session_end: continue | cancel                      # from the tool definition; default continue
  created_at, started_at, finished_at
}
JobStore { save, load, list(owner, state?), where(...) }   # sibling of SessionStore, same adapter family
```

### 13b. Starting a job

`execute` (§12b) gains one branch after the runner chain is built:

```
execute(call, ctx):
  ...
  handle = runner.start(d, args, ctx)
  if d.mode == background or (d.mode == auto and not handle.done_within(config.jobs.promote_after_s)):
      job = Job.new(owner=ctx.token.subject, origin=(session, turn, call.id), tool=d.name,
                    handle=handle, delivery=delivery_mode_for(ctx.session, d), on_session_end=d.on_session_end)
      jobs.save(job); job_runner.adopt(job, handle)        # moves the handle out of the turn: its own pool, limits, sandbox kept alive
      del ctx.running_tools[call.id]
      emit(tool_call_finished, call.id, ok=True, background=True, job_id=job.id)
      emit(tool_job_started, job)
      return ToolResult.ok({ "job_id": job.id, "state": "running", "eta_s": handle.eta(),
                             "note": "Running in the background. Use job_status / job_wait, or the result will be delivered when done." })
  result = await handle
  ...
```

The model sees a normal tool result and finishes the turn ("I've
started the deploy; I'll report back when it completes").

### 13c. Progress protocol

One event shape, four producers:

```
tool_progress { job_id, pct?, message?, updated_at }

builtin:  host.progress(pct, message)                       # HostApi method; rate-limited to 1/s
script:   a stdout line  {"progress": {"pct": 42, "message": "uploading"}}   # JSON-lines; non-progress lines are result output
http:     impl.status_url polled every config.jobs.poll_s; response mapped by impl.status_map (pct, message, done, result)
client:   POST /sessions/{id}/jobs/{job_id}/progress { pct, message }         # client-executed background tools

JobRunner.on_progress(job, p):
  job.progress = p; jobs.save(job)                            # durable; survives reconnect/restart
  session = sessions.load(job.origin.session_id)
  if session: emit(session, tool_progress, job.id, p)         # only if the origin session still exists; else visible via GET /jobs
```

Model-side builtins (global layer, risk low, locked allow):

```
job_status(job_id)                -> { state, progress, started_at, eta_s }
job_wait(job_id, timeout_s<=60)   -> blocks the turn up to timeout; returns result if finished, else status
job_cancel(job_id)                -> owner check; kills the handle; state=cancelled
```

`job_wait` lets the model choose to block briefly ("let me wait a
moment for that") without the deployment having to guess.

### 13d. Completion re-enters the pipeline

```
JobRunner.on_finish(job, result):
  result = redact(result); result = truncate(result)          # §12f, §12g
  job.result = result; job.state = finished|failed|timed_out; job.finished_at = now(); jobs.save(job)
  deliver(job)

deliver(job):
  session = sessions.load(job.origin.session_id)
  if not session or session.state == ended: session = sessions.latest_active_for(job.owner)   # owner's current session, if any
  if not session:
      notifier.send(job.owner, job)                            # adapter: none | webhook | email | chat; framework has no client to push to
      return                                                   # stays undelivered; picked up in 13e
  match job.delivery.mode:
    notify:
        emit(session, tool_job_finished, job.id, summary=result.summary, state=job.state)
        session.pending_job_results.append(job.id)             # injected on the user's next message (13e)
    agent_turn:
        emit(session, tool_job_finished, job.id, ...)
        run_agent_initiated_turn(session, JobResultMessage(job))
  job.delivery.delivered = True; job.delivery.delivered_to_session = session.id; jobs.save(job)
```

An agent-initiated turn is the ordinary turn loop with a different
inbound:

```
run_agent_initiated_turn(session, inbound: JobResultMessage):
  if session.turn.state != idle: session.pending_job_results.append(inbound.job_id); return   # never interrupt a running turn
  if not session.token.valid(): session.pending_job_results.append(inbound.job_id); return    # cannot run as the user now
  ctx = TurnContext(token=session.token, session=session, inbound=inbound)                    # inbound.kind = job_result
  emit(turn_started, initiated_by="job", job_id=inbound.job_id)
  run_hook("on_message", chain, ctx)               # gates still apply (a session gate may say "no unsolicited turns"; then fall back to notify)
  session.conversation.append(job_result_message(job))                                        # a USER-role message (below), never a tool_result
  ... model call, on_tool_call, on_turn_end, mining as usual ...

job_result_message(job) -> Message:
  # The original tool call was already answered with "job started" (13b), and the job may be delivered
  # into a different session. Providers reject a tool_result with no matching preceding tool_use, so a
  # late result is a user-role message with a marker block, valid in any append-only history:
  return Message(role=user, content=[
      text(f"[background job {job.id} finished: tool {job.tool}, started in turn {job.origin.turn_id}"
           f"{'' if same_session else ' of an earlier session'}; state={job.state}]"),
      text(job.result.content)  or  document(job.result.full_ref) ])
```

The model then streams an unsolicited message: "The deploy finished
with 2 warnings: …". The marker names the original call and turn, so
the conversation stays coherent, and the miner can turn the outcome
into a memory (§6). If the model calls `job_status` on a job it sees a
marker for, it gets the same result again.

### 13e. Undelivered results and the next message

```
on user message (POST /messages):
  for job_id in session.pending_job_results + jobs.undelivered_for(session.owner):
      job = jobs.load(job_id)
      session.conversation.append(job_result_message(job))                                      # user-role marker message (13d)
      job.delivery.delivered = True; jobs.save(job)
  ... normal turn ...

GET /jobs?state=finished&delivered=false      -> the owner's undelivered results (any session)
GET /jobs/{id}                                -> state, progress, result (if finished), origin
POST /jobs/{id}/cancel
```

A job that finished while the user was away is therefore never lost:
it is on the stream if a session is open, in the next turn's context
otherwise, and always in the jobs list.

### 13f. Session end and restart

```
end_session(session, reason):                  # §11e gains one step before draining
  for job in jobs.where(origin.session_id=session.id, state in (queued, running)):
      if job.on_session_end == cancel or reason == "client_delete": job_runner.cancel(job)
      # else: continues; owner-scoped; delivered per 13d

on service start:                              # §11f gains
  for job in jobs.where(state=running):
      if job_runner.can_reattach(job): job_runner.reattach(job)        # e.g. http status polling, or a container that outlived the process
      else: job.state = failed; job.result = error(kind="interrupted", "service restart"); deliver(job)
```

Subprocess sandboxes do not survive a service restart; container and
http-polled jobs can. A tool that must survive restarts should be
`http` with a status URL.

### 13g. Limits

```
jobs:
  promote_after_s: 10
  wall_s: 3600                     # per job; tool may lower, never raise
  max_per_user: 5
  max_total: 50
  poll_s: 15                       # http status polling
  result_ttl: 7d                   # undelivered results and stored outputs
  delivery_default: agent_turn     # or notify; a client may override per session (client_kind cli -> notify?)
  notifier: { adapter: none }      # webhook | email | chat for owners with no open session
```

Concurrency is separate from foreground slots (§12h). Background jobs
run with the sandbox policy computed at start; the wall clock is the
job's, not the turn's.

### 13h. Worked example

The user asks the model to run the full regression suite (a team skill
tool `regression.run`, `mode: background`, `on_session_end: continue`).

```
turn 1
  tool_call_proposed       regression.run          state=reviewing
  approval_requested       (risk high, team +1)    -> user allows
  tool_call_started        regression.run
  tool_call_finished       regression.run  ok  background=true  job_id=j_31
  tool_job_started         j_31  eta_s=1500
  assistant_delta ...      "Started the regression suite (about 25 minutes). I'll report back."
  turn_complete

  tool_progress            j_31  12%  "unit: 412/3400"
  tool_progress            j_31  40%  "integration: 8/60"
  (user: "how's it going?")
turn 2
  tool_call_proposed       job_status(j_31)        (locked allow; no audit)
  tool_call_finished       job_status  {running, 40%, eta 900s}
  assistant_delta ...      "About 40% through, integration tests running. Roughly 15 minutes left."
  turn_complete

  (CLI closed; session idle; job continues)
  tool_progress            j_31  100%             (logged; no stream)
  on_finish -> deliver: origin session idle, no other active session -> notifier(none) -> undelivered

  (user opens the web client next morning; new session)
  POST /messages "morning"
    -> undelivered j_31 injected as the late result of regression.run
  assistant_delta ...      "Morning. The regression suite you started last night finished: 3 failures, all in upload/…"
  turn_complete
  memory_created           team:team-a  "Regression run 2026-09-14: 3 failures in upload/ (test_upload_retry flaky)"
```

Had the CLI stayed open, the finish would have produced an
agent-initiated turn with the same summary at the moment the job
completed.

## 14. Client tools and the session layer

The session layer is the last entry in every outer chain. Unlike the
other layers it is not backed by stores: its inner chains are built
from **session state** the client supplied. That is how a CLI exposes
"open in editor" without a skill, how a web client says "we're
mid-incident", and how a client adds its own safety gate.

### 14a. Building the session layer

```
SessionLayer.build(session) -> Layer:
  return Layer(name="session", hooks={
    on_message: [
      SessionInstructionsStep(session.instructions),        # prompt fragments, section=session, unlocked
      ClientToolsStep(session.client_tools),                # tool definitions, execute_on=client
      SessionRulesStep(session.rules),                      # client gates (14c) + remembered approvals (§5e)
      SessionScratchStep(session.scratch),                  # ephemeral memories for this session only
    ],
    on_tool_call:        [ StaticRulesStep("session") ],    # evaluates session.rules last; cannot lock; cannot shield
    on_memory_candidate: [ ScratchAcceptStep(session) ],    # accepts kind=scratch only; never persisted
    on_turn_end:         [],
  })
# rebuilt whenever session.client_tools / instructions / rules change (cheap: no I/O)
```

Everything the session layer contributes is unlocked and applies
*after* global, team, and user, so it can override unlocked values (a
CLI may shadow an unlocked `read_file` with its local one) but never a
locked one, and it can add restrictions but never remove another
layer's.

### 14b. Registering client tools

```
POST /sessions                          { client_kind: "cli", client_tools: [...], instructions?, gates?, active_team? }
POST /sessions/{id}/tools               [ ClientToolDeclaration ]        # replaces the set; empty list clears
DELETE /sessions/{id}/tools/{name}

ClientToolDeclaration {
  name, description, input_schema
  risk: none | low | medium | high            # floor applied below
  requires: [client(kind)]                    # only client(...) capabilities are accepted
  mode: foreground | background               # background client tools report via /jobs/{id}/progress (§13c)
  timeout_s?: <= config.tools.client_wall_s
}

ApiLayer.register_client_tools(session, decls):
  defs = []
  for c in decls:
      if any(cap.kind != "client" for cap in c.requires): return 400 "client tools may only declare client(...) capabilities"
      if c.name in ctx_tools_locked_names(session): return 409 { error: "locked", name: c.name }   # early, clearer than a silent shadow_refused
      d = ToolDefinition(name=c.name, description=c.description, input_schema=c.input_schema,
                         risk=max(c.risk, config.tools.client_risk_floor),       # e.g. low
                         execute_on=client, mode=c.mode, layer=session,
                         impl=ClientImpl(timeout_s=c.timeout_s), requires=c.requires, granted=c.requires)
      defs.append(d)
  session.client_tools = defs; sessions.save(session); session.layer = SessionLayer.build(session)
  emit(session, client_tools_registered, [d.name for d in defs])
  return 200 { registered: [...], shadows: [(d.name, lower_layer) for d in defs if d.name in lower_unlocked_names] }
```

At the next `on_message`:

```
ClientToolsStep.apply(ctx):
  for d in session.client_tools:
      ok = ctx.put("tools", d.name, d, locked=False)
      if ok and d.shadows: emit(tool_shadowed, d.name, shadows=d.shadows, by="session")
```

Trust limits, all fixed by the service, none negotiable by the client:

- `execute_on` is forced to `client`; the server never runs client code.
- Only `client(...)` capabilities; no `fs_*`, `network`, `secrets`, `subprocess` server-side grants ever attach to a client tool.
- Risk floor plus the session-layer escalation (§8c, e.g. +1), so a client tool declared `none` is audited as at least `low`, and `medium` becomes `high` and needs approval unless a locked global allow says otherwise.
- Results are untrusted input (§12h): capped, redacted, never instructions.

### 14c. Client gates (declarative, no code)

A client cannot register steps, only **rules**. They use the same
`ClassifierRule` shape and the same fixed `match` fields as §8a, plus a
few session-only fields that the server can evaluate:

```
POST /sessions/{id}/gates   [ ClientGate ]                 # replaces the set

ClientGate = ClassifierRule with:
  role:   tool_audit
  kind:   deny | require_approval                          # restrictive only; `allow` is accepted but unlocked, so it only pre-empts the model audit
  match:  §8a fields  +  { path_args_within: "<dir>", path_args_not_within: "<dir>", host_args_in: [...] }
  locked: false (forced)

# CLI example: nothing may touch files outside the working directory, whatever tool it is
{ id: "cli-cwd-only", kind: deny, match: { path_args_not_within: "/home/me/proj" }, reason: "outside the working directory" }
# web example: ask before any tool that can send data out
{ id: "web-egress-ask", kind: require_approval, match: { effective_risk_gte: medium, tool_glob: "*.send_*" } }
```

Remembered approvals (§5e, `remember: session`) are appended to the
same `session.rules` list as unlocked `allow` rules and evaluated here.

Because the session `StaticRulesStep` runs last, a client gate can only
tighten. It never sees a call a global rule already denied, and it
cannot shield a call from an earlier layer's `require_approval`.

### 14d. Session instructions and scratch memories

```
PATCH /sessions/{id}   { instructions: "we're mid-incident; prefer read-only tools", active_team: "team-a" }

SessionInstructionsStep.apply(ctx):
  ctx.put("prompt", "session-instructions", PromptFragment(section=session, body=session.instructions, priority=0), locked=False)

# scratch: memories that must not outlive the session (e.g. "current PR is #4412")
POST /memories { kind: scratch, body: "current PR is #4412" }     # audience is implicitly session
SessionScratchStep.fetch/apply -> ctx.memories += session.scratch  (rank fused like any other layer; layer=session)
ScratchAcceptStep.run(ctx, c):  if c.kind == scratch: session.scratch.append(c); return Stop(Accepted)  else Continue
# scratch is dropped at end_session; the miner may still propose a durable version of it as a normal candidate
```

### 14e. Executing a client tool

`ClientRunner` from §12c, with the disconnect cases spelled out:

```
ClientRunner.start(d, args, ctx) -> handle:
  call = ctx.current_call
  session.pending_client_calls[call.id] = PendingClientCall(call, d, deadline=now()+d.impl.timeout_s)
  emit(session, tool_call_proposed, call, execute_on="client", state="execute", args=args)   # the instruction to run it
  return handle_awaiting(session.tool_result_future(call.id), deadline)

POST /sessions/{id}/tool-results   { tool_call_id, ok, content, error? }
ApiLayer.tool_result(request):
  token, session = ...; owner check
  p = session.pending_client_calls.get(request.tool_call_id) or 409 "no such pending client call"
  r = ToolResult(ok=request.ok, content=cap(request.content, config.tools.client_result_kb), kind=request.error?.kind)
  r = redact(r, ctx); r.untrusted = True
  session.tool_result_future(p.call.id).set(r); del session.pending_client_calls[p.call.id]
  return 200

on deadline:   future.set(ToolResult.error(kind=timeout, "client did not return a result"))  # model can continue or re-ask
on disconnect: nothing; the pending call stays. On reconnect, session_state.pending_client_calls lists it with args and
               remaining time so the client can execute it now (or the replay of tool_call_proposed does).
on cancel:     pending client calls are dropped; a late tool-result POST returns 409
```

A background client tool (`mode: background`) returns a job (§13) whose
progress and result arrive via `/jobs/{id}/progress` and `/tool-results`
respectively; the same untrusted-input rules apply.

### 14f. What clients typically register

| Client kind | Tools (all `execute_on: client`)                              | Gates                                  |
|-------------|----------------------------------------------------------------|----------------------------------------|
| CLI         | `local.read_file`, `local.write_file`, `local.run` (with its own confirm UI), `open_in_editor` | deny paths outside cwd     |
| Web         | `clipboard.copy`, `download` (client saves an asset), `open_url`| ask before medium+ egress              |
| Desktop app | `notify`, `pick_file`, `open_in_app`                            | none, or ask on write                  |

The framework does not care which; it only enforces the limits in 14b.
Two sessions of the same user (CLI and web open at once) have separate
session layers: a tool registered in one is invisible in the other.

### 14g. Worked example: CLI session

```
POST /sessions {
  client_kind: "cli", active_team: "team-a",
  instructions: "working in /home/me/proj on the upload service",
  client_tools: [
    { name: "local.read_file",  risk: low,    requires: [client(fs)],     input_schema: {path} },
    { name: "local.write_file", risk: medium, requires: [client(fs)],     input_schema: {path, content} },
    { name: "open_in_editor",   risk: low,    requires: [client(editor)], input_schema: {path, line?} } ],
  gates: [ { id: "cli-cwd-only", kind: deny, match: { path_args_not_within: "/home/me/proj" } } ] }
-> 201; no shadows (global has read_file, not local.read_file)

turn: "open the retry logic in the editor and show me the config it reads"
  on_message: session layer puts session-instructions fragment, 3 client tools (escalated: low->medium, medium->high), cli-cwd-only rule

  model -> open_in_editor(path="src/upload/retry.py")
    on_tool_call: RiskEscalation session +1 -> medium; ScopeGate ok; global: no match; team-a: no match; user: no match;
                  session: cli-cwd-only: path within cwd -> no match; ModelAudit p=.96 -> allow
    ClientRunner: tool_call_proposed execute_on=client state=execute
    CLI opens the file, POST tool-results {ok:true, content:"opened"}
    tool_call_finished  open_in_editor ok

  model -> local.read_file(path="/etc/upload/config.yaml")
    on_tool_call: ... session: cli-cwd-only: /etc/upload/config.yaml not within /home/me/proj -> deny
    tool_call_denied  local.read_file  "outside the working directory"
    model -> read_file(path="/etc/upload/config.yaml")          # the global builtin, server-side
    on_tool_call: global read-only locked allow -> shielded; session gate skipped (shielded) -> runs on the server
    (the CLI gate constrained *its* tools; the server's own read_file is governed by global's ceiling and rules)

  assistant_delta ... "Opened retry.py. The config it reads is /etc/upload/config.yaml: …"
```

The last step is worth noticing: a client gate governs what the client
is willing to do, not what the server may do. Restricting the server's
own tools is a user-layer or global rule, not a session gate. If the
user wants the CLI session to be cwd-only for *every* tool, the gate
should not name paths at all: `match: { path_args_not_within: cwd }`
applies to server tools too, but global's locked allow on `read_file`
still wins by design; the correct lever there is the user's own rule
file or global unlocking `read_file`.

### 14h. Events

```
client_tools_registered  [local.read_file, local.write_file, open_in_editor]
tool_shadowed            (only if a client tool took an unlocked lower-layer name)
tool_call_proposed       open_in_editor  execute_on=client  state=execute  args={...}
tool_call_finished       open_in_editor  ok
tool_call_denied         local.read_file  by=session:cli-cwd-only
session_state            ... pending_client_calls=[{id, name, args, deadline}]   (on every stream open)
```

## 15. Skill discovery and loading

Two phases keep the context small: **discovery** puts one-line summaries
of every relevant skill into the prompt's skill index; **loading** pulls
one skill's full instructions, tools, and prompt fragments into the
context. Loading is an ordinary tool call, so it passes the audit chain
like anything else.

### 15a. Skill stores

```
# directory / git layout (one skill per folder)
skills/
  nightly-triage/
    SKILL.md            # front matter + instructions
    resources/          # files tools or the model may read (flaky.txt, templates)
    tools/fetch_log.py  # script impls referenced from front matter

# SKILL.md
---
name: nightly-triage
description: Triage a failed nightly CI run and summarise failures by test file.
triggers: [nightly, "build-nightly-*", flaky]
tools:
  - name: fetch_log
    description: Fetch the log for a nightly run
    input_schema: { job: string, run: integer }
    risk: medium
    impl: { script: { runtime: python, entry: tools/fetch_log.py } }
    requires: [network: ["ci.example"], secrets: ["ci_token"]]
prompt_fragments:
  - { section: project, body: "Nightly triage: the job is build-nightly-linux; known-flaky tests are in resources/flaky.txt." }
permissions: [network: ["ci.example"], secrets: ["ci_token"]]
auto: false        # true: load automatically when a trigger matches (15e)
pinned: false      # true: always listed in the index regardless of relevance
locked: false
---
1. Call fetch_log for the failed run.
2. Group failures by test file; mark tests listed in resources/flaky.txt as flaky.
3. ...

SkillStore.discover(query: DiscoverQuery, token) -> [SkillSummary]      # cheap; store may keep an index of description+triggers
SkillStore.load(id) -> Skill                                            # full body; version = content hash or commit
SkillStore.resource(id, path) -> bytes                                  # size-capped
SkillStore.load_version(id, version) -> Skill?                          # git can; http may not (§7e)

GitSkillStore: clone at first use, `fetch` on refresh (TTL or /sources/refresh); version = tree hash of the skill folder
HttpSkillStore: GET /skills?q=  and  GET /skills/{id}; ETag as version
DirectorySkillStore: read on each fetch (cheap); version = content hash
```

### 15b. Discovery during `on_message`

```
DiscoverQuery { text: ctx.recall_query.texts[0], hints: {active_team, loaded: [...]}, k: config.skills.k_per_layer }

SkillDiscoveryStep(layer).fetch(ctx):
  if not ctx.token.allows(layer, "use", "skill"): return []
  q = ctx.discover_query
  found  = store.discover(q, ctx.token)                          # relevance-ranked by the store (keyword or embedding over description+triggers)
  pinned = store.discover(DiscoverQuery.pinned(), ctx.token)     # front-matter pinned: always
  loaded = [store.summary(id) for id in session.loaded_skills if id.layer == layer]   # sticky loads (15f) always listed
  return dedupe(found + pinned + loaded)

SkillDiscoveryStep(layer).apply(ctx, summaries):
  for s in summaries:
      s.layer = layer
      s.suggested = triggers_match(s.triggers, ctx.inbound.text)  # cheap glob/keyword; no model
      staleness_check(ctx, s)                                     # §7d, for forks
      ctx.put("skills", s.name, s, locked=s.locked)               # shadowing / locking as everywhere
```

What lands in `ctx.skills` for the example (message mentions "nightly"):

```
code-review     (user fork of global; stale=false)                       tools: [code-review.lint]
nightly-triage  (team-a; suggested=true: trigger "nightly")              tools: [nightly-triage.fetch_log]
release-notes   (global; locked; pinned)                                 tools: []
deploy-checklist(team-a; relevance .31)                                  tools: [deploy-checklist.verify]
```

### 15c. What the model sees

The assembler renders `ctx.skills` as the `__skills` derived fragment
(§9c), suggested and loaded first, and the global layer provides the
`load_skill` builtin:

```
<skills>
Load a skill with load_skill(name) to get its full instructions and tools.
- nightly-triage (team-a) ▲suggested: Triage a failed nightly CI run… tools: fetch_log
- code-review (personal, overrides global): Review a diff against team conventions. tools: lint
- release-notes (global): Draft release notes from merged PRs.
- deploy-checklist (team-a): Verify a deploy against the checklist. tools: verify
</skills>
```

Under budget pressure the fragment shrinks to names only, keeping
suggested and loaded entries verbose.

### 15d. Loading is a tool call

```
load_skill    builtin, global layer, risk: low, requires: []            # global rule: UNLOCKED allow -> no model audit by default,
                                                                        # but any later layer may still add deny / require_approval (§8b)
unload_skill  builtin, global layer, risk: none

on_tool_call for load_skill("nightly-triage"):
  static rules apply as for any call. Because the global allow is unlocked, a team or user rule such as
    { kind: require_approval, match: { tool: load_skill, args_match: { name: "deploy-*" } } }
  still fires (restrictions accumulate; only a *locked* allow would shield). Skill loading is therefore
  auditable by rule; the model audit is skipped unless a deployment removes the global allow.

execute (BuiltinRunner -> HostApi.load_skill):
HostApi.load_skill(name) -> ToolResult:
  summary = ctx.skills.get(name) or return error("unknown skill; see the skill index")
  if len(session.loaded_skills) >= config.skills.max_loaded: evict_lru(session)                 # emits skill_unloaded
  skill = layers.get(summary.layer, ctx.token).skill_store.load(summary.id)
  skill = clip_tools_to_ceiling(skill, summary.layer)                                             # §12a; on_excess hide|degrade

  # 1. tools, at the skill's layer, namespaced
  for t in skill.tools:
      t.name = f"{skill.name}.{t.name}"; t.layer = summary.layer
      ctx.put("tools", t.name, t, locked=False)
  # 2. prompt fragments, at the skill's layer
  for f in skill.prompt_fragments:
      f.id = f.id or f"{skill.name}.{f.section}"; f.layer = summary.layer
      ctx.put("prompt", f.id, f, locked=False)
  # 3. instructions: returned now as the tool result, and kept live as a shrinkable fragment for later turns
  ctx.put("prompt", f"{skill.name}.instructions",
          PromptFragment(section=skills, layer=summary.layer, priority=-10, optional=True, shrinkable=True,
                         body=skill.instructions, shrunk_body=f"[loaded skill {skill.name} v{skill.version[:7]}; call load_skill again for full instructions]"),
          locked=False)
  ctx.prompt_dirty = True
  session.loaded_skills[skill.id] = LoadedSkill(id, layer, version=skill.version, loaded_at=now(), last_used=now())
  emit(skill_loaded, skill.id, layer, version, tools=[t.name for t in skill.tools], fragments=[...])
  return ToolResult.ok(content=skill.instructions, meta={ tools: [...], resources: skill.resource_paths })
```

The next model call re-derives the tool list and system prompt (§9g),
so `nightly-triage.fetch_log` is callable immediately.

### 15e. Auto-load by trigger

```
SkillAutoLoadStep(layer).run(ctx):                 # on_message, after SkillDiscoveryStep, contributing
  for s in ctx.skills.values() if s.layer == layer and s.auto and s.suggested and s.id not in session.loaded_skills:
      if not auto_load_allowed(ctx, s): continue   # a restrictive rule may forbid auto-load: { kind: deny, match: { tool: load_skill, args_match: {name: s.name}, auto: true } }
      HostApi.load_skill(s.name, auto=True)        # same path as 15d; emits skill_loaded auto=true
```

Auto-load bypasses the model's choice but not policy: the same
`load_skill` rules apply, with `auto: true` matchable so a user can say
"never auto-load, always let the model decide".

### 15f. Stickiness, unloading, eviction

```
skills:
  sticky: session          # session | turn. session: loaded skills stay loaded until unloaded/evicted/session end
  max_loaded: 6
  k_per_layer: 5

next turn (sticky=session):
  SkillDiscoveryStep lists loaded skills regardless of relevance (15b)
  on_message re-puts their tools and fragments from session.loaded_skills without reloading the store:
    LoadedSkillsStep(layer).apply(ctx): for ls in session.loaded_skills at this layer: re-put tools/fragments cached on ls
  last_used bumps when any of the skill's tools is called or its name is cited

unload_skill(name)  -> removes tools + fragments from ctx, drops from session.loaded_skills, emits skill_unloaded
evict_lru(session)  -> unload the least recently used non-pinned loaded skill
end_session         -> everything unloaded (nothing to do; session state goes away)
```

### 15g. Resources

```
skill_resource(skill, path, range?)   builtin, risk: low; requires: []    # reads from the skill's own store, not the host fs
  -> store.resource(id, path), size-capped (config.skills.resource_kb), redacted, returned as text or asset ref
Script tools of the skill get resources/ materialised read-only into their sandbox workspace (§12c), so
  tools/fetch_log.py can open ../resources/flaky.txt without any fs_read grant on the host.
```

### 15h. Version pinning within a session

```
session.loaded_skills[id].version is pinned: a `fetch` on the git store mid-session does not change loaded
instructions or tools until the model or user reloads.
on source refresh:
  for ls in session.loaded_skills: if store.version(ls.id) != ls.version: emit(skill_outdated, ls.id, ls.version, now=...)
load_skill(name) on an already-loaded, outdated skill -> reloads at the new version (tools re-put; old tool names removed)
```

### 15i. Discovery at scale

A global repo with hundreds of skills must not be listed wholesale:

- The store keeps a small index over `name + description + triggers`
  (keyword by default; embedding if the store has an embedder). Rebuilt
  on refresh.
- `k_per_layer` caps how many summaries a layer contributes; pinned and
  loaded skills are always included on top.
- The index fragment is shrinkable, so budget pressure trims descriptions
  before dropping entries.
- `GET /skills?q=` exposes the same discovery to clients for a browser
  or a "find me a skill for X" command.

### 15j. Worked example

```
turn 1  "the nightly run is red again"
  context_ready        skills: 4 listed (1 suggested), 0 loaded
  model -> load_skill("nightly-triage")
    on_tool_call: global unlocked allow for load_skill -> decided; no later restriction matches -> no model audit
    HostApi.load_skill: team-a store.load -> v 7e02…; 1 tool clipped ok; 1 fragment; instructions fragment
  skill_loaded         nightly-triage  team-a  v7e02  tools=[nightly-triage.fetch_log]
  (model reads instructions from the tool result; next call has the new tool)
  model -> nightly-triage.fetch_log(job=..., run=1234)  -> §12i
  ...

turn 2  "what about yesterday's run?"
  context_ready        skills: 4 listed, 1 loaded (nightly-triage kept: sticky=session)
  <skills> shows nightly-triage as loaded; <skills> section also carries the shrunk instructions fragment
  model -> nightly-triage.fetch_log(run=1233)          # no reload needed

later   maintainers push a new version of nightly-triage; /sources/refresh
  skill_outdated       nightly-triage  v7e02 -> v91aa
turn 3  model (or user via client) -> load_skill("nightly-triage") -> reloaded at v91aa; tools re-put
```

### 15k. API and events

```
GET  /skills?q=&layers=&k=                 discovery across permitted layers (summaries; suggested/pinned flags need no session)
GET  /skills/{id}                          full skill (instructions, tools, fragments, version, derived_from)
GET  /skills/{id}/resources/{path}         a resource file (size-capped)
POST /sessions/{id}/skills/{name}/load     client-initiated load (same path as load_skill; audited as a tool call)
DELETE /sessions/{id}/skills/{name}        unload
GET  /sessions/{id}/skills                 loaded skills with versions and outdated flags

events: skill_suggested (debug), skill_loaded {auto?}, skill_unloaded {reason: model|client|evicted}, skill_outdated, skill_shadowed, shadow_refused
```

## 16. Classifier engine and model client adapters

Two ports sit at the bottom of everything: `ModelClient` (one provider
behind one interface) and `ClassifierEngine` (the support agent's brain,
usually a `ModelClient` with a different model and a structured-output
prompt). Neither knows about layers, sessions, or the chain.

### 16a. The `ModelClient` port

```
ModelClient {
  complete(req: ModelRequest) -> Stream<ModelEvent>         # always streaming; adapter may buffer for non-streaming providers
  count_tokens(req) -> int
  capabilities() -> { context_window, max_output, structured_output, tool_use, parallel_tools, caching, thinking, forced_tool_choice }
}

ModelRequest {
  model: text                                # from config; agent and classifier may differ
  system: [SystemBlock]                      # ordered; each block has `stable: bool` so the adapter can place cache breakpoints
  messages: [Message]                        # append-only conversation; assistant content stored verbatim (16c)
  tools: [ToolSpec]                          # { name, description, input_schema, strict? }
  max_output: int
  effort?: low | medium | high | xhigh | max
  structured?: JSONSchema                    # force a JSON reply matching this schema
  parallel_tools: bool                       # default true
  deadline?: duration                        # adapter enforces; used by the classifier
  metadata: { session_id, turn_id, purpose: agent | audit | mine | summarise }
}

Message  = { role: user | assistant, content: [Block] }
Block    = text{text} | tool_call{id, name, args} | tool_result{tool_call_id, content, is_error}
         | image{...} | document{...} | provider_opaque{provider, model, payload}   # e.g. thinking blocks; replayed untouched
ModelEvent =
    message_start{message_id, input_usage}
  | text_delta{text}
  | reasoning_delta{text}?                   # only if the provider exposes a summary; never required
  | tool_call_start{id, name}
  | tool_call_args_delta{id, partial_json}   # accumulate; parse only at tool_call_end
  | tool_call_end{id, args}                  # args parsed here (json.loads), never string-matched
  | message_end{stop_reason, usage}          # stop_reason: end_turn | tool_use | max_tokens | refusal{category?} | other
  | error{kind: retryable | fatal, status?, retry_after?}
```

### 16b. Rendering a request from the turn context

```
render(ctx) -> ModelRequest:
  prompt = ctx.assembled                                       # §9: sections in stable-first order
  system = [ SystemBlock(prompt.stable_text,   stable=True),   # identity, policy, project, skills index (shared across users)
             SystemBlock(prompt.volatile_text, stable=False) ] # user_prefs, memories, session
  tools  = [ToolSpec(d.name, d.description, d.input_schema, strict=schema_is_strict_safe(d.input_schema))
            for d in ctx.tools.values() in sorted-by-name order]   # deterministic order: a reordered tool list is a cache miss
  return ModelRequest(model=config.model.agent, system=system, messages=ctx.session.conversation.messages,
                      tools=tools, max_output=config.model.max_output, effort=config.model.effort,
                      parallel_tools=True, metadata={..., purpose: agent})
```

Conversation rules the core keeps so any adapter can cache and replay:

- **Append-only.** The core never edits or deletes earlier messages. Per-turn
  reminders go into the volatile system block, not into history.
- **Assistant content is stored verbatim** as blocks, including any
  `provider_opaque` blocks the adapter returned. They are replayed
  unchanged when the same model is used and are dropped by the adapter
  when the model changes.
- **All tool results of one assistant turn go back in one user message**,
  in the order the calls were made, failures as `is_error: true`. Never
  split them across messages and never drop a failed one.
- **Context-window overflow** is checked by the core before every model
  call and handled without rewriting history:

  ```
  before model.complete(req):
    need = model.count_tokens(req) + req.max_output
    if need <= caps.context_window - config.context.reserve: proceed
    elif caps.compaction and config.context.overflow == provider_compaction:
        req.compaction = enabled                                    # adapter passes the provider's server-side compaction; blocks returned are stored verbatim like any provider_opaque block
    else:                                                           # config.context.overflow == exhaust (default when compaction unavailable)
        emit(context_exhausted, turn_id, tokens=need, window=caps.context_window)
        session.state = exhausted                                   # POST /messages now returns 409 { error: "context_exhausted", resume_with: session.id }
        end_session(session, reason="exhausted")                    # mines leftovers, writes the session-summary memory (§11e)
        return Stop(Respond("This conversation has reached its length limit; start a new session (it will carry a summary)."))
  ```

  A client then creates a successor with `resume_from` (§11g), which seeds
  the tail of the transcript and relies on the summary memory for the rest.
  The prompt budget (§9e) already trims memories and optional fragments, so
  overflow here means the *conversation* itself no longer fits.

### 16c. The Anthropic adapter

```
AnthropicModelClient(config):
  client = anthropic SDK client (auth from env / profile / WIF; timeout config.model.timeout_s; max_retries config.model.max_retries)

  complete(req):
    body = {
      model: req.model,                                       # e.g. "claude-opus-5" (config; exact id, no date suffix)
      max_tokens: req.max_output,
      system: [ {type: text, text: b.text, cache_control: {type: ephemeral, ttl: config.cache.ttl}} if b is the LAST stable block
                else {type: text, text: b.text}   for b in req.system ],
      tools: [ {name, description, input_schema, strict: t.strict} for t in req.tools ],      # tools render before system: both cached by that breakpoint
      tool_choice: {type: auto},                              # never forced: unsupported on Claude Fable 5.1; strict:true keeps args valid
      messages: to_wire(req.messages) with cache_control on the last block of the most recently appended user turn,
      thinking: {type: adaptive} unless model is Fable (omit: always on),
      output_config: { effort: req.effort } + ({ format: {type: json_schema, schema: req.structured} } if req.structured),
      stream: true,
    }
    if config.model.fallbacks: body.fallbacks = "default"; betas += ["server-side-fallback-2026-07-01"]   # opt-in refusal fallback

    with client.messages.stream(**body) as stream:
      for ev in stream:                                       # wire events -> ModelEvent
        match ev.type:
          message_start:        yield message_start(ev.message.id, ev.message.usage)
          content_block_start:  if ev.content_block.type == tool_use: yield tool_call_start(ev.content_block.id, ev.content_block.name); buf[index] = ""
                                if ev.content_block.type == thinking: opaque[index] = collect
          content_block_delta:  match ev.delta.type:
                                  text_delta:       yield text_delta(ev.delta.text)
                                  input_json_delta: buf[index] += ev.delta.partial_json; yield tool_call_args_delta(id, ev.delta.partial_json)
                                  thinking_delta:   yield reasoning_delta(...) if config.model.show_reasoning  # display: summarized
          content_block_stop:   if index in buf: yield tool_call_end(id, json.loads(buf[index]))
          message_delta:        stop = ev.delta.stop_reason; usage = ev.usage
          message_stop:         final = stream.get_final_message()
                                ctx_blocks = [provider_opaque(anthropic, req.model, b) for b in final.content if b.type in (thinking, redacted_thinking)]
                                yield message_end(map_stop(stop, final.stop_details), usage, opaque_blocks=ctx_blocks)

  map_stop(stop, details):
    end_turn -> end_turn; tool_use -> tool_use; max_tokens -> max_tokens
    refusal  -> refusal{category: details.category, explanation: details.explanation}   # HTTP 200; the core turns it into an assistant message + audit log entry
    pause_turn / other -> other

  on SDK exception:
    RateLimitError (429), APIStatusError >= 500, APIConnectionError -> error{retryable, retry_after}   # SDK already retried max_retries times
    BadRequestError / AuthenticationError / PermissionDeniedError / NotFoundError / UnprocessableEntity -> error{fatal}   # never retried; turn fails with reason
```

Caching notes the adapter owns:

- Render order is tools, then system, then messages. One breakpoint on
  the last **stable** system block caches tools plus the shared prompt
  for every user of the deployment; one on the last user turn caches the
  conversation incrementally. Volatile system text sits after the first
  breakpoint and never poisons it.
- The adapter logs `usage.cache_read_input_tokens` per call; the core
  surfaces it in `turn_complete.usage`. Zero across repeated calls means
  something volatile crept before a breakpoint (timestamps, unsorted
  tools, a changing memories block placed too early).
- Short prompts silently do not cache (model-dependent minimum); that is
  expected for tiny deployments, not a bug.

### 16d. Other adapters

```
OpenAICompatibleModelClient  # for local or third-party servers speaking that dialect
LocalModelClient             # llama.cpp / ollama-style; capabilities() reports no caching, maybe no strict tools
FakeModelClient(script)      # tests: yields a scripted event list; used by every step and flow test in this document
```

Rules for a provider that lacks a capability:

- no `structured_output` -> the adapter asks for JSON in the prompt,
  validates against the schema, retries once with the validation error,
  then returns `error{fatal}` so the caller's fallback runs.
- no `parallel_tools` -> the adapter sets `parallel_tools=false`; the
  core's loop already handles one call per turn.
- no `caching` -> breakpoints are ignored; `usage` reports zero.
- no `forced_tool_choice` -> never needed; the core never forces.

### 16e. The `ClassifierEngine` port

```
ClassifierEngine {
  audit(AuditInput) -> AuditOutput                        # §8d
  mine(turns, hints, instructions) -> [MemoryCandidate]   # §6b
  summarise(session) -> MemoryCandidate                   # §11e
  kind: model | rules_only | remote
}
```

Implementations:

```
RulesOnlyEngine:   audit -> default_for(config.classifier.undecided_default, call)   # allow | ask | deny by effective risk
                   mine  -> []          summarise -> None
RemoteEngine(url): audit/mine/summarise -> POST to an http service that owns its own model; same input/output schemas; deadline enforced
ModelEngine(model_client, config):                         # the usual one; its own ModelClient, usually a cheaper model
```

`ModelEngine.audit`:

```
audit(inp):
  system = [ SystemBlock(AUDIT_SCAFFOLD, stable=True),                     # fixed text: role, output rules; never changes
             SystemBlock(inp.instructions, stable=True) ]                   # layered prompt_fragment rules (§9h); stable per token/layers
  user   = render_audit_input(inp)                                          # window (tool results summarised), the call, provenance, prior calls this turn
  req = ModelRequest(model=config.classifier.model, system=system, messages=[user(user)], tools=[],
                     max_output=384, effort=low, structured=AUDIT_OUTPUT_SCHEMA, deadline=config.classifier.audit_deadline_s,
                     metadata={purpose: audit})
  try:
      ev = collect(model_client.complete(req))                              # no streaming consumer; buffer to the end
      if ev.stop_reason is refusal or max_tokens: return AuditOutput.undecided(reason=ev.stop_reason)
      out = AuditOutput.parse(ev.text)                                      # schema-valid by construction when structured_output is supported
  except deadline_exceeded, error{retryable}:                               # after SDK retries
      out = AuditOutput.undecided(reason="classifier unavailable")
  audit_log.write(inp.call, out, model=req.model, usage=ev.usage)
  return out

AUDIT_OUTPUT_SCHEMA = { type: object, additionalProperties: false,
                        required: [p_requested, effect_summary, reason],
                        properties: { p_requested: {type: number}, effect_summary: {type: string}, reason: {type: string},
                                      mismatch: { type: object, additionalProperties: false, required: [asked_for, tool_does],
                                                  properties: { asked_for: {type: string}, tool_does: {type: string} } } } }
# no numeric min/max in the schema (not supported); clamp p_requested in code

apply_thresholds(out, t):   # §8d
  undecided -> default_for(config.classifier.undecided_default)
  p >= t.allow -> ALLOW;  p <= t.deny -> DENY;  else ASK_USER
```

`ModelEngine.mine` is the same shape with the mining scaffold, the
turn(s) and recalled memories as input, `max_output` in the low
thousands, a schema of `{candidates: [ {kind, body, rationale, confidence, audience, tags, links} ]}`,
and no deadline (it runs off the critical path). `summarise` returns
one candidate of kind `episodic`.

Why the classifier gets its own `ModelClient`:

- **Different model.** The audit sits on the critical path with a
  sub-second budget; the deployment may pick a smaller, faster model
  for it than for the agent. The port makes that a config line.
- **Different caching.** The audit scaffold plus layered instructions is
  a small stable prefix that caches across every call in the deployment;
  the varying window comes after the breakpoint.
- **Different failure policy.** Agent model errors fail the turn.
  Classifier errors degrade to `undecided_default`, logged, so a
  classifier outage never blocks or silently allows.

### 16f. Configuration

```yaml
model:
  adapter: anthropic
  agent:      { model: claude-opus-5, max_output: 16000, effort: high, show_reasoning: false, fallbacks: true }
  timeout_s: 600
  max_retries: 2
  cache: { ttl: 5m }                      # 1h for bursty traffic with long gaps

classifier:
  engine: model                           # model | rules_only | remote
  model: <a smaller, faster model id>     # deployment's choice; the audit has a sub-second budget
  audit_deadline_s: 2.5
  thresholds: { allow: 0.85, deny: 0.15 }
  undecided_default: ask_if_risk_gte_medium_else_allow
```

### 16g. Worked example: one audit call on the wire (abbreviated)

```
POST /v1/messages   (adapter builds this from the ModelRequest in 16e)
{
  "model": "<classifier model>", "max_tokens": 384, "stream": true,
  "system": [
    { "type": "text", "text": "<AUDIT_SCAFFOLD>" },
    { "type": "text", "text": "<global audit-instructions fragment>\n<team-a fragment>", "cache_control": { "type": "ephemeral" } }
  ],
  "output_config": { "effort": "low",
                     "format": { "type": "json_schema", "schema": { ...AUDIT_OUTPUT_SCHEMA... } } },
  "messages": [ { "role": "user", "content": [ { "type": "text", "text":
      "<window>\nuser: show me the upload config\nassistant: (called ci_status -> failed: 3 tests)\n</window>\n<call name=\"delete_file\" layer=\"global\" risk=\"high\">{\"path\": \"/etc/upload/config.yaml\"}</call>" } ] } ]
}

stream: message_start -> content_block_start(text) -> text_delta... -> content_block_stop -> message_delta(end_turn) -> message_stop
text:   {"p_requested": 0.08, "effect_summary": "delete /etc/upload/config.yaml", "reason": "user asked to see the file, not remove it",
         "mismatch": {"asked_for": "show", "tool_does": "delete"}}
-> AuditOutput -> thresholds: 0.08 <= 0.15 -> DENY -> Stop(Deny) in §8d
usage: cache_read_input_tokens > 0 on every call after the first for this token's layers
```

### 16h. Testing the ports

```
FakeModelClient([ text_delta("Checking…"), tool_call_start(c1, "ci_status"), tool_call_args_delta(c1, '{"job":'),
                  tool_call_args_delta(c1, '"build-nightly-linux"}'), tool_call_end(c1, {...}), message_end(tool_use) ])
FakeClassifier(verdicts={"delete_file": DENY, "*": ALLOW})
```

Every flow in this document is testable with those two fakes and
in-memory sources. Each real adapter additionally has a contract test:
stream a tool call, replay a conversation with opaque blocks, request a
structured output, and observe a cache hit on the second call.

## 17. Event bus and streaming layer

Everything a client sees is an event. Steps, runners, the classifier,
the miner, and the job runner publish; one per-session bus assigns
sequence numbers, appends to a durable log, pushes to the session's
single stream, and fans out to internal listeners. Delivery to the
client is best-effort; the log is the source of truth.

### 17a. Envelope

```
Event {
  seq:        int            # per session, monotonic, assigned by the bus; also the SSE id
  type:       text           # e.g. assistant_delta, tool_call_proposed (03-web-api.md table)
  session_id, turn_id?, ts
  level:      info | debug   # debug: audit_verdict, memory_recalled, tool_hidden, ...; filtered per stream
  payload:    object
  v:          1              # envelope version; additive changes only, unknown fields ignored by clients
}
```

### 17b. The per-session bus

```
EventBus(session):
  log:       EventLog                      # durable, seq-numbered (17c)
  stream:    StreamTransport?              # the one open connection, or None
  listeners: [Listener]                    # internal: metrics, audit log, notifier, webhooks
  lock:      per-session mutex             # seq assignment and ordering happen under it

  publish(type, payload, level=info, turn_id=None):
    with lock:
        e = Event(seq=log.next_seq(), type, payload, level, turn_id, ts=now())
        if level == info or config.events.persist_debug: log.append(e)     # durable before delivery
    stream?.send(e)                                                          # best effort; may drop on backpressure (17d)
    for l in listeners: l.on_event(e)                                        # never on the hot path: listeners enqueue, process async
    return e.seq
```

One mutex per session is enough: only one turn runs at a time, and
background producers (mining, jobs, sweepers) are few and short.
Everything that wants to emit goes through `publish`; nothing writes to
the stream directly.

### 17c. The event log

```
EventLog(session_id, store):
  append(e) -> seq          # store.append; batched fsync per config.events.flush_ms
  since(seq) -> [Event]     # for replay
  oldest_seq() -> int       # after retention trimmed the head
  retention: keep last config.events.max_events (e.g. 5000) or config.events.max_age (e.g. 7d), whichever trims more
  compact_turn(turn_id):    # after turn_complete (17e)
    deltas = events of type assistant_delta with this turn_id
    replace them with nothing; the assistant_message event already carries the full text
    (replay for a completed turn therefore yields assistant_message, not thousands of deltas)
```

Deltas are persisted *during* a turn so a mid-turn reconnect can replay
partial text; once the turn completes they are compacted away.

### 17d. SSE transport

```
GET /sessions/{id}/events?level=info|debug      Accept: text/event-stream

SSEStream(conn, level):
  send(e):
    if e.level == debug and level != debug: return
    frame = f"id: {e.seq}\nevent: {e.type}\ndata: {json(e)}\n\n"
    if not buffer.try_put(frame, max=config.events.stream_buffer_kb):        # slow client
        close(reason="stream_overflow")                                      # client reconnects with Last-Event-ID; log has everything
  heartbeat: every config.api.heartbeat_s send ": keepalive\n\n"           # comment frame; keeps proxies and the client's watchdog happy
  on open (11b): replay log.since(last_event_id) (level-filtered), then session_state, then live
```

Coalescing keeps the log and the wire sane:

```
DeltaCoalescer(bus, flush_ms=30):
  on text_delta from the model: buf += text
  every flush_ms, or when a non-delta event is about to be published, or at message end:
      if buf: bus.publish(assistant_delta, {text: buf}); buf = ""
# one assistant_delta per ~30 ms instead of one per token; ordering with tool_call_proposed etc. is preserved
# because the coalescer flushes before any other event from the same turn is published
```

Encoding rules: JSON per frame, no multi-line data, `id` is the seq so
`Last-Event-ID` works with no extra state, `event` is the type so
`EventSource` listeners can bind by name, and `retry:` is sent once on
open with the server's preferred reconnect delay.

### 17e. WebSocket transport (optional adapter)

```
GET /sessions/{id}/ws   (Upgrade)          # config.api.streaming: sse | websocket | both

client -> server frames:  { "type": "hello", "last_seq": 4127, "level": "info" }
                          { "type": "user_message", ... }  { "type": "tool_result", ... }  { "type": "approval_response", ... }
                          { "type": "cancel", ... }  { "type": "memory_decision", ... }  { "type": "job_progress", ... }
server -> client frames:  the same Event envelope as SSE, plus { "type": "ack", "for": <client frame id>, "seq": <event seq or null> }

WSStream.on_frame(f):
  route f.type to the same handler the POST endpoint uses (ApiLayer.post_message, .tool_result, .answer_approval, ...)
  reply ack with the resulting seq or an error object
  # no new semantics: a WebSocket is SSE + POST over one connection. Replay on `hello` exactly as 11b.
```

Default stays SSE plus POST: it works through every proxy and CDN, needs
no framing library on the client, and replay is built into the protocol.
WebSocket is for clients that want one connection or true full-duplex
streaming of client tool output.

### 17f. Ordering across producers

- Within a turn: the turn runner publishes in program order; the
  coalescer flushes before any non-delta event; per-tool-call events for
  concurrently audited calls interleave, but each call's own events stay
  ordered (`proposed` → `started` → `finished`/`denied`).
- Across producers: mining, jobs, sweepers, and the notifier all call
  `publish` on the owner's session bus; the mutex serialises seq
  assignment, so a client never sees seq go backwards, even when a
  `memory_created` from last turn's mining lands in the middle of this
  turn's deltas.
- `turn_id` on every event lets a client attribute late events to the
  right turn.

### 17g. Service-level routing

Some producers have no session in hand (a fork made through the API, a
job finishing after its session ended):

```
ServiceBus.publish_to_owner(owner, type, payload):
  session = sessions.latest_active_for(owner) or sessions.latest_idle_for(owner)
  if session: session.bus.publish(type, payload)                           # lands in that session's log and stream
  else: notifier.send(owner, type, payload)                                # no session: out-of-band (13d); nothing is logged in a session
```

Cross-session events therefore always appear in *some* session log of
the owner, or go out of band; they are never dropped silently.

### 17h. Internal listeners

```
Listener { on_event(e) }                          # enqueue only; a worker drains the queue

MetricsListener:   counts and latencies per type; per-layer timings from context_ready; cache hit rates from turn_complete.usage
AuditLogListener:  persists tool_call_*, audit_verdict, approval_*, memory_*, skill_loaded, shadow_refused, chain_rebuilt
                   to the audit store (queryable via GET /classifier/verdicts and friends)
NotifierListener:  tool_job_finished / memory_confirm for owners whose session has no open stream -> notifier adapter (13d)
WebhookListener:   deployment-configured: POST selected event types to a URL (e.g. turn_complete for billing)
```

Listeners never block `publish`; a slow listener falls behind on its own
queue and is reported, not the client.

### 17i. Client-side reconnect loop

```
client.run():
  last = load_last_seq() or 0
  loop:
    try:
      es = open_event_stream(session_id, token=fresh_token(), last_event_id=last, level="info")
      for e in es:
        last = e.seq; save_last_seq(last)
        match e.type:
          session_state:      rebuild_view(e); answer any pending approvals / client calls it lists
          assistant_delta:    append_text(e.turn_id, e.payload.text)
          assistant_message:  replace_text(e.turn_id, e.payload.text)          # authoritative; covers replay after compaction
          tool_call_proposed: if e.payload.execute_on == "client" and e.payload.state == "execute": run_local_tool(e)
          approval_requested: prompt_user(e)
          memory_confirm:     prompt_user_nonblocking(e)
          replay_gap:         refetch_session(); last = e.payload.oldest
          token_expiring:     refresh_token_soon()
          auth_expired, stream_replaced, session_ended: break
          _:                  ignore                                            # forward compatible
    except disconnected, stream_overflow:
      sleep(backoff()); continue
```

Two invariants make clients simple: `assistant_message` always
supersedes any deltas for its turn, and `session_state` after every open
is enough to rebuild the view without interpreting replayed events.

### 17j. Worked timeline with coalescing and compaction

```
live (client connected):
  seq 40  turn_started
  seq 41  context_ready
  seq 42  assistant_delta  "Checking the nightly "        (coalesced ~30 ms of tokens)
  seq 43  assistant_delta  "build…"
  seq 44  tool_call_proposed ci_status                    (coalescer flushed first)
  seq 45  memory_created    (from LAST turn's mining; interleaved, seq-ordered, turn_id=previous)
  seq 46  tool_call_started ci_status
  ...
  seq 58  assistant_message  full text
  seq 59  turn_complete
  compact_turn: seq 42, 43, 50-57 (deltas) removed from the log; seq numbers are not reused

reconnect later with Last-Event-ID: 41
  replay: 44, 45, 46, ..., 58, 59         (no deltas; assistant_message carries the text)
  session_state {turn: idle, pending: []}
```

## Things this walkthrough pins down

- `allow` rules return `Continue` with a recorded verdict and mark the
  call decided; only `deny` and `require_approval` return `Stop`. This
  keeps "only gating stops the chain" true while still short-circuiting
  the model audit.
- Loading a skill mid-turn mutates `ctx.tools` and `ctx.prompt` at the
  skill's layer, and the next model call re-derives its tool list and
  system prompt from the context.
- Templated layers (team, user) are cached per template value; the outer
  chain is cached per session and rebuilt if the token is refreshed with
  different scopes or groups.
- `ctx.tools.resolve` is a dictionary lookup, because precedence was
  settled when the accumulator was filled.
- Approval resumes the chain from the step after the asker; later layers
  may still deny but may not re-ask. A user approval never overrides a
  `deny`, and a remembered approval never overrides a locked asker.
- Mining rules split by direction: restrictive rules (drop, threshold,
  redact, disable) from any layer apply to every candidate before
  routing; widening rules (retag, auto-accept) apply only at their own
  layer. A user can therefore disable mining for themselves even though
  the team layer runs first.
- Dedupe happens at write time against the target store, without a
  model call: duplicate → update, contradiction → supersede with a link,
  otherwise insert.
- Mining writes as the user with the session token; if the token is
  about to expire the turn is queued and mined with the next fresh one.
- Promotion re-runs the locked restrictive rules, so policy applies to
  explicit user acts too.
- A fork's tools are re-labelled with the fork's layer and clipped to
  that layer's capability ceiling, so a fork can never run with more
  privilege than a skill authored directly in that layer.
- Staleness is detected inside `on_message` by comparing the fork's
  recorded base version with what the base layer just put into the
  context; no extra lookup, and unknown when the base layer is absent.
- Diff and rebase are structural (instructions, tools by name, fragments
  by id), three-way when the base store can serve old versions, two-way
  otherwise.
- Static audit rules split by direction like mining rules: `deny` and
  `require_approval` accumulate across layers and no layer can remove
  another's; `allow` acts at its own layer and yields to any restriction
  unless it is locked, in which case it shields the call from lower
  layers. Peer-team conflicts therefore need no merge strategy.
- Risk escalation by tool layer and the scope gates run before any
  layer's rules and cannot be configured away by a layer.
- A layer's `StaticRulesStep` evaluates only that layer's rules; there is
  no global re-sort of `ctx.rules`.
- Recalled memories and the skill index are derived fragments, so one
  budget applies to everything; they shrink before anything is dropped.
- Section order puts stable shared content first and volatile per-user
  content last, so provider prefix caches hit across users and across
  calls within a turn.
- Locked fragments are never dropped; if they alone exceed the budget the
  turn fails rather than silently dropping policy.
- Recall is model-free: the query is the message plus a slice of the last
  assistant reply, each store searches its own way, ranks are fused with
  RRF, and a small pipeline decides what fits the budget.
- Pinned memories bypass ranking and are the last to be dropped; a
  promoted memory that exists in two layers is recalled once.
- The recall policy and the prompt assembler share one tail: shrinking
  the memories fragment pops the same list recall selected from.
- Stores own embeddings; a shared embedder is optional; nothing in the
  core depends on scores being comparable across stores.
- Events are appended to a durable per-session log before delivery, so a
  reconnect is a replay from `Last-Event-ID` followed by a
  `session_state` snapshot; nothing server-side waits on the stream.
- A session never changes owner; a newer token from the same subject is
  swapped in on any request, and the chain is rebuilt only if scopes or
  groups changed. A running turn keeps the token it started with.
- Ending a session drains mining, optionally writes a session-summary
  memory through the normal routing chain, then applies retention. The
  chain itself is never persisted; it is rebuilt from token and config.
- Capabilities are clipped to the layer ceiling when a tool enters
  `ctx.tools`, so the model never sees a tool it cannot run; enforcement
  at run time is per impl kind: host API for builtins, host allowlist for
  http, the sandbox for scripts, and the client for client tools.
- Secrets are resolved by name inside runners (or attached by the egress
  proxy) and never enter the context, logs, or results; results are
  redacted and size-bounded before the model sees them.
- The v1 sandbox is a restricted subprocess (namespaces, binds, egress
  proxy, limits); container and wasm are adapters with the same contract.
- A background job is owned by the user, not the session; its result
  re-enters the pipeline as the late result of the original tool call,
  either as an agent-initiated turn or injected on the next message, and
  is never lost (jobs list, notifier adapter for owners with no session).
- Agent-initiated turns run the same `on_message` gates as user
  messages, never interrupt a running turn, and fall back to notify-only
  when a gate refuses or the token is no longer valid.
- The session layer is built from session state, not stores; everything
  it contributes is unlocked and applies last, so a client can shadow
  unlocked tools and add restrictions but never remove one or shield a
  call.
- Client tools are limited by construction: client-only capabilities, a
  risk floor plus session escalation, execution only on the client, and
  untrusted results. Client gates are declarative rules, never code.
- A client gate governs what the client will do; restricting server-side
  tools is a user-layer or global rule.
- Skill loading is a tool call (`load_skill`), so it is audited and can
  be rule-gated like any other call; auto-load by trigger takes the same
  path with `auto: true` matchable.
- Loaded skills are sticky for the session and version-pinned; their
  tools and fragments are re-put each turn from cached session state,
  and instructions stay live as a shrinkable fragment.
- Discovery is store-side and capped per layer; pinned, loaded, and
  trigger-suggested skills are always listed, everything else competes
  on relevance.
- `ModelClient` is one provider behind one streaming interface; the
  core renders requests with stable-first system blocks, deterministic
  tool order, append-only history, verbatim assistant blocks, and all
  tool results of a turn in one user message, so any adapter can cache
  and replay correctly.
- The Anthropic adapter never forces tool choice, uses strict tools and
  structured output for schema guarantees, opts into refusal fallbacks
  by config, maps refusal to a stop reason the core handles, and lets
  the SDK retry 429/5xx while treating 4xx as fatal.
- The classifier has its own `ModelClient` (its own model, cache prefix,
  and failure policy); an unavailable or refusing classifier degrades to
  the configured undecided default, never to a silent allow.
- The global layer is always in the chain as the policy baseline; only
  its skills, tools, and memories need a scope. Restrictive mining rules
  run once before routing. The global `load_skill` allow is unlocked so
  lower-layer restrictions still fire. Late job results are user-role
  marker messages. Context overflow ends the session via
  `context_exhausted` unless provider compaction is on. User-layer paths
  are templated on `${user_root}` per subject (`decisions/0010`).
- One bus per session assigns seq under a mutex and appends to a durable
  log before best-effort delivery; deltas are coalesced on the wire and
  compacted out of the log once the turn completes, so replay of a
  finished turn yields the consolidated message.
- SSE plus POST is the default transport; WebSocket is an optional
  adapter that routes frames to the same handlers. Cross-session events
  land in some session log of the owner or go out of band, never dropped.
