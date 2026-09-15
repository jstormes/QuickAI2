# Threat model

What the layers, locking, scopes, sandbox, and audit are defending
against, and what they are not. Every mechanism below points at the note
that specifies it.

## Principals and trust

| Principal | Trust | Why |
|---|---|---|
| Service operator / config | full | owns the process, secrets, users_root, ceilings, and the global layer's sources |
| Identity provider | full for identity | the only source of `sub`, `scopes`, `groups` (`04-auth-and-permissions.md`) |
| Global-layer maintainers | full for policy | global runs first, may lock, and its rules and fragments apply to every token. **A compromised global source is a full compromise**; protect it like production config (review, signed commits, restricted push) |
| Team-layer maintainers | partial | scripts sandboxed, risk escalated, capability ceilings, cannot lock unless the store allows; can affect only their members |
| The user (own layer) | partial | may shadow unlocked values and add restrictions; cannot override locked; user scripts are sandboxed too |
| The client program | low | client tools run only on the client, client-only capabilities, results untrusted, gates declarative only (`08-walkthrough.md` §14) |
| The model | untrusted proposer | every tool call is gated by rules and the classifier; text is never gated (§7 pipeline) |
| The model provider | assumed honest, treated as outside the boundary | a non-goal (`00-vision.md`); nothing secret is sent that the model does not need |
| Content: tool results, client results, skill text, memories, attachments | untrusted data | see "prompt injection" below |

## Assets

Personal and team memories; skills and their tools; secrets; the raw
token at rest; transcripts and event logs; the audit log; the service's
own filesystem and network.

## Threats and mitigations

| Threat | Mitigation | Where |
|---|---|---|
| Tool call the user did not ask for | static rules (restrictions accumulate), model audit, approvals, service scope gates | `08` §8, §5 |
| Privilege escalation via a fork | fork tools re-labelled to the fork's layer and clipped to its ceiling | `08` §7a |
| Policy bypass by same-name shadowing | `locked` on skills, tools, fragments; locked allow shields | `07`, `08` §8b |
| Skill aliasing a builtin under another name | builtin impls only for `origin: source` in allowed layers | `08` §12c |
| Script escape or egress | sandbox for every script impl: namespaces, read-only binds, egress proxy allowlist, limits; user-layer scripts sandboxed too | `08` §12d |
| Path traversal via skill resources | resource paths normalised and confined to the skill folder | `08` §15g |
| Secret leakage to model, memory, or logs | secrets resolved in runners or attached by the proxy; results redacted; locked global drop rule for credential memories | `08` §12f, §6c |
| Team data leaking to another team via mining | private by default; team write needs a positive signal; candidates whose provenance includes another team need explicit confirmation | `08` §6d |
| Cross-user access to a session, job, asset, or personal layer | owner checks on every session/job/asset endpoint; user-layer paths keyed by subject; asset refs opaque and owner-bound | `08` §11, §13, `01` §9 |
| Resource exhaustion | rate limits, concurrency slots, rlimits, result caps, job limits, per-subject message rate | `08` §12h, §13g |
| Approval fatigue / bypass | remembered approvals are session-scoped and cannot override a locked asker | `08` §5e |
| Tampering with the audit trail | append-only records, retention, read gated by `read-audit(-all)` | `04` |
| Lost or replayed events | durable log before delivery, seq-numbered replay | `08` §17 |

## Prompt injection: mitigated by framing and audit, not eliminated

Skill instructions, prompt fragments, recalled memories, tool results,
client results, and attachments all reach the model as text. The design
does **not** claim the model cannot be steered by them. It does three
things:

1. **Provenance framing.** Each is rendered inside a tagged block that
   names its origin and layer (`<memories>`, `<skills>`, tool results
   with their tool name), and the global prompt baseline instructs the
   model that such blocks are data, not instructions. This is the same
   framing the classifier receives.
2. **Actions are gated regardless of what the text says.** A tool call
   induced by injected text still passes every static rule, the scope
   gates, the model audit (which sees the provenance), and approvals.
   Injection can waste a turn; it cannot by itself run a high-risk tool.
3. **Restrictions cannot be removed by content.** Nothing in a skill,
   memory, or result can unlock a locked rule or add an `allow`.

Residual risk, accepted for v1: injected text can influence the
assistant's *words* and low-risk tool choices, and can attempt to
exfiltrate data through an allowed low-risk tool. Deployments that
cannot accept that must lower the `allow` thresholds or require approval
for tools that can send data out.

## Supply chain

Skill and rule sources are git repositories or HTTP services maintained
outside the framework. The framework pins loaded skills per session and
records versions in the audit log, but does not verify signatures or
enforce review. Treat the global repo as production config; team repos
as team-owned code. Signature verification is an adapter feature a
deployment may add (open in `IDEAS.md`).

## Out of scope

Compromise of the host, the IdP, or the model provider; side channels
through timing or token counts; denial of service against the IdP.
