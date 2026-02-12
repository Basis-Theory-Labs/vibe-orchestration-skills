# Source Types

Every PSP mapping describes how to handle five distinct ways a card can be presented for payment. This document explains each type, when it's used, and what a mapping must define.

---

## Overview

| # | Source Type | Data Flow | PCI Scope? | Basis Theory Proxy? |
|---|---|---|---|---|
| 1 | `raw_pan` | Card data flows through your servers | Yes | No |
| 2 | `basis_theory_token` | Stored token detokenized at payment time | No | Yes |
| 3 | `basis_theory_token_intent` | Ephemeral token detokenized at payment time | No | Yes |
| 4 | `network_token` | DPAN + cryptogram from card networks | No | No |
| 5 | `processor_token` | PSP's own stored payment method ID | No | No |

---

## 1. Raw PAN (`raw_pan`)

**What it is:** The actual card number, expiry date, and CVC are sent directly in the API request.

**When it's used:** PCI Level 1 certified environments where card data can be handled directly.

**Mapping requirements:**
- `requires_pci: true`
- `request_transform` maps `{{source.number}}`, `{{source.expiry_month}}`, `{{source.expiry_year}}`, and `{{source.cvc}}` to the PSP's card fields

**Example (Adyen):**
```json
{
  "request_transform": {
    "paymentMethod.type": "scheme",
    "paymentMethod.number": "{{source.number}}",
    "paymentMethod.expiryMonth": "{{source.expiry_month}}",
    "paymentMethod.expiryYear": "{{source.expiry_year}}",
    "paymentMethod.cvc": "{{source.cvc}}"
  },
  "requires_pci": true
}
```

---

## 2. Basis Theory Token (`basis_theory_token`)

**What it is:** A stored Basis Theory token that gets detokenized at the moment of payment via the Basis Theory proxy. The actual card data never touches your servers.

**When it's used:** Recurring payments, saved cards, any scenario where a card was previously collected and stored as a Basis Theory token.

**Mapping requirements:**
- `detokenization_strategy: "basis_theory_proxy"`
- `proxy_config` defining the expression prefix (`"token"`) and card field mappings
- `request_transform` uses Basis Theory expression syntax to reference token fields

**Example (Adyen):**
```json
{
  "request_transform": {
    "paymentMethod.type": "scheme",
    "paymentMethod.number": "{{ token: {{source.id}} | json: '$.data.number' }}",
    "paymentMethod.expiryMonth": "{{ token: {{source.id}} | json: '$.data.expiration_month' }}",
    "paymentMethod.expiryYear": "{{ token: {{source.id}} | json: '$.data.expiration_year' }}",
    "paymentMethod.cvc": "{{ token: {{source.id}} | json: '$.data.cvc' }}"
  },
  "detokenization_strategy": "basis_theory_proxy",
  "proxy_config": {
    "transform_expression_prefix": "token",
    "card_field_mappings": {
      "number": "paymentMethod.number",
      "expiry_month": "paymentMethod.expiryMonth",
      "expiry_year": "paymentMethod.expiryYear",
      "cvc": "paymentMethod.cvc"
    }
  }
}
```

---

## 3. Basis Theory Token Intent (`basis_theory_token_intent`)

**What it is:** An ephemeral, one-time-use Basis Theory token. Same proxy-based detokenization as `basis_theory_token`, but the token can only be used once.

**When it's used:** First-time card payments where the card was just collected in the frontend. Token intents expire after use.

**Mapping requirements:**
- Same as `basis_theory_token` but with `transform_expression_prefix: "token_intent"`

**Example (Checkout.com):**
```json
{
  "request_transform": {
    "source.type": "card",
    "source.number": "{{ token_intent: {{source.id}} | json: '$.data.number' }}",
    "source.expiry_month": "{{ token_intent: {{source.id}} | json: '$.data.expiration_month' }}",
    "source.expiry_year": "{{ token_intent: {{source.id}} | json: '$.data.expiration_year' }}",
    "source.cvv": "{{ token_intent: {{source.id}} | json: '$.data.cvc' }}"
  },
  "detokenization_strategy": "basis_theory_proxy",
  "proxy_config": {
    "transform_expression_prefix": "token_intent",
    "card_field_mappings": {
      "number": "source.number",
      "expiry_month": "source.expiry_month",
      "expiry_year": "source.expiry_year",
      "cvc": "source.cvv"
    }
  }
}
```

---

## 4. Network Token (`network_token`)

**What it is:** A Device PAN (DPAN) issued by Visa or Mastercard's token services, along with a cryptogram that proves the token is valid. Network tokens improve authorization rates and reduce fraud.

**When it's used:** When your token vault provisions network tokens from the card networks, or when using services like Basis Theory that manage network tokenization.

**Mapping requirements:**
- `request_transform` maps the DPAN, expiry, and cryptogram to PSP-specific fields
- Each PSP has its own format for network tokens (Adyen uses `mpiData`, Checkout.com uses `source.type: "network_token"`)

**Example (Adyen):**
```json
{
  "request_transform": {
    "paymentMethod.type": "networkToken",
    "paymentMethod.number": "{{source.number}}",
    "paymentMethod.expiryMonth": "{{source.expiry_month}}",
    "paymentMethod.expiryYear": "{{source.expiry_year}}",
    "mpiData.cavv": "{{source.cryptogram}}",
    "mpiData.eci": "{{source.eci}}"
  }
}
```

**Example (Checkout.com):**
```json
{
  "request_transform": {
    "source.type": "network_token",
    "source.token": "{{source.number}}",
    "source.expiry_month": "{{source.expiry_month}}",
    "source.expiry_year": "{{source.expiry_year}}",
    "source.cryptogram": "{{source.cryptogram}}",
    "source.eci": "{{source.eci}}"
  }
}
```

---

## 5. Processor Token (`processor_token`)

**What it is:** A payment method token that was previously stored with the PSP itself. Each PSP has its own token format and storage mechanism.

**When it's used:** When a card was previously saved with the PSP (e.g., via `storePaymentMethod: true` in Adyen, or `store_for_future_use: true` in Checkout.com).

**Mapping requirements:**
- `request_transform` places the token ID in the PSP's expected field
- No proxy or detokenization needed — the PSP already has the card data

**Example (Adyen):**
```json
{
  "request_transform": {
    "paymentMethod.type": "scheme",
    "paymentMethod.storedPaymentMethodId": "{{source.id}}"
  }
}
```

**Example (Checkout.com):**
```json
{
  "request_transform": {
    "source.type": "id",
    "source.id": "{{source.id}}"
  }
}
```

---

## How Source Types Affect SDK Generation

When generating an SDK, each source type maps to a different code path:

1. **`raw_pan`** — Card fields are placed directly in the request body
2. **`basis_theory_token` / `basis_theory_token_intent`** — Request is routed through the Basis Theory proxy with expression templates
3. **`network_token`** — Card + cryptogram fields are placed in PSP-specific locations
4. **`processor_token`** — Just the token ID, sent directly to the PSP (no proxy needed)

The generated provider code uses the `request_transform` from the mapping to build the correct request body for each source type.
