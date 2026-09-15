# Authentication, authorization, and permissions

## Authentication: OAuth2-style login

- Clients obtain an access token through an OAuth2-style flow against an
  identity provider (IdP). The framework does not implement an IdP; it
  validates tokens issued by one (JWT with JWKS, or introspection).
- The IdP is an adapter. Local single-user mode may use a built-in
  "dev IdP" that issues a fixed token, so the code path is identical.
- Flows expected per front end:
  - CLI: device authorization flow (or auth-code with a loopback redirect).
  - Web: authorization code + PKCE.
  - Native app: authorization code + PKCE with a custom-scheme or loopback
    redirect.
- Every API request carries `Authorization: Bearer <token>`. Session
  creation binds the session to the token's subject; later requests on
  that session must present a token for the same subject.

## Authorization: permission scopes

Authorization is **scope based** in the OAuth2 sense: the token carries a
set of permission strings, and the API and stores check them.

### Terminology

OAuth2 calls permissions "scopes". To avoid confusion with the framework's
*layer* concept, prose in this design says **permission** for an OAuth2
scope string such as `use-global-skill`; code calls the set `scopes`.

There is no separate identity object. The validated **access token**
(`sub`, `scope`, `groups` claims) is passed through the system as-is.
Which layers a request touches is decided by looking at the token's
scopes against the table below.

### Scope-to-layer table

| Layer  | Read/use scope           | Write scope                  | Extra condition            |
|--------|--------------------------|------------------------------|----------------------------|
| global | `use-global-<resource>`  | `create-global-<resource>`   | none                       |
| team   | `use-team-<resource>`    | `create-team-<resource>`     | team id in `groups` claim  |
| user   | `use-personal-<resource>`| `create-personal-<resource>` | store keyed on `sub`       |
| session| always                   | always                       | private to the session     |

`<resource>` is `skill`, `memory`, and (proposed) `prompt`, `tool`,
`classifier-rule`. The global layer is always in every chain: its prompt
fragments, classifier rules, and gates are included regardless of scope
as a policy baseline; only its skills, tools, and memories need a
`use-global-*` scope.

### Starting permission set

| Permission                | Grants                                                   |
|---------------------------|----------------------------------------------------------|
| `use-global-skill`        | discover and load skills from the global layer           |
| `use-personal-skill`      | discover and load skills from the user's own layer       |
| `create-global-skill`     | write/update/delete skills in the global layer           |
| `create-personal-skill`   | write/update/delete skills in the user's own layer       |
| `use-team-skill`          | discover and load skills from the user's teams' layers   |
| `create-team-skill`       | write/update/delete skills in the user's teams' layers   |
| `use-personal-memory`     | search and read memories in the user's own layer         |
| `create-personal-memory`  | write/update/delete memories in the user's own layer     |
| `use-team-memory`         | search and read memories in the user's teams' layers     |
| `create-team-memory`      | write/update/delete memories in the user's teams' layers |
| `run-server-tools`        | execute server-side tools at all (checked by the service scope gate before any rule) |
| `run-high-risk-tools`     | execute tools whose *effective* risk is high (still subject to rules and audit) |

Notes on the starting set:

- `use-*` and `create-*` are deliberately separate: most users can use
  global skills, few can publish them.
- `create-personal-skill` also lets a user **override** a global or team
  skill for themselves by forking it into their layer (needs the matching
  `use-*` scope to read the base). This never affects other users.
- There is deliberately **no `create-global-memory`**. Memories are
  personal or team; a global memory store, if configured, is curated
  outside the agent (`decisions/0007-memory-routing.md`).
- Team permissions apply to **every team in the token's `teams` claim**.
  Which teams that is comes from the IdP, not from the permission string.
  Finer grain (publish to team A but only use team B) is not supported in
  the starting set; see IDEAS.md.
- `create-*` implies `use-*` for the same layer? Proposal: **no implicit
  grants**; the IdP/admin assigns both explicitly. Simpler to reason
  about, and the token is the single source of truth.
- `use-personal-memory` gates recall from the personal layer;
  `create-personal-memory` gates writes to it, including background memory
  mining. A session with neither runs memoryless at the user layer (global
  memories, if any, are governed separately). A session with `use` but not
  `create` can recall but never learns; useful for read-only or audited
  contexts.

### Likely additions (not yet decided)

| Permission                | Grants                                                   |
|---------------------------|----------------------------------------------------------|
| `use-global-memory`       | search admin-curated global memories (read-only for agents) |
| `bypass-tool-audit`       | skip the classifier for this subject (service accounts)  |
| `manage-sources`          | call `/sources/refresh` and similar admin endpoints      |
| `use-<layer>-prompt`      | include that layer's prompt fragments (global always on) |
| `create-<layer>-prompt`   | write prompt fragments to that layer via the API         |
| `use-<layer>-classifier-rule`    | apply that layer's classifier rules (global always on) |
| `create-<layer>-classifier-rule` | write classifier rules to that layer via the API |
| `use-<layer>-tool`        | include that layer's tools in the session's registry     |
| `create-<layer>-tool`     | write tool definitions to that layer via the API         |

Pattern: `<verb>-<layer>-<resource>` with verbs `use`, `create`, and
later maybe `manage`. Ten of the twelve starting permissions follow it;
`run-server-tools` and `run-high-risk-tools` are execution gates rather
than layer grants and use the `run-` verb deliberately.

## Where permissions are enforced

```
request --> API layer: token valid? subject matches session?
        --> AccessToken object (sub, scopes, groups) passed down
        --> Layer chain: global is always present (policy baseline); a team
            or user layer is present only if the token has a matching use
            scope (and, for team, the team id in groups); inside any layer a
            step runs only with the scope for its resource; writes require
            the matching create scope
        --> Tool dispatcher: execution permissions checked per tool
        --> Classifier: may consult permissions (e.g. bypass-tool-audit)
```

Key point: **stores see the validated token object, never the raw
bearer header, and never validate anything themselves.** They only read
`sub`, `scopes`, and `groups`. This keeps storage adapters ignorant of
how authentication works while still letting them filter by caller.

## Multiple backends and ownership

Permissions say *which layer* a subject may touch. Which physical store
backs that layer is configuration (`02-layering-and-composition.md`).
So "global skills are maintained by team X in repo Y" is expressed as: the
global skill layer is backed by repo Y, and only subjects holding
`create-global-skill` (team X) can write to it through the API. Team X may
also bypass the API entirely and push to the repo directly; the store
adapter picks that up on refresh.

The team layer works the same way one level down: team T's skill store is
whatever the config maps `team_id = T` to, and members of T holding
`create-team-skill` can write to it. Two teams can use completely
different adapters (one a git repo, one an HTTP service).

## Open items

- Token refresh on long-lived sessions (CLI left open overnight).
- Service accounts for background jobs (memory mining runs *as* whom? The
  session's subject, with a delegated/on-behalf-of token, seems right).
- Whether permission checks on memory search should filter results or
  fail the request. Proposal: filter silently by excluding layers.
- Admin UI/CLI for assigning permissions lives in the IdP, not here.
