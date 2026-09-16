# Design notes: reading order

Start at the top; `02` and `03` are reference material to consult as
needed. `GLOSSARY.md` fixes the vocabulary.

| Order | Note | What it is |
|---|---|---|
| 1 | `00-vision.md` | Goals, non-goals, guiding principles |
| 1b | `10-requirements.md` | The goals as testable statements; non-functional rows (targets are the owner's to fill) |
| 2 | `07-turn-pipeline.md` | The spine: outer chain of layers, hook points, inner chains of steps, the turn context |
| 3 | `01-core-abstractions.md` | The ports: token, memory, skill, prompt, tool, classifier, writers, asset store, identifiers, model client |
| 4 | `04-auth-and-permissions.md` | OAuth2 login, scope-to-layer table, starting permissions, audit records |
| 5 | `05-design-patterns.md` | Which classic pattern does which job |
| 6 | `06-object-model.md` | UML class diagrams (Mermaid), one per subsystem |
| 7 | `08-walkthrough.md` | Pseudocode for every flow, with event timelines |
| 8 | `09-threat-model.md` | Principals, assets, mitigations, accepted risks |
| ref | `02-layering-and-composition.md` | Layers, composition rules, configuration reference with defaults, startup validation |
| ref | `03-web-api.md` | Endpoints, event types, error catalogue, permission matrix |
| ref | `GLOSSARY.md` | One meaning per term |

Decisions live in `../decisions/` (0001–0016, numbered; later records
may refine earlier ones; 0012–0016 record the policy, capability,
approval, accepted-risk, and step-kind rules). Open questions, the
decisions still needed from the owner, and the parking lot live in
`../IDEAS.md`.

## Flow index

| Flow | Walkthrough section | Event timeline |
|---|---|---|
| Service startup | `08` §1 | – |
| Login and session creation | `08` §2 | – |
| One turn with a tool call | `08` §3 | `08` §4 |
| Tool-call approval | `08` §5 | `08` §5h |
| Memory mining and confirmation | `08` §6 | `08` §6g |
| Skill fork and override | `08` §7 | `08` §7h |
| Tool audit and classifier rules | `08` §8 | `08` §8h |
| Prompt assembly | `08` §9 | `08` §9j |
| Memory recall and search | `08` §10 | `08` §10h |
| Session lifecycle and reconnect | `08` §11 | `08` §11h |
| Tool execution and sandbox | `08` §12 | `08` §12i |
| Background jobs and notification | `08` §13 | `08` §13h |
| Client tools and the session layer | `08` §14 | `08` §14h |
| Skill discovery and loading | `08` §15 | `08` §15j |
| Classifier engine and model client adapters | `08` §16 | `08` §16g |
| Event bus and streaming | `08` §17 | `08` §17j |
| Client-initiated skill load (system turn) | `08` §15d | `08` §15j |
| Freeze and unfreeze | `08` §11 | – |
| Owner-level approvals (memory approvals across sessions) | `08` §5g, §11c | `08` §6g |

The list at the end of `08-walkthrough.md` ("Things this walkthrough
pins down") is the one-page summary of every rule the flows forced.
