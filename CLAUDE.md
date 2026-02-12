# PSP Orchestration Kit

A toolkit for generating payment service provider (PSP) integrations using declarative JSON mapping files and Claude Code skills.

## What This Repo Does

This repo contains everything Claude Code needs to:
1. **Research a new PSP** and create a mapping file describing its payments API
2. **Generate a working SDK** in any language from one or more mapping files

The mapping files are the single source of truth. They declaratively describe how each PSP's API maps to a unified payment model — field names, auth, status codes, error codes, 3DS, recurring payments, and all 5 payment source types.

## Repository Structure

```
schema/psp-mapping.schema.json    # JSON Schema — the contract all mappings follow
mappings/*.json                   # One file per PSP (adyen.json, checkout.json, etc.)
sdk-template/                     # Language-agnostic patterns for SDK generation
  unified-types.md                # Enums, request/response models
  client-pattern.md               # Provider interface + client wrapper
  error-handling.md               # Error mapping logic
  language-adaptation.md          # Guide for adapting patterns to any language
docs/                             # Human-readable documentation
  schema-reference.md             # Field-by-field schema docs
  source-types.md                 # The 5 source types explained
  adding-a-psp.md                 # Manual guide for adding a PSP
.claude/skills/                   # Claude Code skills
  generate-mapping-for-psp/skill.md  # /generate-mapping-for-psp [psp-name]
  generate-sdk/skill.md           # /generate-sdk [language] [psp-names] [output-dir]
```

## Key Concepts

### The 5 Source Types

Every mapping file describes how each of these payment source types is handled by the PSP:

| Source Type | Description | Example |
|---|---|---|
| `raw_pan` | Raw card number + expiry + CVC | Direct PCI-scoped integration |
| `basis_theory_token` | Stored Basis Theory token | Detokenized via proxy at payment time |
| `basis_theory_token_intent` | Ephemeral token intent | Same proxy flow, one-time use |
| `network_token` | Network token (DPAN) + cryptogram | From Visa/Mastercard token services |
| `processor_token` | Token stored with the PSP | Adyen storedPaymentMethodId, Checkout source ID |

### Mapping Conventions

- **Dot-notation** for nested fields: `"paymentMethod.type": "scheme"` becomes `{ "paymentMethod": { "type": "scheme" } }`
- **`{{placeholder}}`** for dynamic values from the unified model: `"{{amount.value}}"`, `"{{source.id}}"`
- **`$config.*`** references config fields from the authentication section: `"$config.merchant_account"`
- **`$raw`** in response mappings captures the full PSP response
- **`use_mapping`** in field mappings references `status_mappings` or `error_mappings` for value transformation

### Unified Statuses

`authorized` | `declined` | `error` | `cancelled` | `pending` | `action_required` | `partially_authorized` | `captured` | `refunded` | `voided` | `received`

### Unified Error Categories

`card_declined` | `insufficient_funds` | `expired_card` | `invalid_card` | `cvc_declined` | `blocked_card` | `fraud_detected` | `three_ds_failed` | `issuer_unavailable` | `not_supported` | `acquirer_error` | `pin_error` | `cancelled_by_shopper` | `avs_declined` | `authentication_error` | `validation_error` | `rate_limit` | `psp_error` | `unknown`

## Skills

### `/generate-mapping-for-psp [psp-name]`

Researches a PSP's payments API and creates a complete mapping file. The skill:
1. Web-searches for the PSP's API docs, error codes, 3DS support, and network token format
2. Confirms integration details with the user
3. Loads the schema and an existing mapping as reference
4. Builds the mapping section by section
5. Validates and writes to `mappings/{psp-name}.json`

### `/generate-sdk [language] [psp-names] [output-dir]`

Generates a working SDK from mapping files. The skill:
1. Lists available mappings, confirms language and output directory
2. Loads selected mapping files + SDK templates
3. Generates idiomatic code: unified models, provider per PSP, client wrapper, HTTP layer
4. Writes to the output directory with proper project structure

## Working With Mapping Files

When reading or editing mapping files:
- Always validate against `schema/psp-mapping.schema.json`
- Use existing mappings (`adyen.json`, `checkout.json`) as reference for conventions
- The `source_types` section is the most important — it defines request transforms per source type
- Error mappings group PSP codes by unified category (not 1:1)
- Status mappings are a direct lookup table (PSP status string -> unified status)
