# 0003 – Skills can define their own tools

**Status:** accepted (2026-09-14)

## Context

Skills were originally instructions plus static resources. Many useful
skills need a capability the host does not provide (call a specific API,
run a specific script). Without skill-defined tools, every such capability
would have to be added to the service itself, defeating the goal of skills
being maintained independently in their own repos.

## Decision

A skill may declare tools (name, description, input schema, risk level,
where it executes, and an implementation reference). Loading a skill into
a session registers its tools under a skill-namespaced name; unloading
removes them. Skill tools pass through the same dispatcher, classifier
audit, and approval policy as built-in tools, and are constrained by the
skill's declared capabilities.

## Consequences

- Skills become mini-plugins; a skill store is effectively a plugin
  repository.
- The service needs a capability enforcement point and, for script-based
  implementations, a sandbox. Both are now specified: capability
  ceilings and the restricted-subprocess sandbox in `08-walkthrough.md`
  §12, recorded in decision 0011.
- Tool risk can be raised by the runtime based on the skill's layer, since
  global skills come from less trusted maintainers than user-authored ones.
- Clients may be asked to execute tools declared by skills; the API
  exposes the active tool list so clients can check what is expected.
