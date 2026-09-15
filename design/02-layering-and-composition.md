# Layering and composition

How multiple sources of the same thing coexist.

## Layers

Decided layers, lowest precedence first (`decisions/0005-team-layer.md`):

| Layer     | Owner                      | Holds                                          |
|-----------|----------------------------|------------------------------------------------|
| global    | platform/infra team        | shared skills repo, shared memory service, org policy prompt fragments |
| team      | a team's maintainers       | team skills repo, team memory store, team prompt fragments |
| user      | the individual             | personal skills, personal memories, preferences|
| session   | runtime                    | ad-hoc instructions, temporary memories        |

A user may belong to **several teams**. The chain builder adds one
team-layer entry per team id in the token's `groups` claim, ordered by
the IdP's claim order (or alphabetically if unordered). Same-name skills
across two of a user's teams are a conflict; see IDEAS.md.

Other candidate layers (framework defaults, project) are not decided. The
core only requires that layers be totally ordered, so adding one is a
configuration and permission change, not a core change. **The layer list
is the developer's to set**: a solo deployment may have only user and
session; a large one may have several shared layers. See
`05-design-patterns.md` for how the layer chain is built from that
list.

### Team identity

- Team membership comes from the token: a `groups` claim listing team
  ids (claim name configurable). The IdP is the source of truth.
- Permissions for the team layer are *not* per-team strings. A token
  holds `use-team-skill` once, and the `groups` claim says which teams it
  applies to. This keeps the permission set small as teams multiply.
- Team stores are configured per team id; a team without a configured
  store simply contributes nothing.

## Composition rules

All of these are consequences of chain order plus locking
(`07-turn-pipeline.md`): layers run global → team → user → session,
later layers overwrite earlier unlocked values, locked values are
final.

- **Skills:** the same `name` in a later layer shadows the earlier one.
  Both remain addressable by full id so the shadowed one can still be
  loaded explicitly. A skill marked `locked` in an earlier (more trusted)
  layer is exempt: it cannot be shadowed. Users create overrides by forking
  (`01-core-abstractions.md` §3).
- **Memories:** never shadowed; all layers contribute to search. Ranking may
  boost nearer layers (user > team > global) since they are more
  specific. Agent-driven writes target only personal or team, chosen by
  the memory routing policy (`01-core-abstractions.md` §2) with personal
  as the fallback; global memory is admin-curated and read-only here.
- **Prompt fragments:** grouped by section; within a section, priority
  descending, then the earlier layer first, then id. A fragment can be
  marked `locked` so later layers cannot override it (used for org
  policy).
- **Permissions gate layers.** A (store, layer) pair participates in a
  request only if the token carries the matching permission
  (`04-auth-and-permissions.md`). Missing permission means the layer is
  silently absent, not an error.

## Configuration shape

The layer list *is* the outer chain. The full sketch, with every
resource nested under its layer (user-layer paths templated on
`${user_root}`), is in `07-turn-pipeline.md` (Configuration view).
Service-level settings sit beside it:

```yaml
layers: [...]                      # see 07-turn-pipeline.md

classifier:
  engine: model                    # model | rules_only | remote
  model: <small fast model>
  tool_audit: { enabled: true }
  memory_mining: { enabled: true }

tools:
  sandbox: { kind: subprocess_restricted, uid: agent-sbx }   # every script impl is sandboxed
  runners:
    http: { timeout_s: 30 }

api:
  listen: 127.0.0.1:8080
  streaming: sse            # sse | websocket | both
  auth:
    adapter: oauth2_jwt     # or dev_idp for local single-user mode
    issuer: https://idp.example.internal
    audience: basic-agent
    permission_claim: scope # which token claim carries permission scopes
```

## Failure behavior

- A source that is unreachable should degrade, not fail the turn. The
  step logs it and contributes nothing; the other steps still run. A
  `required` flag on a contributing step opts it into hard failure (e.g.
  the org policy prompt).
- Caching per source (with TTL) is an adapter concern, but every source
  exposes "refresh" so a front end can offer a reload command.

## Configuration reference (defaults)

Every `config.*` key the walkthrough uses, with its default. A key not
listed here is a bug in the notes, not an implementer's decision.

| Key | Default | Notes |
|---|---|---|
| `users_root` | required | per-subject root for `${user_root}`; single-user mode: `~/.agent` with `subject: local` |
| `layers[]` | required | outer chain; `global` always present; see `07-turn-pipeline.md` |
| `sources.git.push_interval_s` | 0 (push per write) | git-backed writers |
| `model.adapter` / `model.agent.model` | `anthropic` / deployment's choice | see `08-walkthrough.md` §16f |
| `model.agent.max_output` / `effort` | 16000 / `high` | |
| `model.timeout_s` / `max_retries` | 600 / 2 | |
| `model.cache.ttl` | 5m | |
| `context.reserve` | 8000 tokens | headroom kept below the window |
| `context.overflow` | `provider_compaction` if the adapter supports it, else `exhaust` | |
| `prompt_budget` | window − reserve − conversation estimate | |
| `prompt.provenance_comments` | false (true for `GET /prompt`) | |
| `classifier.engine` / `model` | `model` / deployment's choice | |
| `classifier.audit_deadline_s` | 2.5 | the sub-second *goal* is a target, this is the hard cap |
| `classifier.thresholds.allow` / `deny` | 0.85 / 0.15 | |
| `classifier.undecided_default` | `ask_if_risk_gte_medium_else_allow` | values: `allow`, `ask`, `deny`, `ask_if_risk_gte_<level>_else_allow` |
| `classifier.window` / `prompt_budget` | 8 messages / 4000 tokens | |
| `memory.k_per_store` / `max_recalled` / `max_per_kind` | 8 / 6 / 3 | |
| `memory.min_score` / `half_life` | 0.0176 (RRF) / off | |
| `memory.layer_boost` | user 1.2, team 1.1, global 1.0 | |
| `memory.merge` / `embedder` | `rrf` / per store | |
| `memory.step_timeout_ms` | 300 | per layer search step |
| `mining.min_token_ttl` | 60 s | queue the turn if the token expires sooner |
| `mining.dup_threshold` / `dedupe_window` | 0.92 / 24 h | |
| `mining.confirm_timeout_s` | 86400 | `on_timeout: keep_personal` |
| `skills.sticky` / `max_loaded` / `k_per_layer` / `resource_kb` | `session` / 6 / 5 / 256 | |
| `tools.ceilings.<layer>` | required per configured layer | |
| `tools.sandbox.kind` / `uid` | `subprocess_restricted` / required | every script impl is sandboxed |
| `tools.runtimes` | `{}` (no scripts allowed) | allowlist |
| `tools.limits.*` | wall 60 s, cpu 30 s, mem 512 MB, pids 64, stdout 256 KB, result 64 KB | |
| `tools.risk_escalation` | team +1, `global-from-skill` +1, session +1 | keyed by `origin`/layer |
| `tools.builtin_refs_allowed_from` | `[global]` | |
| `tools.on_excess` | `hide` | or `degrade` |
| `tools.client_risk_floor` / `client_wall_s` / `client_result_kb` | `low` / 300 / 64 | |
| `tools.max_concurrent` / `rate.session` / `rate.<tool>` | 4 / 60 per min / none | |
| `tools.redact_patterns` | built-in secret patterns | plus `secrets.known_values()` |
| `jobs.*` | see `08-walkthrough.md` §13g | |
| `sessions.idle_timeout` / `retention` / `retain_transcript` | 30 min / 30 d / `duration` | |
| `sessions.summary` / `summary_min_turns` / `resume_tail_turns` | `personal` / 3 / 6 | |
| `sessions.token_at_rest` | `encrypted` (key from `secrets`) | or `never` (disables post-restart mining) |
| `messages.queue` | `reject` | or `queue_one` |
| `gate.content_deny_patterns` | `[]` | |
| `assets.max_kb` / `inline_kb` / `ttl` | 10240 / 256 / 7 d | |
| `events.flush_ms` / `max_events` / `max_age` / `stream_buffer_kb` / `persist_debug` | 50 / 5000 / 7 d / 512 / false | |
| `api.heartbeat_s` / `token_warning_s` / `streaming` | 15 / 300 / `sse` | |
| `audit.retention` | 90 d | audit store |
| `notifier.adapter` | `none` | |

## Startup validation

The service refuses to start (exit with `config_invalid` and the list
of problems) when:

- a layer, step, adapter, runtime, or sandbox kind is not registered;
- a template variable (`${team_id}`, `${user_root}`, `${workspace}`) is used where it cannot be resolved, or a user-layer path is not under `${user_root}`;
- `tools.ceilings` lacks an entry for a configured layer, or grants a capability kind the layer may not have (server-side capabilities on `session`);
- `required: true` is set on a gating step, or `locked: true` on a rule/fragment in a layer that may not lock (session; team unless the store allows it);
- `risk_escalation` names an unknown layer or origin;
- the global layer is missing, or any layer name is not one of the known set plus configured extras;
- `users_root` is unset outside single-user mode;
- a `required` source is unreachable at `prebuild` (global sources are prebuilt; others are checked lazily and reported).

Warnings (start anyway, log once): a source with no writer behind a
writable endpoint; a ceiling broader than the earlier layer's; a classifier
deadline above 5 s; `tools.runtimes` empty while a skill store contains
script tools.
