# basic_agent

## Project status

This is a **design and ideas project only**. There is no code yet, no build,
and no tests. The directory exists to collect and refine ideas about a new
project (working name: "basic agent", repository "QuickAI2") before any
implementation begins. It is tracked in git at
git@github.com:jstormes/QuickAI2.git; commit design changes as you go.

## How to work here

- Treat the contents as design notes, brainstorms, sketches, and open
  questions, not as a codebase.
- Prefer plain Markdown files for ideas. Keep one topic per file where it
  helps, and link related notes to each other.
- Do not scaffold application code, package manifests, build tooling, CI, or
  test suites unless explicitly asked. Implementation is a later phase.
- When asked to explore an idea, favor writing it down (options, trade-offs,
  open questions) over building a prototype.
- It is fine to include small illustrative snippets or pseudocode inside
  design notes when they clarify a concept.

## Suggested layout (create as needed)

- `CLAUDE.md` – this file
- `IDEAS.md` – running list of raw ideas and questions
- `design/` – longer design notes on specific topics (start with `00-vision.md`;
  `03-web-api.md` covers the streaming HTTP API all front ends use,
  `04-auth-and-permissions.md` covers OAuth2 login and permission scopes,
  `07-turn-pipeline.md` is the spine: the chain of layer handlers,
  `05-design-patterns.md` names the patterns, `06-object-model.md` is the
  UML class model in Mermaid, `08-walkthrough.md` is pseudocode for
  every flow, `09-threat-model.md` names the trust boundaries)
- `decisions/` – short notes recording decisions and why they were made

## When this changes

Once the project moves from ideas to implementation, update this file to
describe the chosen stack, build and test commands, and code conventions.
