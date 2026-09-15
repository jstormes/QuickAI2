# 0011 – Resolutions from the pre-audit review (important findings)

**Status:** accepted (2026-09-15). Follows 0010.

## Context

The coverage-gap review listed 28 "important" items: things an
implementer would have to guess. Each is now specified; the security
items are gathered in the new `design/09-threat-model.md`.

## Decisions

- **Step protocol.** Every step declares `kind: gating | contributing`;
  contributing steps have `fetch`/`apply`; fail mode follows kind;
  `required: true` fails the turn. Routing steps are contributing and
  push a failed write to `unmined_candidates`.
- **All steps named and registered**: retag, auto_load, audit_log are
  configurable; LoadedSkillsStep, the service gates, and the session
  layer's steps are implicit.
- **Turn context and session state** are fully declared in the object
  model; audit state is one per-call record; the persisted session
  field list is complete, and memory approvals survive session end.
- **Writer ports** for tools, prompts, and rules; git writers commit per
  write with the subject as author and return 409 on conflict.
- **Tool origin** is a field; rules match on it; builtin impls are only
  allowed for tools from a trusted tool source.
- **Dedupe is model-free**: hash or store similarity for duplicates;
  contradictions only when the miner says so.
- **Gates on messages, second message during a turn, and model
  terminal conditions** (max_tokens, refusal, fatal error) each have a
  defined outcome; `turn_complete.outcome` names it.
- **Attachments** are owner-bound assets rendered as image/document
  blocks, counted against the context check.
- **Error catalogue** in the API note; per-item validation of source
  content with a debug event; **configuration reference with defaults**
  and startup validation rules in the layering note.
- **Every script impl is sandboxed**, including the user's own; the
  policy differs by layer, the mechanism does not.
- **Cross-team provenance** blocks auto-accept of a team memory.
- **Asset store** port with opaque owner-bound refs; **skill resource
  paths** confined to the skill folder.
- **Audit records** have one schema, retention, and `read-audit` /
  `read-audit-all` scopes; sessions can be frozen.
- **Identifier formats** are tabulated in the core note.
- **Multi-node** is sketched as per-session leases with forwarded
  publishes; single-node remains the v1 target.
- **Threat model** written: principals, assets, mitigations, prompt
  injection as mitigated-not-eliminated, supply chain as accepted risk.

## Consequences

- The starting permission set is now fourteen (adds the two audit
  scopes).
- The API gains asset upload, freeze, audit read, and effective-config
  endpoints, and two events.
- Remaining minor findings (vocabulary drift, object-model naming) are
  not covered here.
