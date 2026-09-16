# 0014 – Approval memory, token at rest, and mining identity

**Status:** accepted (2026-09-15)

## Context

Three security-relevant behaviours lived only in walkthrough pseudocode.

## Decision

- **Approval memory.** A user may remember an approval for `once` or for
  the `session`, never beyond. The asker (rule or classifier) computes the
  offered options; a locked asker never offers `session`. A remembered
  approval is an unlocked session-layer `allow`, so it can never override
  a locked asker or remove a restriction. `remember_match: exact | tool`
  is set by the asker's rule.
- **Token at rest.** `sessions.token_at_rest: encrypted | never`. With
  `encrypted`, the raw token is stored under a key from the
  `SecretsProvider` and decrypted only by the service for post-restart
  work as the user; `never` disables that work.
- **Mining identity.** Memory mining runs as the user with the session's
  token; a turn whose token is near expiry is queued and mined with the
  next fresh token; leftovers are mined at session end if the token is
  still valid, else logged as dropped.

## Consequences

- Nothing is ever remembered across sessions through approvals; a
  permanent allow is a deliberate rule edit.
- Deployments that cannot hold the key lose post-restart mining and say
  so in config.
