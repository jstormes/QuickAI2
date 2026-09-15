# Vision: a loosely coupled AI agent framework

## One-paragraph summary

A framework for building AI agents where the *core* (conversation loop, tool
dispatch, context assembly) is separated by explicit interfaces from
everything that varies between deployments: where memories live, where skills
live, where system prompts come from, who the current user is, and what the
front end looks like. Every one of those concerns can have **multiple
backends at once**, composed in layers (global, team, user, session), each
possibly owned and maintained by different people or systems.

## Goals

1. **Memories** – organized and semantically searchable, but the framework
   never assumes a specific store. A memory backend could be flat files,
   SQLite, Postgres+pgvector, a hosted vector DB, or an HTTP service.
2. **Skills** – reusable, discoverable units of instruction/capability. Same
   rule: storage is behind an interface, and several stores can be active.
3. **Classifier support agent** – a secondary, cheaper agent that audits the
   primary agent. First job: decide whether a tool call was genuinely
   requested/warranted by the user. Second job: generate memories in the
   background from the conversation.
4. **Arbitrary front end** – CLI, web, desktop/mobile app, or all three
   simultaneously against the same core.
5. **Multi-source user association** – which memories and skills belong to
   "this user" is decided by the OAuth2 token's claims and scopes, not
   hard-wired. Global shared skills, team skills, and user-private skills
   can live in different repos maintained by different people. Same for
   memories. Layers: global, team, user, session.
6. **Multi-source system prompts** – the system prompt is assembled from
   several sources (framework defaults, org policy, project, user
   preferences, session), again pluggable and layered.
7. **Presents itself as a streaming web API** – the framework runs as a
   service and exposes an HTTP API with streaming responses. Every front
   end (CLI, web, app) is a client of that API. See
   `03-web-api.md` and `decisions/0001-streaming-web-api.md`.

## Non-goals (for now)

- Being a hosted multi-tenant product. It is a service you run (locally or
  on a server), not a SaaS offering, though nothing should prevent that
  later.
- Picking a single LLM vendor. Model access should be behind an adapter too,
  though that is a secondary concern to the ones above.
- Solving multi-agent orchestration in general. The classifier is a fixed,
  well-defined support role, not a general agent-to-agent protocol.

## Guiding principles

- **Ports and adapters.** The core defines interfaces ("ports"). Storage,
  identity, model, and UI are adapters. The core has zero imports from any
  adapter.
- **Composition over configuration.** Multiple providers of the same kind are
  combined by the chain of layers and steps, which handles precedence, merging, and conflict
  rules. Adding a source should never require changing the core.
- **Read paths and write paths are separate.** Most sources are read-only
  from the agent's point of view (e.g. a global skills repo). Only some
  accept writes (e.g. the user's memory store). The interface makes this
  explicit.
- **Everything is addressable.** Memories, skills, and prompt fragments each
  have a stable identity including which source they came from, so they can
  be cited, updated, or removed later.
- **The core never blocks on background work.** Memory creation and
  classification run off the critical path of the user's turn.
- **Sessions are private; skills, memories, and the classifier are
  shared.** A session belongs to one client. Everything a session learns
  that is worth keeping goes into the shared memory stores, where any
  other session whose token carries the right scopes can find it.
