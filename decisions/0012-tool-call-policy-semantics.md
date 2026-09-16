# 0012 – Tool-call policy semantics

**Status:** accepted (2026-09-15)

## Context

Static classifier rules come from every layer. Earlier drafts evaluated
them "locked first" as one sorted list, which per-layer steps cannot
implement and which let an earlier unlocked `allow` silence a later
`require_approval`.

## Decision

- Each layer's `StaticRulesStep` evaluates only that layer's rules, in
  chain order.
- `deny` and `require_approval` are **restrictive**: any layer may add
  one and no layer can remove another's.
- `allow` is **widening**: it pre-empts the model audit but yields to any
  restriction from any layer, unless it is `locked`, in which case it
  shields the call from later layers' restrictions.
- Calls at or above `classifier.always_audit_risk_gte` (default `high`)
  are model-audited even when an unlocked `allow` matched; only a locked
  `allow` skips the audit there.
- The model-based audit is the last `on_tool_call` step and runs only for
  calls no rule decided.

## Consequences

- Peer team layers never conflict; no verdict merge strategy exists.
- **Accepted risk:** an unlocked team or user `allow` switches off the
  model audit for a global tool below the always-audit threshold for that
  layer's members. Bounded by global locked restrictions and the audit
  log. See `decisions/0015`.
