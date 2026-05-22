# Payment Skills

Reusable agent skills for payment teams.

This repo turns payment expertise into working skill packs: prompts, domain
models, examples, schemas, and source material an AI agent can use to help with
real payment work. The first non-builder skill is intentionally deep instead of
fragmented: the Director of Payments Copilot contains the operating cadence,
diagnostic lens, executive narrative, annual planning model, and strategy
workflows in one place.

## Why This Exists

Most payment content is static. A PDF explains what to do, a diagram explains how
the system works, and an internal doc explains what the team decided.

Payment teams need a more useful shape:

- Ask the operating model what to review on Monday morning.
- Turn raw payment metrics into a WBR agenda with owners.
- Diagnose payment performance by separating controllable levers from external
  issuer, network, and processor behavior.
- Generate PSP mappings and SDKs from declarative integration specs.
- Convert payment strategy into QBR, annual-planning, and three-year narratives
  executives can act on.

## Skill Packs

```text
skills/
  director-of-payments-copilot/
    SKILL.md
    knowledge/
    examples/

  psp-orchestration/
    SKILL.md
    mappings/
    schema/
    sdk-template/
    agent-skills/
```

## Current Skills

### Director of Payments Copilot

Use the Director of Payments Playbook as a deep interactive operating model for
payments leadership.

Example asks:

- `Build a Monday WBR agenda from these payment metrics.`
- `Explain what changed this week in CFO language.`
- `Separate controllable payment issues from external issuer behavior.`
- `Turn these payment initiatives into QBR talking points.`
- `Draft a three-year payments strategy from these constraints.`
- `Pressure-test this annual plan against revenue, cost, CX, and operational risk.`

Source: https://go.basistheory.com/resources/director-of-payments-playbook

### PSP Orchestration

The original PSP orchestration kit now lives under
`skills/psp-orchestration/`. It contains declarative PSP mappings, a JSON schema,
SDK generation templates, and agent command files for generating new PSP mappings,
SDKs, and demos.

## Examples

- `examples/director-of-payments-post.md` - post draft for introducing the
  playbook as a skill, not just a PDF.
- `skills/director-of-payments-copilot/knowledge/product-directions.md` -
  product directions centered on the copilot instead of scattered shallow skills.

## Repository Rule

Each skill should be useful as a standalone folder. If an agent only has that
folder, it should still understand the domain, know what inputs to ask for, and
produce a concrete artifact.
