# /generate-mapping-for-psp

Create a PSP mapping file by researching a payment processor's API and building a complete JSON mapping.

## Usage

```
/generate-mapping-for-psp [psp-name]
```

If `psp-name` is not provided, ask the user which PSP they want to map.

## Process

### Step 1: Research the PSP

Use web search to find the PSP's API documentation. Search for:

1. `{psp-name} payments API documentation` — Find the main API reference
2. `{psp-name} API authentication` — How to authenticate (API key, bearer token, etc.)
3. `{psp-name} payment request fields` — Request body for creating a payment
4. `{psp-name} error codes decline codes` — Full list of error/refusal codes
5. `{psp-name} 3D Secure API` — How 3DS data is sent and received
6. `{psp-name} network token payment` — How network tokens (DPAN + cryptogram) are formatted
7. `{psp-name} stored payment method token` — How to use previously saved cards
8. `{psp-name} capture refund void API` — Capture, refund, and cancel endpoints
9. `{psp-name} recurring payment types` — Card-on-file, subscription, unscheduled flags
10. `{psp-name} idempotency key header` — What header name is used for idempotency
11. `{psp-name} API content type encoding` — Whether the API uses JSON or form-encoded requests
12. `{psp-name} auto capture default behavior` — Whether payments are captured automatically by default

### Step 2: Confirm with User

Present a summary of what was found:
- Base URLs (test + production)
- Authentication method
- API version
- Content type (JSON vs form-encoded)
- Idempotency header name
- Auto-capture default behavior (does authorize auto-capture?)
- Refund endpoint pattern (sub-resource of payment vs top-level resource)
- Status casing (does casing vary across endpoints?)
- Key differences from existing PSPs (Adyen/Checkout.com)
- Any gaps in documentation found

Ask the user to confirm before proceeding, and whether they have any additional context (e.g., specific API docs pages, Notion content, or integration guides).

### Step 3: Load References

Read these files to understand the schema and conventions:

```
schema/psp-mapping.schema.json    # The schema the mapping must validate against
mappings/adyen.json               # Reference mapping (complex PSP)
mappings/checkout.json            # Reference mapping (different patterns)
docs/source-types.md              # How to map the 5 source types
```

### Step 4: Build the Mapping

Build the mapping file section by section:

1. **`psp`** — Name, display name, base URLs, API version, docs URL, content type, idempotency header
2. **`authentication`** — Type, header, config fields with env var suggestions
3. **`source_types`** — This is the most important section. For each of the 5 source types:
   - `raw_pan`: Map card fields to PSP's request format
   - `basis_theory_token`: Same fields but with `{{ token: ... }}` expression syntax
   - `basis_theory_token_intent`: Same but with `{{ token_intent: ... }}` prefix
   - `network_token`: Map DPAN + cryptogram to PSP-specific fields (this varies significantly between PSPs)
   - `processor_token`: Map the PSP's stored payment method ID
4. **`operations`** — Map each operation (authorize, capture, get, confirm, refund, cancel) with request and response field mappings
5. **`status_mappings`** — Map every PSP status to a unified status
6. **`error_mappings`** — Group all PSP error codes by unified error category
7. **`amount_format`** — Minor units vs major units, special currencies
8. **`recurring`** — Map recurring types (card_on_file, subscription, unscheduled)
9. **`three_ds`** — Map 3DS request and response fields
10. **`setup`** — Onboarding steps, required and optional env vars

### Step 5: Validate and Write

1. Write the mapping to `mappings/{psp-name}.json`
2. Validate the file is well-formed JSON
3. Check key sections are populated (source_types has all 5, operations has authorize at minimum)
4. Present a summary to the user

## Key Rules

- Every mapping MUST have all 5 source types, even if the PSP doesn't natively support all of them. Document limitations in comments.
- Use existing mappings (adyen.json, checkout.json, stripe.json) as the gold standard for conventions.
- The `request_transform` in source types uses dot-notation keys and `{{placeholder}}` values.
- Error mappings group PSP codes BY unified category (not 1:1 mapping).
- Status mappings are a flat PSP-status-string to unified-status lookup.
- The `$schema` field should always be `"../schema/psp-mapping.schema.json"`.
- Network token format varies significantly between PSPs — research this carefully.
- For Basis Theory proxy transforms, the expression syntax is `{{ token: {id} | json: '$.data.{field}' }}`.

## Key Gotchas (Learned from Real Integrations)

These are common pitfalls discovered during real PSP integrations. Research each one explicitly:

1. **Status casing varies across endpoints** — Some PSPs (e.g., Adyen) return PascalCase from the authorize endpoint (`Received`, `Pending`) but lowercase from capture/refund/cancel endpoints (`received`, `pending`). Include both casing variants in `status_mappings`.

2. **Auto-capture behavior** — Some PSPs (e.g., Checkout.com) auto-capture payments by default. If the PSP auto-captures, add `{ "from": "$unified.capture", "to": "{psp_capture_field}", "default": false }` to the authorize `request_mapping`.

3. **Content type encoding** — Most PSPs use JSON, but some (e.g., Stripe) use `application/x-www-form-urlencoded`. This requires the SDK to flatten nested objects to bracket notation (e.g., `source[type]=card`). Set `psp.content_type` correctly.

4. **Idempotency header name** — PSPs use different header names. Set `psp.idempotency_header` (e.g., `Idempotency-Key` for Stripe/Adyen, `Cko-Idempotency-Key` for Checkout.com).

5. **Refund as top-level resource** — Some PSPs (e.g., Stripe) don't use a sub-resource pattern for refunds. Stripe uses `POST /v1/refunds` with the payment ID in the body, not `POST /payments/{id}/refunds`. The `operations.refund.path` must reflect this.

6. **Customer field restrictions** — Some PSPs (e.g., Stripe) expect their own customer IDs (e.g., `cus_xxx`) in the customer field, not freeform strings. Document this in the mapping.

7. **Default values for required fields** — Some PSPs require fields that have a "right default" for the common case. For example, Stripe needs `payment_method_types: ["card"]` to avoid requiring a `return_url`, and `confirm: true` + `capture_method: manual` for authorize-only. Use the `default` field in `request_mapping` entries.

## Example Interaction

```
User: /generate-mapping-for-psp stripe

Claude: I'll research Stripe's Payments API to build a mapping file.
[researches Stripe's API docs, error codes, 3DS, network tokens...]

Here's what I found about Stripe:
- Base URL: https://api.stripe.com (same for test/live, key determines environment)
- Auth: Bearer token via Authorization header
- API version: 2024-12-18 (set via Stripe-Version header)
- Amount: Minor units (cents)
- Key difference: Stripe uses form-encoded requests, not JSON

Should I proceed with building the mapping? Do you have any additional context?

User: Yes, proceed. We use their PaymentIntents API.

Claude: [builds mapping section by section, writes to mappings/stripe.json]
```
