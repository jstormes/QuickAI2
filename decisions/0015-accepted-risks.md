# 0015 – Accepted risks (v1)

**Status:** accepted (2026-09-15)

## Context

`design/09-threat-model.md` names risks the design mitigates but does not
eliminate. An audit needs them signed off, not discovered.

## Decision

The following are accepted for v1, each with its bound:

1. **Prompt injection** through skill text, fragments, memories, tool
   results, client results, attachments, and job results. Mitigated by
   provenance framing and by gating actions rather than text; not
   eliminated. Bound: every action passes rules, gates, audit, approvals.
2. **Unsigned supply chain** for git and HTTP sources. Bound: per-session
   version pinning, versions in the audit log, global repo treated as
   production config. Signing is an adapter feature (IDEAS).
3. **Classifier failure default** `ask_if_risk_gte_medium_else_allow`:
   on classifier outage, low-risk undecided calls run, logged. Bound:
   static rules and scope gates still apply; the default is configurable.
4. **Team `allow` audit bypass** below `classifier.always_audit_risk_gte`
   (`decisions/0012`).
5. **Classifier steering** by advisory fragments from team and user
   layers. Bound: locked global restrictions, the always-audit threshold,
   audit records of the fragments in force.
6. **Proxy confused deputy**: a sandboxed script may reach any allowed
   host with the attached secret. Bound: narrow host patterns.
7. **Shared sandbox uid**: inter-user isolation rests on namespaces and
   per-call workspaces.

## Consequences

Revisit each when the corresponding IDEAS item is decided or when a
deployment's policy cannot accept the bound.
