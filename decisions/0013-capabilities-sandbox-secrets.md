# 0013 – Capabilities, ceilings, sandbox, and secrets

**Status:** accepted (2026-09-15)

## Context

Skills ship tools, tools need filesystem, network, subprocess, and
secrets, and layers are maintained by people of different trust.

## Decision

- Tools declare `requires: [Capability]`; each layer has a ceiling; the
  grant is clipped to the ceiling when the tool enters `ctx.tools`
  (`tools.on_excess: hide | degrade`) and re-checked at run time.
  Path and host arguments are recognised only by `x-arg-kind`
  annotations on the input schema.
- Every script implementation is sandboxed regardless of layer
  (`subprocess_restricted` in v1; container and wasm as adapters). The
  sandbox *policy* is per layer (`tools.sandbox.policy_by_layer`:
  `strict` for global and team, `relaxed` for user); the mechanism is
  not optional.
- Builtin implementations are allowed only for tools from a trusted tool
  source (`tools.builtin_refs_allowed_from`, default global).
- Secrets are resolved by name inside runners or attached to outbound
  requests by the egress proxy; they never enter the turn context, the
  event log, results, or memory candidates. Results are redacted with the
  provider's registered patterns.

## Consequences

- Fork tools are re-labelled to the fork's layer and clipped again.
- The egress proxy is a trusted component holding every script-usable
  secret. **How it attaches secrets to HTTPS requests is an open question
  (IDEAS: proxy vs http-only team tools)**; this record fixes the
  contract, not the mechanism.
- A tool with unannotated string arguments cannot be path-restricted.
