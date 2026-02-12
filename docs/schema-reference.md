# Schema Reference

Complete field reference for `schema/psp-mapping.schema.json`.

---

## Top-Level Fields

| Field | Type | Required | Description |
|---|---|---|---|
| `$schema` | string | No | Path to the JSON Schema file for editor validation |
| `version` | string | Yes | Semver version of this mapping (e.g., `"1.0.0"`) |
| `psp` | object | Yes | PSP identification and API surface |
| `authentication` | object | Yes | Authentication configuration |
| `source_types` | object | Yes | Payment source type transforms |
| `operations` | object | Yes | Unified operation mappings |
| `status_mappings` | object | Yes | PSP status to unified status lookup |
| `error_mappings` | object | Yes | PSP error code categorization |
| `amount_format` | object | Yes | How the PSP expects monetary amounts |
| `recurring` | object | No | Recurring payment type mappings |
| `three_ds` | object | No | 3D Secure field mappings |
| `setup` | object | No | Developer onboarding instructions |

---

## `psp`

Identifies the payment service provider.

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Machine-readable ID (`^[a-z][a-z0-9_]*$`). Used in file names and generated code |
| `display_name` | string | Yes | Human-readable name (e.g., `"Checkout.com"`) |
| `base_urls.test` | string (URI) | Yes | Sandbox base URL |
| `base_urls.live` | string | Yes | Production base URL. May contain `{{placeholders}}` |
| `api_version` | string | Yes | API version (e.g., `"v71"`) |
| `docs_url` | string (URI) | Yes | Primary API documentation link |
| `content_type` | string | Yes | Content-Type header (default: `"application/json"`) |

### Live URL Placeholders

Some PSPs require merchant-specific URL prefixes in production. Use `{{placeholder}}` syntax:

```json
"live": "https://{{production_prefix}}-checkout-live.adyenpayments.com/checkout/v71"
```

---

## `authentication`

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | enum | Yes | `"api_key"`, `"bearer_token"`, or `"basic_auth"` |
| `header_name` | string | Yes | HTTP header name (e.g., `"X-API-Key"`, `"Authorization"`) |
| `header_prefix` | string | No | Prefix before credential (e.g., `"Bearer"`) |
| `config_fields` | array | Yes | SDK initialization parameters |

### `config_fields` Entry

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Config field name used in SDK code |
| `description` | string | Yes | Human-readable description |
| `required` | boolean | Yes | Whether the field is required at init |
| `env_var` | string | No | Suggested environment variable name |

---

## `source_types`

Contains up to 5 entries, one per payment source type. Each entry has the same shape.

### Source Type Entry

| Field | Type | Required | Description |
|---|---|---|---|
| `request_transform` | object | Yes | PSP field -> value mapping for this source type |
| `requires_pci` | boolean | No | Whether raw card data flows through your servers (default: `false`) |
| `detokenization_strategy` | enum | No | `"basis_theory_proxy"` or `"server_side"` |
| `proxy_config` | object | No | Basis Theory proxy configuration |

### `request_transform`

Key-value pairs where:
- **Keys** are dot-notation PSP field paths (e.g., `"paymentMethod.type"`)
- **Values** are static strings, `{{placeholder}}` dynamic values, or `$config.*` references

```json
{
  "paymentMethod.type": "scheme",
  "paymentMethod.number": "{{source.number}}",
  "paymentMethod.expiryMonth": "{{source.expiry_month}}",
  "merchantAccount": "$config.merchant_account"
}
```

### `proxy_config`

| Field | Type | Description |
|---|---|---|
| `transform_expression_prefix` | string | `"token"` or `"token_intent"` |
| `card_field_mappings` | object | Maps `number`, `expiry_month`, `expiry_year`, `cvc` to PSP field paths |

---

## `operations`

Available operations: `authorize`, `capture`, `get`, `confirm`, `refund`, `cancel`.

### Operation

| Field | Type | Required | Description |
|---|---|---|---|
| `method` | enum | Yes | HTTP method (`GET`, `POST`, etc.) |
| `path` | string | Yes | URL path with `{param}` placeholders |
| `request_mapping` | array | No | Unified -> PSP field mappings |
| `response_mapping` | array | No | PSP -> unified field mappings |

### Field Mapping

| Field | Type | Required | Description |
|---|---|---|---|
| `from` | string | Yes | Source path. Prefixed with `$unified.`, `$response.`, or `$config.` |
| `to` | string | Yes | Target path in dot-notation |
| `use_mapping` | string | No | References `status_mappings` or `error_mappings` for value transform |
| `default` | any | No | Fallback value if source is absent |

---

## `status_mappings`

A flat object mapping PSP status strings to unified statuses.

```json
{
  "Authorised": "authorized",
  "Refused": "declined",
  "Pending": "pending"
}
```

**Unified statuses:** `authorized`, `declined`, `error`, `cancelled`, `pending`, `action_required`, `partially_authorized`, `captured`, `refunded`, `voided`, `received`

---

## `error_mappings`

| Field | Type | Required | Description |
|---|---|---|---|
| `source_field` | string | Yes | Dot-path to error code in PSP response |
| `categories` | object | Yes | Unified category -> `{ codes: [...] }` |
| `http_errors` | object | No | HTTP status code -> unified category |

### Categories

Each key is a unified error category, containing a `codes` array of PSP-specific codes:

```json
{
  "expired_card": { "codes": ["6", "20054", "30033"] },
  "insufficient_funds": { "codes": ["12", "28", "29"] }
}
```

**Unified error categories:** `card_declined`, `insufficient_funds`, `expired_card`, `invalid_card`, `cvc_declined`, `blocked_card`, `fraud_detected`, `three_ds_failed`, `issuer_unavailable`, `not_supported`, `acquirer_error`, `pin_error`, `cancelled_by_shopper`, `avs_declined`, `authentication_error`, `validation_error`, `rate_limit`, `psp_error`, `unknown`

---

## `amount_format`

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | enum | Yes | `"minor_units"` (cents) or `"major_units"` (dollars) |
| `zero_decimal_currencies` | array | No | Currencies with no fractional part (JPY, KRW, etc.) |
| `three_decimal_currencies` | array | No | Currencies with 3 decimal places (BHD, KWD, etc.) |

---

## `recurring`

| Field | Type | Description |
|---|---|---|
| `unified_to_psp` | object | Maps unified recurring types to PSP values |
| `psp_field` | string | Where the recurring type goes in the PSP request |

```json
{
  "unified_to_psp": {
    "one_time": null,
    "card_on_file": "CardOnFile",
    "subscription": "Subscription",
    "unscheduled": "UnscheduledCardOnFile"
  },
  "psp_field": "recurringProcessingModel"
}
```

---

## `three_ds`

| Field | Type | Description |
|---|---|---|
| `request_fields` | object | Unified 3DS field -> PSP request field path |
| `response_fields` | object | PSP 3DS response field -> unified field |

---

## `setup`

| Field | Type | Description |
|---|---|---|
| `steps` | array of strings | Ordered setup instructions |
| `required_env_vars` | array of strings | Required environment variables |
| `optional_env_vars` | array of strings | Optional environment variables |
