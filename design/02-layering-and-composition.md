# Layering and composition

How multiple sources of the same thing coexist.

## Layers

Decided layers, lowest precedence first (`decisions/0005-team-layer.md`):

| Layer     | Owner                      | Holds                                          |
|-----------|----------------------------|------------------------------------------------|
| global    | platform/infra team        | shared skills repo, shared memory service, org policy prompt fragments |
| team      | a team's maintainers       | team skills repo, team memory store, team prompt fragments |
| user      | the individual             | personal skills, personal memories, preferences|
| session   | runtime                    | ad-hoc instructions, temporary memories        |

A user may belong to **several teams**. The context resolver expands the
`team` layer into one sub-layer per team the token grants, ordered by the
IdP's claim order (or alphabetically if unordered). Same-name skills
across two of a user's teams are a conflict; see IDEAS.md.

Other candidate layers (framework defaults, project) are not decided. The
core only requires that layers be totally ordered, so adding one is a
configuration and permission change, not a core change. **The layer list
is the developer's to set**: a solo deployment may have only user and
session; a large one may have several shared layers. See
`05-design-patterns.md` for how the composites and chains are built from
that list.

### Team identity

- Team membership comes from the token: a `teams` (or `groups`) claim
  listing team ids. The IdP is the source of truth.
- Permissions for the team layer are *not* per-team strings. A token
  holds `use-team-skill` once, and the `teams` claim says which teams it
  applies to. This keeps the permission set small as teams multiply.
- Team stores are configured per team id; a team without a configured
  store simply contributes nothing.

## Composition rules

All of these are consequences of chain order plus locking
(`07-turn-pipeline.md`): handlers run global → team → user → session,
later handlers overwrite earlier unlocked values, locked values are
final.

- **Skills:** same `name` in a higher layer shadows the lower one. Both
  remain addressable by full id so the shadowed one can still be loaded
  explicitly. A skill marked `locked` in a lower (more trusted) layer is
  exempt: it cannot be shadowed. Users create overrides by forking
  (`01-core-abstractions.md` §3).
- **Memories:** never shadowed; all layers contribute to search. Ranking may
  boost nearer layers (user > team > global) since they are more
  specific. Agent-driven writes target only personal or team, chosen by
  the memory routing policy (`01-core-abstractions.md` §2) with personal
  as the fallback; global memory is admin-curated and read-only here.
- **Prompt fragments:** grouped by section, ordered by (layer, priority).
  A fragment can be marked `locked` so higher layers cannot override it
  (used for org policy).
- **Permissions gate layers.** A (store, layer) pair participates in a
  request only if the token carries the matching permission
  (`04-auth-and-permissions.md`). Missing permission means the layer is
  silently absent, not an error.

## Configuration shape

The layer list *is* the handler chain. The full sketch, with every
resource nested under its layer, is in `07-turn-pipeline.md`
(Configuration view). Service-level settings sit beside it:

```yaml
layers: [...]                      # see 07-turn-pipeline.md

classifier:
  engine: { adapter: model, model: <small fast model> }   # or rules_only, or http
  tool_audit: { enabled: true }
  memory_mining: { enabled: true }

tools:
  runners:
    script: { sandbox: required_below_layer: user }
    http:   { timeout_s: 30 }

api:
  listen: 127.0.0.1:8080
  streaming: sse            # sse | websocket | both
  auth:
    adapter: oauth2_jwt     # or dev_idp for local single-user mode
    issuer: https://idp.example.internal
    audience: basic-agent
    permission_claim: scope # which token claim carries permission scopes
```

## Failure behavior

- A source that is unreachable should degrade, not fail the turn. The
  handler logs it and continues with the remaining sources. A `required`
  flag can opt a source into hard failure (e.g. org policy prompt).
- Caching per source (with TTL) is an adapter concern but the handler
  should expose "refresh" so a front end can offer a reload command.
