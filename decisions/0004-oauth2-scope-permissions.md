# 0004 – OAuth2-style authentication with scope-based permissions

**Status:** accepted (2026-09-14)

## Context

The framework is a web API used by CLI, web, and native clients, backed
by skill and memory stores that different people and systems maintain.
It needs a way to know who is calling and what they may touch, without
teaching storage adapters anything about auth.

## Decision

- Clients authenticate with an OAuth2-style login against an identity
  provider; the API validates bearer tokens. The IdP is an adapter, with a
  dev IdP for local single-user mode.
- Authorization is scope based: permission strings on the token gate
  access. Starting set:
  `create-global-skill`, `create-personal-skill`, `use-global-skill`,
  `use-personal-skill`, `use-personal-memory`, `create-personal-memory`.
  (The last two were originally one permission, `allowed-personal-memory`,
  split so recall and learning can be granted separately.)
- The validated access token object (`sub`, `scopes`, `groups`) is passed
  through to layer handlers and their stores, which filter by it directly. There is
  no intermediate identity or context object.

## Consequences

- Adding a layer (team, project) means adding matching permissions; the
  naming pattern should be fixed early (see IDEAS.md).
- Background memory mining needs a token that acts on the user's behalf
  after the request ends.
- Earlier drafts had a resolved `Scope` / `LayerContext` object and a
  resolver. Removed (2026-09-14): it restated the token's claims and added
  vocabulary the team does not need. Which layers apply is a lookup from
  scopes, documented as a table in `04-auth-and-permissions.md`.
- Who holds `create-global-skill` is decided in the IdP, which is where
  "maintained by different people" is expressed.
