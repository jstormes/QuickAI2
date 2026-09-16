# Requirements

The goals from `00-vision.md` restated as testable requirements, and the
non-functional rows an audit will ask about. Targets marked **TBD
(owner)** are decisions the project owner has not yet made; the
"bounded by" column shows the configuration default that currently
limits the behaviour, from `02-layering-and-composition.md`.

## A. Functional requirements (from the seven goals)

| # | Requirement | Acceptance criterion |
|---|---|---|
| R1 | Memories are stored behind a `MemoryStore`/`MemoryWriter` port; the core compiles with no adapter imported. | Static check: the core package imports no adapter module. Contract test per adapter: put, search, get-with-membership, touch, supersede. |
| R2 | Memories are organised (kind, tags, links, layer, owner) and semantically searchable across every permitted layer in one query. | Integration test: three stores (global read-only, team, user) each return hits for one query; fused order matches RRF; a memory outside the caller's chain is `NotFound` by id. Recall quality target: **TBD (owner)** (e.g. recall@6 on a labelled set). |
| R3 | Skills are discoverable by summary and loaded on demand, from several stores per layer, with same-name shadowing and locking. | Integration test: two skill stores in the global layer, one in user; a user fork shadows the global skill; a locked global skill refuses the shadow with `shadow_refused`. |
| R4 | A classifier audits every tool call the model proposes; no call runs without a verdict from a rule, the model audit, or an approval. | Test: with `FakeClassifier` denying one tool, that tool never executes; dry-run log lists it. Accuracy target on the eval set (false-allow rate, false-deny rate): **TBD (owner)**, measured from `audit_dry_run` records. |
| R5 | Memories are created in the background by the miner, never on the critical path, and routed personal-by-default. | Test: `turn_complete` precedes `memory_created`; a team-tagged candidate with no positive signal lands in the user store with `suggested_audience`. |
| R6 | Three client kinds (CLI, web, app) can run at once against one service, each in its own session. | Integration test: three sessions for one subject; no event from one appears on another's stream; skills, memories, and the classifier are shared. |
| R7 | Which layers a request touches is decided only by the token's `sub`, `scopes`, and `groups`. | Test matrix over the scope-to-layer table in `04`: each scope combination yields the expected outer chain from `chain_for`. |
| R8 | The system prompt is assembled from every layer's fragments with override-by-id, locking, section allow-lists, and budgeting. | Test: user fragment overrides an unlocked global id; locked global id survives; a team fragment in `policy` is refused; over budget drops optional fragments and never locked ones. |
| R9 | The service is reachable only through the streaming web API; every client-visible effect is an event in the per-session log. | Test: replay from `Last-Event-ID` after a mid-turn disconnect reproduces the turn; `session_state` rebuilds the client view. |
| R10 | Every script tool runs in a sandbox; secrets never appear in context, logs, or results. | Test: a script tool that prints its environment shows no secret; a result containing a registered secret pattern is redacted; a path outside the grant is refused with `capability`. |
| R11 | The core never blocks on background work. | Test: mining and job delivery run after `turn_complete`; a slow mining store does not delay the next turn. |

## B. Non-functional requirements

| Row | Requirement | Bounded today by (config default) | How measured |
|---|---|---|---|
| Time to first token | **TBD (owner)** | slowest single `on_message` source; `memory.step_timeout_ms` 300 | `context_ready` per-layer timings; p50/p95 over a day |
| Turn latency (no tools), p50 / p95 | **TBD (owner)** | model provider | `turn_complete` minus `turn_started` |
| Tool-call audit latency | **TBD (owner)** (typical case) | hard cap `classifier.audit_deadline_s` 2.5 | `tool_call_started` minus `tool_call_proposed`; `audit_verdict` latency field |
| Concurrent sessions per node | **TBD (owner)** | `sessions.max_per_subject` 20; no per-node cap | `/metrics` gauge |
| Sessions per subject | 20 (default; owner may change) | `sessions.max_per_subject` | 429 on create above the cap |
| Availability | **TBD (owner)** | single-node v1; restart marks running turns `turn_interrupted` | uptime of `/health` = ok |
| Durability of events | **TBD (owner)** | `events.flush_ms` 50 (loss window on crash) | crash test: events after last fsync are lost, earlier ones replay |
| Retention: sessions / events / audit / assets / memories | 30 d / 7 d / 90 d / 7 d / unlimited (defaults; owner may change) | `sessions.retention`, `events.max_age`, `audit.retention`, `assets.ttl` | sweeper metrics |
| Cost per turn | **TBD (owner)** | no cap; one classifier call per undecided tool call, one mining call per turn, optional summary per session | `turn_complete.usage` summed per session and per subject |
| Model context headroom | never exceeded | `context.reserve` 8000 tokens; `context_exhausted` ends the session | count of `context_exhausted` events |

## C. Test strategy

- Every flow in `08-walkthrough.md` is testable with `FakeModelClient`
  (scripted events) and `FakeClassifier` (`08` §16h) plus in-memory
  sources; the walkthrough's event timelines are the expected outputs.
- Each adapter has a contract test (`08` §16h): stream a tool call,
  replay opaque blocks, structured output, a cache hit on the second
  call; stores: put/search/get/touch/supersede with membership.
- Classifier calibration uses dry-run mode (`08` §8e): new global rules
  are trialled with `audit_dry_run` before enforcement; the accuracy
  rows above are measured from the same records.
- Configuration is validated at startup (`02`); the validation rules are
  themselves tested with a fixture per rule.
