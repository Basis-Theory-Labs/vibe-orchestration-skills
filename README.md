# Payment Skills

Reusable agent skills for payment teams.

This repo turns payment expertise into working skill packs: prompts, domain models,
runbooks, examples, schemas, and source material an AI agent can use to help with
real payment work.

## Why This Exists

Most payment content is static. A PDF explains what to do, a diagram explains how
the system works, and an internal doc explains what the team decided.

Payment teams need a more useful shape:

- Ask the operating cadence what to review on Monday morning.
- Turn weekly metrics into a WBR agenda with owners.
- Diagnose an authorization-rate drop by region, processor, method, and decline
  reason.
- Generate PSP mappings and SDKs from declarative integration specs.
- Convert payment strategy into QBR and annual-planning narratives executives can
  act on.

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

  payment-ops-cadence-planner/
    SKILL.md

  auth-rate-diagnosis/
    SKILL.md

  payment-incident-commander/
    SKILL.md
```

## Current Skills

### Director of Payments Copilot

Use the Director of Payments Playbook as an interactive operating model.

Example asks:

- `Build a Monday WBR agenda from these payment metrics.`
- `Explain what changed this week in CFO language.`
- `Separate controllable payment issues from external issuer behavior.`
- `Turn these payment initiatives into QBR talking points.`
- `Draft a three-year payments strategy from these constraints.`

Source: https://go.basistheory.com/resources/director-of-payments-playbook

### PSP Orchestration

The original PSP orchestration kit now lives under
`skills/psp-orchestration/`. It contains declarative PSP mappings, a JSON schema,
SDK generation templates, and agent command files for generating new PSP mappings,
SDKs, and demos.

## Planned Skills

- `payment-ops-cadence-planner` - generate daily, weekly, monthly, quarterly, and
  annual payment operating cadences.
- `auth-rate-diagnosis` - investigate authorization-rate drops and decline shifts.
- `payment-incident-commander` - run payment incidents with severity, owners,
  comms, and recovery checks.

## Examples

- `examples/director-of-payments-post.md` - post draft for introducing the
  playbook as a skill, not just a PDF.
- `docs/project-ideas.md` - follow-on project ideas for turning payment expertise
  into interactive artifacts.

## Repository Rule

Each skill should be useful as a standalone folder. If an agent only has that
folder, it should still understand the domain, know what inputs to ask for, and
produce a concrete artifact.
