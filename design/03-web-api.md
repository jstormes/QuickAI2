# Web API with streaming

The framework's public surface is an HTTP API. Everything a front end can
do, it does through this API. This note sketches the shape; it is not a
final spec.

## Why

- One protocol serves CLI, web, and native apps at once.
- The service is long-running, which gives background work (memory
  mining, classifier audits, source refresh) a natural home.
- Each client gets its own session, so the service can serve many
  clients (and many front-end types) at the same time without sharing
  conversation state between them.
- Front ends can be written in any language.

## Resources (sketch)

```
GET    /sessions?state=                   the caller's own sessions (history picker)
POST   /sessions                          create -> {session_id}; body: active_team?, client_kind?, resume_from?, job_delivery?: agent_turn | notify,
                                          client_tools?, instructions?, gates?
PATCH  /sessions/{id}                     change active_team (must be in token groups) and/or session instructions
GET    /sessions/{id}                     metadata, state, turn state, token subject + scopes, chain layers
POST   /sessions/{id}/end                 end now (drain mining, optional summary memory, retention applies)
DELETE /sessions/{id}                     end and purge immediately

POST   /sessions/{id}/messages            send user_message; returns turn_id (409 turn_in_progress while a turn runs; 409 context_exhausted with
                                          resume_with once the conversation no longer fits the model's window). ?wait=true blocks until
                                          turn_complete, or returns 202 with the pending state if the turn parks on an approval or client call.
                                          Optional Idempotency-Key header: a repeat within 24 h returns the original response.
GET    /sessions/{id}/events?level=       stream (SSE) of outbound events; id=seq, event=type, heartbeat comments; level=info|debug
GET    /sessions/{id}/ws                  optional WebSocket: same events out, POST-equivalent frames in (08-walkthrough.md §17e)
POST   /sessions/{id}/tool-results        client-executed tool result { tool_call_id, ok, content }; untrusted, capped, redacted; Idempotency-Key supported
GET    /sessions/{id}/assets/{ref}?range=  full output of a truncated tool result (also reachable by the model via read_asset)
POST   /sessions/{id}/jobs/{job_id}/progress   client-executed background tool reports progress (owner check; the job's origin session must be this session)
GET    /jobs?state=&delivered=            the caller's background jobs across sessions
GET    /jobs/{id}                         state, progress, result if finished, origin (owner check)
POST   /jobs/{id}/cancel                  (owner check)
POST   /sessions/{id}/approvals/{aid}     answer an approval_requested { decision, remember?, note? }
GET    /sessions/{id}/approvals?state=pending&kind=tool|memory   outstanding requests (for reconnect)
POST   /sessions/{id}/cancel

GET    /memories?q=&kinds=&tags=&layers=&k=&include_superseded=   search across permitted layers; rank-fused, no recall policy
GET    /memories?suggested_for=team:T     personal memories the miner suggested for team T
GET    /memories/{id}
POST   /memories/{id}/promote?layer=team:T copy a personal memory into a team layer (provenance kept)
POST   /memories                          explicit write (respects layer rules)
DELETE /memories/{id}

GET    /skills?q=&layers=&k=              discover across permitted layers (summaries incl. tool names, pinned/suggested flags)
GET    /skills/{id}/resources/{path}      a skill resource file (size-capped)
POST   /sessions/{id}/skills/{name}/load  client-initiated load (audited as a tool call)
DELETE /sessions/{id}/skills/{name}       unload
GET    /sessions/{id}/skills              loaded skills: version, outdated flag, last used
GET    /skills/{id}                       load full skill incl. tool definitions
POST   /skills                            create skill in a layer (needs create-* permission)
PUT    /skills/{id}
DELETE /skills/{id}
POST   /skills/{id}/fork?layer=user       copy a skill into a later (less trusted) layer under the same name
                                          (&overwrite=true, &strip_excess=true; 409 if base is locked)
GET    /skills/{id}/base                  the skill this one was forked from, if any
GET    /skills/{id}/diff                  three-way diff (base then / base now / fork) or two-way if the base store cannot serve old versions
POST   /skills/{id}/rebase                re-merge the fork onto the current base; 409 with conflict hunks if manual resolution is needed
GET    /sessions/{id}/tools               tools currently registered in a session, with layer + source
POST   /sessions/{id}/tools               replace the client's tool declarations (client(...) capabilities only; execute_on forced to client)
DELETE /sessions/{id}/tools/{name}
POST   /sessions/{id}/gates               replace the client's declarative gates (restrictive rules, unlocked, evaluated last)
GET    /tools?layer=...                   tool definitions available from a layer
POST   /tools                             write a tool definition to a layer (needs create-* permission)
DELETE /tools/{id}

GET    /prompt                            assembled system prompt for this token, provenance comments on (&budget=N to preview truncation)
GET    /prompt/fragments                  every fragment: id, layer, section, priority, locked, optional, condition state, shadows, refused
POST   /prompt/fragments                  write a fragment to a layer (needs create-* permission)
DELETE /prompt/fragments/{id}
GET    /classifier/rules                  effective rules for this token, with layer + id (+ ?explain=<tool> for the static path a call would take)
POST   /classifier/rules                  write a rule to a layer (needs create-* permission)
DELETE /classifier/rules/{id}
GET    /classifier/verdicts?session=&dry_run=   audit log of verdicts (read-audit for own sessions; read-audit-all for any); dry_run=true lists what dry-run rules would have stopped
POST   /sources/refresh                   ask every layer's sources to reload
POST   /sessions/{id}/assets              upload an attachment (owner-checked, size-capped) -> {asset_ref}
POST   /sessions/{id}/freeze              freeze/unfreeze a session (owner, or read-audit-all holder); frozen sessions deny every message
GET    /audit?session=&owner=&kind=&since=  audit records (own sessions with read-audit; any owner with read-audit-all)
GET    /config/effective                  the validated, defaulted configuration (manage-sources)
GET    /health
```

SSE is the default. WebSocket at `/sessions/{id}/ws` is an optional
transport adapter carrying the same events out and POST-equivalent
frames in (`08-walkthrough.md` §17e); the bearer token goes on the
upgrade request and a `token` frame refreshes it mid-socket.

## Streaming

- **Default: Server-Sent Events** on `/sessions/{id}/events`. Text/event-
  stream, one JSON object per event, `id:` set so clients can resume with
  `Last-Event-ID` after a disconnect. `assistant_delta` events are
  coalesced (~30 ms) on the wire and compacted out of the log once the
  turn completes; `assistant_message` always supersedes them.
- Every event carries `{ v, seq, type, session_id, turn_id, ts, level, payload }`.
  `level=debug` events are only sent when the stream asks for them.
- `assistant_delta` events stream model tokens; `assistant_message` is the
  final consolidated message for clients that do not render deltas. No
  server-side step sees assistant text before the client does
  (`07-turn-pipeline.md`, Streaming).
- `memory_created` and `memory_confirm` may arrive **after**
  `turn_complete`, because memory mining runs off the critical path.
  Clients should keep the stream open or fetch them on the next turn.
- A turn may start without a user message: `turn_started` with
  `initiated_by: job` follows a `tool_job_finished` when the session's
  delivery mode is `agent_turn` (`08-walkthrough.md` §13). Clients must
  render unsolicited assistant messages.
- Clients that do not want streaming can POST a message with
  `?wait=true` and get the final message in the response body. If the
  turn parks on an approval or a client tool call, the POST returns 202
  with the pending state instead, and the client must open the stream or
  poll `GET /sessions/{id}`.

## Event types

| Type                 | Direction | Payload                                     |
|----------------------|-----------|---------------------------------------------|
| user_message         | in        | text, attachments                           |
| tool_result          | in        | tool_call_id, result                        |
| approval_response    | in        | approval_id, allow/deny, remember (once | session), note |
| cancel               | in        | turn_id                                     |
| job_progress         | in        | job_id, pct, message (WebSocket frame; equals POST /sessions/{id}/jobs/{job_id}/progress) |
| token                | in        | bearer (WebSocket frame only; refreshes the connection's token, same rules as a new bearer on any request) |
| session_state        | out       | sent after every stream open: turn state, pending approvals, pending client calls (with args, deadline), active_team, chain layers, client tools |
| client_tools_registered | out    | names registered; shadows if any took an unlocked earlier-layer name |
| tool_shadowed        | out       | tool name, the earlier-layer tool it now overrides, by layer |
| stream_replaced      | out       | sent to the old connection when a new one opens for the same session |
| replay_gap           | out       | requested Last-Event-ID is older than the retained log; client should refetch state |
| chain_rebuilt        | out       | new token changed scopes/groups; layer list, active_team |
| token_expiring       | out       | expires_at                                  |
| auth_expired         | out       | stream closes; reopen with a fresh token    |
| turn_interrupted     | out       | turn_id, reason (service restart)           |
| session_ended        | out       | reason: client | idle                       |
| turn_started         | out       | turn_id, initiated_by: user | job (+ job_id) |
| context_ready        | out       | per-layer timings, counts of skills/tools/memories loaded |
| assistant_delta      | out       | text fragment                               |
| assistant_message    | out       | full text, citations: memory ids with layer, skill ids |
| tool_call_proposed   | out       | tool, args, execute_on, state=reviewing (first; audit pending) or state=execute (second, client tools only: run it now), static verdict if any |
| approval_requested   | out       | approval_id, tool call, asked_by (rule/classifier, layer, locked), reason, remember_options, timeout |
| approval_resolved    | out       | approval_id, decision, by, or timed_out/cancelled |
| tool_call_denied     | out       | tool_call_id, reason                        |
| audit_verdict        | out       | tool_call_id, source (rule id or model), decision, p_requested (debug-level) |
| audit_dry_run        | out       | tool_call_id, would_have, by (when a layer's rules run in dry-run mode) |
| prompt_truncated     | out       | dropped fragment ids, shrunk fragments (only when over budget) |
| memory_recalled      | out       | ids and layers of memories placed in the prompt (debug-level) |
| tool_call_started    | out       | tool_call_id                                |
| tool_call_finished   | out       | tool_call_id, ok, kind on error, duration, truncated, full_ref, background?, job_id? |
| tool_hidden          | out       | tool name, layer, excess capabilities (debug-level; when on_excess=hide) |
| tool_job_started     | out       | job_id, tool, eta_s                          |
| tool_progress        | out       | job_id, pct, message, updated_at             |
| tool_job_finished    | out       | job_id, state, summary; followed by an agent-initiated turn or injected on next message |
| memory_created       | out       | memory id, kind, title, layer, suggested_audience?, supersedes? |
| memory_confirm       | out       | approval_id (kind=memory), candidate, team, options, timeout, on_timeout=keep_personal |
| memory_decision      | in        | approval_id, accept_team / keep_personal / discard (same endpoint as tool approvals) |
| memory_decision_resolved | out   | approval_id, decision or timed_out                |
| memory_updated       | out       | memory id, layer (dedupe hit updated an existing memory) |
| memory_discarded     | out       | candidate hash, reason (debug-level; off by default) |
| skill_suggested      | out       | skill id, matched trigger (debug-level)     |
| skill_loaded         | out       | skill id, name, layer, version, tools registered, auto? |
| skill_unloaded       | out       | skill id, tools removed, reason: model | client | evicted |
| skill_outdated       | out       | skill id, loaded version, current version   |
| skill_shadowed       | out       | skill id, the id it overrides, stale, base_version, fork_base_version |
| shadow_refused       | out       | accumulator, name, attempted_by layer, locked_by layer |
| skill_forked         | out       | new skill id, from, by (service-level, not per session) |
| turn_complete        | out       | turn_id, usage, outcome (values listed after this table), cancelled? |
| turn_flagged         | out       | turn_id, reason, layer (post-hoc flag from an on_turn_end step) |
| error                | out       | code, message, recoverable                  |
| context_exhausted    | out       | turn_id, tokens, window; session ends and a successor should be created with resume_from |
| turn_denied          | out       | turn_id, reason (an on_message gate stopped the turn before the model) |
| source_item_invalid  | out       | source, item id, reason (debug-level; a malformed skill/rule/fragment was skipped) |

`turn_complete.outcome` is one of `ok | responded | denied | max_tokens |
refusal | error | cancelled | exhausted`; `assistant_message` carries
`truncated` or `refusal` when the outcome says so.

## Error catalogue

Used by the `error` event (`code`) and as HTTP error bodies (`{ error: code, ... }`).

| Code                        | Where              | Meaning / client action                                            |
|-----------------------------|--------------------|--------------------------------------------------------------------|
| `not_owner`                 | HTTP 403           | token subject does not own the session/job/asset                   |
| `scope_missing`             | HTTP 403           | token lacks the scope named in `scope`                             |
| `turn_in_progress`          | HTTP 409           | a turn is running; cancel or wait for `turn_complete`              |
| `context_exhausted`         | HTTP 409 / event   | conversation no longer fits; create a successor with `resume_with` |
| `approval_conflict`         | HTTP 409           | approval already answered, expired, or unknown                     |
| `no_pending_client_call`    | HTTP 409           | tool result for a call that is not awaiting the client             |
| `exists` / `locked`         | HTTP 409           | skill/tool/fragment name taken, or base is locked (fork)           |
| `invalid_args`              | tool result        | model's arguments failed schema validation                         |
| `capability`                | tool result        | tool exceeds layer ceiling, path/host not permitted, builtin ref disallowed |
| `timeout` / `rate_limited` / `cancelled` | tool result | as named                                                     |
| `job_limit`                 | tool result        | `jobs.max_per_user` or `jobs.max_total` exceeded; the model may retry later |
| `store_unavailable`         | event              | session store or event log write failed; turn failed               |
| `prompt_budget_exceeded`    | event + HTTP 500   | locked fragments alone exceed the model window; a misconfiguration |
| `source_unavailable`        | event + HTTP 503   | a `required` source is down; turn failed                           |
| `model_error`               | event              | provider returned a non-retryable error; turn failed               |
| `model_unavailable`         | event              | provider unavailable after retries; turn failed                    |
| `config_invalid`            | startup            | see the validation rules in `02-layering-and-composition.md`      |

## Sessions

- **One client, one session.** A session is created by a client and owned
  by it. There is no fan-out of one session's events to several clients.
- **Active team.** A session may carry `active_team`, set by the client
  at creation or changed later. It must be one of the token's groups. It
  is the main signal the memory routing steps use to send team-relevant
  memories to that team's store; without it, memories stay personal.
- **What is shared and what is not.** Sessions are private to their
  client. Skills, memories, and the classifier are shared service-level
  resources that every session draws on (subject to the session's
  token's scopes and groups).

  | Resource     | Shared across sessions | Owned by            |
  |--------------|------------------------|---------------------|
  | Session      | no                     | one client          |
  | Memories     | yes (layer-filtered)   | each layer's memory store |
  | Skills       | yes (layer-filtered)   | each layer's skill store  |
  | Classifier   | yes                    | the service         |
  | System prompt| assembled per session from shared sources             |
- A session owns a conversation, the validated `AccessToken` that created
  it, a durable seq-numbered event log, its pending approvals, and a
  single event stream. Only one open stream per session is expected; a
  reconnect replaces the previous stream (the old one receives
  `stream_replaced`) and resumes with `Last-Event-ID`, followed by a
  `session_state` snapshot. Lifecycle, token refresh, restart, and
  end-of-session behaviour: `08-walkthrough.md` §11.
- Approval requests therefore have exactly one responder: the owning
  client.
- Continuity between clients comes from the memory layers, not from
  session sharing. A user switching from CLI to web starts a new session
  and relies on memories (and optionally a session-summary memory) to
  carry context.
- Session state lives in a pluggable `SessionStore` (memory for dev,
  sqlite single-node, postgres/redis multi-node). The chain is never
  persisted; it is rebuilt from token and config.

## Auth

- Authentication is OAuth2-style: clients log in with an identity provider
  and send `Authorization: Bearer <token>` on every request. Local
  single-user mode uses a built-in dev IdP so the path is the same.
- Authorization is by OAuth2 permission scopes on the token, e.g.
  `use-global-skill`, `create-team-skill`, `use-personal-memory`,
  `create-personal-memory`.
  Full list and enforcement points in `04-auth-and-permissions.md`.
- The API validates the bearer token into an `AccessToken` object
  (`sub`, `scopes`, `groups`) and passes that object down. It never
  interprets identity beyond validation.
- A session is bound to the subject that created it. Requests on that
  session with a different subject are rejected.
- Endpoints and the permissions they require (sketch):

  | Endpoint                          | Requires                          |
  |-----------------------------------|-----------------------------------|
  | `GET /skills`, `GET /skills/{id}` | `use-{global,team,personal}-skill` (each filters its layer) |
  | `POST/PUT/DELETE /skills`         | `create-{global,team,personal}-skill` for the target layer; team writes also need the team in the token's `groups` claim |
  | `POST /skills/{id}/fork`, `/rebase` | `use-<base layer>-skill` plus `create-<target layer>-skill`; refused if the base is locked |
  | `GET /memories`                   | `use-{team,personal}-memory` (each filters its layer); `use-global-memory` for the global layer |
  | `POST/DELETE /memories`, `/promote` | `create-{team,personal}-memory` for the target layer |
  | `GET /prompt`, `/prompt/fragments` | any valid token (global fragments always; other layers by `use-<layer>-prompt`, proposed) |
  | `POST/DELETE /prompt/fragments`   | `create-<layer>-prompt` (proposed)  |
  | `GET /classifier/rules`           | any valid token (global rules always; other layers by `use-<layer>-classifier-rule`, proposed) |
  | `POST/DELETE /classifier/rules`   | `create-<layer>-classifier-rule` (proposed) |
  | `GET /classifier/verdicts`        | `read-audit` (own sessions) or `read-audit-all` |
  | `GET /tools`                      | `use-<layer>-tool` (proposed)     |
  | `POST/DELETE /tools`              | `create-<layer>-tool` (proposed)  |
  | `/sessions/*` (all)               | session owner (token `sub` == session owner); creation needs only a valid token |
  | `/sessions/{id}/approvals/*`      | session owner                     |
  | `/sessions/{id}/tools`, `/gates`, `/assets`, `/skills/*` | session owner |
  | `/jobs/*`                         | job owner                         |
  | `POST /sources/refresh`, `GET /config/effective` | `manage-sources` (proposed) |
  | `GET /audit`                      | `read-audit` (own sessions) or `read-audit-all` |
  | `POST /sessions/{id}/freeze`      | owner, or `read-audit-all`        |
  | `GET /health`                     | none                              |

## Tools executed by the client

Some tools only make sense on the client (open a file in the user's editor,
read the clipboard). The API supports this by emitting
`tool_call_proposed` with `execute_on: client` **twice**: first with
`state=reviewing` while the audit runs, then, only if the call was
allowed, with `state=execute`. A client must execute only on
`state=execute` and then POST a `tool_result`. Server-side tools run in
the service and only emit started/finished events.

Skill-defined tools may be either kind. A client-side skill tool is a
contract: the skill declares the name and input schema, and the client
must know how to fulfil it (or return an error saying it cannot). The
`/sessions/{id}/tools` endpoint lets a client see which tools are active
and which of them it is expected to execute.
