# Skill Format

Each payment skill is a folder under `skills/`.

Required:

- `SKILL.md` - the operating instructions for the agent.

Optional:

- `knowledge/` - source notes, domain models, playbook summaries, and definitions.
- `examples/` - example inputs and outputs.
- `templates/` - reusable agendas, narratives, checklists, or payload shapes.
- `schemas/` - JSON schemas or structured contracts the skill should follow.
- `agent-skills/` - tool-specific wrappers or command files.

## `SKILL.md` Sections

Use this structure unless the skill needs something more specific:

```text
# Skill Name

## Purpose
What outcome this skill creates.

## When To Use
Triggers and example requests.

## Inputs
What the agent should collect or infer.

## Process
The working steps.

## Output
The artifact shape the user should receive.

## Guardrails
Important boundaries and failure modes.

## References
Local files and public source links.
```

## Output Standard

A skill should not end with generic advice. It should produce one of:

- a meeting agenda
- a diagnosis
- an action plan
- an executive narrative
- a runbook
- a generated mapping, schema, SDK, or demo
- a concise question list when required inputs are missing
