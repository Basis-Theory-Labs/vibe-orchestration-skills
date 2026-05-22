# PSP Orchestration Kit

> Declarative JSON mappings + agent skills for generating payment processor integrations.

## The Idea

Payment processors all do the same thing — authorize, capture, refund — but each one has a different API shape. This kit captures each PSP's API as a declarative mapping file, then uses agent skills to generate working SDKs from those mappings.

**Three skills, any PSP, any language:**

```
/generate-mapping-for-psp stripe             # Research & build a mapping file
/generate-sdk python adyen,stripe ./my-sdk  # Generate a working SDK
/build-demo                                 # Generate an interactive web demo
```

## How It Works

```
┌─────────────────────┐     ┌──────────────────┐     ┌─────────────────────┐     ┌─────────────────────┐
│  PSP API Docs       │     │  Mapping File    │     │  Generated SDK      │     │  Interactive Demo   │
│  (Adyen, Stripe..)  │────▶│  (JSON)          │────▶│  (Python, TS, ..)   │────▶│  (Web UI + Server)  │
│                     │     │                  │     │                     │     │                     │
└─────────────────────┘     └──────────────────┘     └─────────────────────┘     └─────────────────────┘
    /generate-mapping-for-psp    Source of Truth        /generate-sdk               /build-demo
```

Each mapping file declaratively describes:
- **Authentication** — API keys, bearer tokens, header configuration
- **Source types** — How raw cards, tokens, and network tokens map to PSP fields
- **Operations** — authorize, capture, refund, cancel with full field mappings
- **Status codes** — PSP-specific statuses mapped to a unified set
- **Error codes** — Hundreds of PSP error codes categorized into unified types
- **3D Secure** — Field mappings for 3DS authentication data
- **Recurring** — Card-on-file, subscription, and unscheduled payment types

## Repository Structure

```
schema/psp-mapping.schema.json  # JSON Schema all mappings validate against
mappings/
  adyen.json                    # Complete Adyen mapping
  checkout.json                 # Complete Checkout.com mapping
sdk-template/                   # Language-agnostic SDK patterns
  unified-types.md              # Universal payment models
  client-pattern.md             # Provider interface pattern
  error-handling.md             # Error mapping logic
  language-adaptation.md        # Guide for adapting patterns to any language
docs/                           # Schema reference, source types guide
agent-skills/                   # Agent command files
```

## Quick Start

### Create a Mapping for a New PSP

```text
/generate-mapping-for-psp stripe
```

The agent will research Stripe's API, walk through the mapping, and write
`mappings/stripe.json`.

### Generate an SDK

```bash
/generate-sdk python adyen,checkout ./output
```

The agent reads the mapping files, loads the SDK templates, and generates idiomatic Python code with unified models, per-PSP providers, and a client wrapper.

### Build an Interactive Demo

```bash
/build-demo
```

The agent finds the generated SDK, detects its language, and generates a web-based demo app (HTTP server + single-page UI) in the same language. The demo lets you test authorize, capture, refund, and cancel flows against live PSP sandboxes, with a built-in Basis Theory tokenization tab.

## The 5 Source Types

Every mapping handles all five ways a card can be presented:

| Source Type | What It Is | PCI Required? |
|---|---|---|
| `raw_pan` | Card number + expiry + CVC | Yes |
| `basis_theory_token` | Stored Basis Theory token | No (proxy) |
| `basis_theory_token_intent` | One-time Basis Theory token | No (proxy) |
| `network_token` | Visa/MC network token + cryptogram | No |
| `processor_token` | PSP's stored payment method | No |

## Mapping File Example

A simplified look at how Adyen's authorize operation is mapped:

```json
{
  "operations": {
    "authorize": {
      "method": "POST",
      "path": "/payments",
      "request_mapping": [
        { "from": "$unified.amount.value", "to": "amount.value" },
        { "from": "$unified.amount.currency", "to": "amount.currency" },
        { "from": "$config.merchant_account", "to": "merchantAccount" }
      ],
      "response_mapping": [
        { "from": "$response.pspReference", "to": "id" },
        { "from": "$response.resultCode", "to": "status", "use_mapping": "status_mappings" }
      ]
    }
  }
}
```

## Documentation

- [Schema Reference](docs/schema-reference.md) — Every field in the mapping schema
- [Source Types](docs/source-types.md) — Deep dive into the 5 source types
- [Adding a PSP](docs/adding-a-psp.md) — Manual guide (or just use the skill)

## Built With

- [Basis Theory](https://basistheory.com) — Token vault and proxy for PCI-free card processing
- Agent-readable skill files, declarative JSON mappings, and language-neutral SDK
  templates
