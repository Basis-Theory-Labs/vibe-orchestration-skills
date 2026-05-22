# Client Pattern

This document describes the provider interface and client wrapper pattern that every generated SDK must follow.

---

## Architecture

```
┌──────────────┐     ┌────────────────────┐     ┌──────────────┐
│   Client     │────▶│  Provider          │────▶│  PSP API     │
│  (Unified)   │     │  (Per-PSP impl)    │     │              │
└──────────────┘     └────────────────────┘     └──────────────┘
                     ┌────────────────────┐     ┌──────────────┐
                     │  Provider          │────▶│  PSP API     │
                     │  (Another PSP)     │     │              │
                     └────────────────────┘     └──────────────┘
```

The SDK has three layers:

1. **Provider Interface** — Abstract contract that every PSP implements
2. **Provider Implementation** — Per-PSP class generated from a mapping file
3. **Client** — Unified entry point that wraps one or more providers

---

## Provider Interface

Every PSP provider must implement this interface:

```
interface PaymentProvider:
    name: string                     # PSP identifier (e.g., "adyen")

    authorize(request: TransactionRequest, idempotency_key?: string)
        -> TransactionResponse
        raises TransactionError

    capture(transaction_id: string, amount: Amount, reference?: string)
        -> TransactionResponse
        raises TransactionError

    get(transaction_id: string)
        -> TransactionResponse
        raises TransactionError

    refund(request: RefundRequest, idempotency_key?: string)
        -> RefundResponse
        raises TransactionError

    cancel(transaction_id: string, reference?: string)
        -> TransactionResponse
        raises TransactionError
```

### Provider Initialization

Each provider is initialized with:
- PSP-specific config fields (from the mapping's `authentication.config_fields`)
- Whether to use test or live environment
- Optional Basis Theory API key for proxy-based detokenization

```
provider = AdyenProvider(
    api_key="...",
    merchant_account="...",
    is_test=True,
    bt_api_key="..."
)
```

---

## Provider Implementation (Generated)

For each PSP mapping, generate a provider class that:

### 1. Builds the request

```
method authorize(request):
    # Start with operation-level request_mapping
    payload = apply_request_mappings(mapping.operations.authorize.request_mapping, request, config)

    # Apply source-type-specific transform
    source_transform = mapping.source_types[request.source.type].request_transform
    merge(payload, expand_dot_notation(source_transform, request))

    # Apply recurring type if present
    if request.type and mapping.recurring:
        psp_value = mapping.recurring.unified_to_psp[request.type]
        set_field(payload, mapping.recurring.psp_field, psp_value)

    # Apply 3DS fields if present
    if request.three_ds and mapping.three_ds:
        for unified_field, psp_field in mapping.three_ds.request_fields:
            value = get_field(request.three_ds, unified_field)
            if value:
                set_field(payload, psp_field, value)

    return payload
```

### 2. Makes the HTTP request

```
method execute_request(operation, payload, idempotency_key):
    url = base_url + interpolate_path(operation.path, payload)
    headers = build_auth_headers(mapping.authentication)

    if idempotency_key:
        headers[idempotency_header] = idempotency_key

    # Route through BT proxy if source type requires it
    if needs_proxy(request.source.type):
        response = bt_proxy_request(url, headers, payload)
    else:
        response = http_request(operation.method, url, headers, payload)

    return response
```

### 3. Transforms the response

```
method transform_response(response_data, operation):
    result = {}
    for mapping in operation.response_mapping:
        value = get_field(response_data, mapping.from.replace("$response.", ""))

        if mapping.use_mapping == "status_mappings":
            value = status_mappings[value] or "declined"
        elif mapping.use_mapping == "error_mappings":
            value = find_error_category(error_mappings, value)

        set_field(result, mapping.to, value)

    return TransactionResponse(result)
```

---

## Client Wrapper

The client provides a unified API over one or more providers:

```
class PaymentClient:
    providers: map<string, PaymentProvider>

    constructor(providers: list<PaymentProvider>):
        for provider in providers:
            self.providers[provider.name] = provider

    method authorize(provider_name, request, idempotency_key?):
        return self.providers[provider_name].authorize(request, idempotency_key)

    method capture(provider_name, transaction_id, amount, reference?):
        return self.providers[provider_name].capture(transaction_id, amount, reference)

    method get(provider_name, transaction_id):
        return self.providers[provider_name].get(transaction_id)

    method refund(provider_name, request, idempotency_key?):
        return self.providers[provider_name].refund(request, idempotency_key)

    method cancel(provider_name, transaction_id, reference?):
        return self.providers[provider_name].cancel(transaction_id, reference)
```

### Usage

```
client = PaymentClient([
    AdyenProvider(api_key="...", merchant_account="...", is_test=True),
    CheckoutProvider(secret_key="...", processing_channel="...", is_test=True)
])

response = client.authorize("adyen", TransactionRequest(
    amount=Amount(value=1000, currency="USD"),
    source=Source(type=SourceType.BASIS_THEORY_TOKEN, id="tok_..."),
    reference="order-123"
))
```

---

## HTTP Layer

The HTTP layer must support:

1. **Direct requests** — Standard HTTP to the PSP's API
2. **Proxy requests** — HTTP via Basis Theory's proxy for token detokenization
3. **Idempotency** — PSP-specific idempotency headers
4. **Error handling** — Map HTTP errors to `TransactionError`

### Basis Theory Proxy

For `basis_theory_token` and `basis_theory_token_intent` source types, requests go through `https://api.basistheory.com/proxy` with:
- `BT-API-KEY` header set to the Basis Theory API key
- `BT-PROXY-URL` header set to the PSP's actual endpoint
- The request body contains Basis Theory expression templates that get detokenized in-flight

### Idempotency Headers

Each PSP uses a different header name for idempotency. The mapping file's `psp.idempotency_header` specifies this:

| PSP | Header Name |
|---|---|
| Stripe | `Idempotency-Key` |
| Adyen | `Idempotency-Key` |
| Checkout.com | `Cko-Idempotency-Key` |

If `psp.idempotency_header` is absent, default to `Idempotency-Key`.

### Status Casing

Some PSPs return different casing across endpoints. For example, Adyen returns PascalCase statuses from `/payments` (e.g., `Received`, `Pending`) but lowercase from `/captures`, `/refunds`, `/cancels` (e.g., `received`, `pending`).

The `status_mappings` in the mapping file should include both casing variants when a PSP exhibits this behavior. SDK generators should be aware that status mapping lookups must match exactly — consider case-insensitive lookup or ensure both variants are in the mapping.

### Refund Endpoint Patterns

PSPs handle refunds differently:

| Pattern | PSPs | Path |
|---|---|---|
| **Sub-resource** | Adyen, Checkout.com | `POST /payments/{id}/refunds` |
| **Top-level resource** | Stripe | `POST /v1/refunds` (with `payment_intent` in body) |

When a refund is a top-level resource, the original transaction ID is passed in the request body (e.g., `payment_intent` field for Stripe) rather than as a path parameter. The mapping file's `operations.refund.path` indicates which pattern is used — if the path contains `{transaction_id}`, it's a sub-resource; otherwise, it's top-level.

### Auto-Capture Behavior

Some PSPs auto-capture payments by default (e.g., Checkout.com). To ensure authorize-only works correctly, the mapping's `operations.authorize.request_mapping` should include a `capture` field with `"default": false` for PSPs that auto-capture. This explicitly tells the PSP not to capture immediately.

### Error Handling

HTTP errors should be caught and transformed:
1. Parse the error response body
2. Look up the error code in `error_mappings.categories`
3. Fall back to `error_mappings.http_errors` for HTTP-status-based classification
4. Throw a `TransactionError` with the unified error response

---

## Code Generation Checklist

When generating a provider from a mapping file:

- [ ] Provider class implements the `PaymentProvider` interface
- [ ] Constructor accepts all `authentication.config_fields`
- [ ] `authorize` applies source-type transforms based on `request.source.type`
- [ ] `authorize` applies recurring type and 3DS mappings when present
- [ ] All operations use correct HTTP method and path from the mapping
- [ ] Path parameters (e.g., `{transaction_id}`) are interpolated
- [ ] Response fields are mapped using `response_mapping`
- [ ] Status values are transformed via `status_mappings`
- [ ] Error codes are categorized via `error_mappings`
- [ ] HTTP errors are caught and transformed to `ErrorResponse`
- [ ] Proxy routing is used for BT token source types
- [ ] Idempotency key is forwarded using the PSP-specific header from `psp.idempotency_header`
- [ ] HTTP layer handles `content_type` from `psp.content_type` (JSON vs form-encoded)
- [ ] Form-encoded requests flatten nested objects and arrays to bracket notation
- [ ] Status mapping handles both casing variants when PSP returns different casing per endpoint
- [ ] Refund handles both sub-resource and top-level resource patterns based on `operations.refund.path`
- [ ] Default values in `request_mapping` are applied (e.g., `capture: false` for auto-capture PSPs)
