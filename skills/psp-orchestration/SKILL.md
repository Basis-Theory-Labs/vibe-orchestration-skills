# PSP Orchestration

## Purpose

Generate and use declarative PSP mappings for payment integrations, SDKs, and
interactive demos.

## When To Use

Use when the user asks to:

- add or update a PSP mapping
- generate a payment SDK
- build a PSP demo app
- compare payment processor request or response shapes
- model raw PAN, Basis Theory token, token intent, network token, or processor
  token source flows

## Inputs

- PSP name
- target language for generated SDKs
- selected PSP mappings
- output directory
- PSP docs or known integration notes

## Process

1. Load `schema/psp-mapping.schema.json`.
2. Load the relevant mapping files from `mappings/`.
3. Use `sdk-template/` for generated SDK patterns.
4. Validate mappings against the schema.
5. Keep PSP-specific behavior in mappings or provider implementations, not in the
   unified client wrapper.

## Output

Depending on the ask, produce:

- a PSP mapping JSON file
- generated SDK files
- an interactive demo
- a mapping comparison
- a concrete list of missing PSP documentation

## References

- `README.md`
- `schema/psp-mapping.schema.json`
- `mappings/`
- `sdk-template/`
- `docs/`
- `agent-skills/`
