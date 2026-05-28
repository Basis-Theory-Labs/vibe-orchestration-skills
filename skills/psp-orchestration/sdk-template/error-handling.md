# Error Handling

This document describes how generated SDKs should handle and classify errors from PSP APIs.

---

## Error Flow

```
PSP Response (error)
       │
       ▼
┌─────────────────────┐
│ Is HTTP 2xx with    │──Yes──▶ Check response body for decline
│ decline status?     │         (e.g., Adyen returns 200 with "Refused")
└─────────────────────┘
       │ No
       ▼
┌─────────────────────┐
│ Parse error body    │──Fail──▶ Create generic error
│ as JSON             │          with HTTP status mapping
└─────────────────────┘
       │ Success
       ▼
┌─────────────────────┐
│ Extract error code  │
│ from source_field   │
└─────────────────────┘
       │
       ▼
┌─────────────────────┐
│ Look up in          │──Found──▶ Return matched category
│ error_mappings      │
│ .categories         │
└─────────────────────┘
       │ Not found
       ▼
┌─────────────────────┐
│ Fall back to        │──Found──▶ Return HTTP-based category
│ http_errors mapping │
└─────────────────────┘
       │ Not found
       ▼
  Return "unknown"
```

---

## Two Types of Errors

### 1. Decline Responses (HTTP 2xx)

Some PSPs return HTTP 200 for declined transactions. The error is in the response body.

**Adyen example:** HTTP 200, `resultCode: "Refused"`, `refusalReasonCode: "24"` (CVC declined)

For these, the SDK should:
1. Map the status via `status_mappings` (e.g., "Refused" -> "declined")
2. Map the error code via `error_mappings` (e.g., "24" -> "cvc_declined")
3. Return a `TransactionResponse` with declined status and response code

### 2. HTTP Error Responses (4xx/5xx)

The PSP returns a non-success HTTP status with an error body.

**Checkout.com example:** HTTP 422, `{ "error_codes": ["card_number_invalid"] }`

For these, the SDK should:
1. Parse the error response body
2. Extract the error code from `error_mappings.source_field`
3. Look up the code in `error_mappings.categories`
4. If not found, fall back to `error_mappings.http_errors`
5. Throw a `TransactionError` with the unified `ErrorResponse`

---

## Error Mapping Algorithm

```python
def classify_error(error_code, http_status, mapping):
    """
    Classify a PSP error into a unified error category.

    Args:
        error_code: The PSP-specific error code (string or number)
        http_status: The HTTP status code
        mapping: The error_mappings section from the PSP mapping file

    Returns:
        Unified error category string
    """
    # Step 1: Search categories for this error code
    for category, config in mapping["categories"].items():
        if str(error_code) in [str(c) for c in config["codes"]]:
            return category

    # Step 2: Fall back to HTTP status mapping
    if str(http_status) in mapping.get("http_errors", {}):
        return mapping["http_errors"][str(http_status)]

    # Step 3: Default to unknown
    return "unknown"
```

---

## TransactionError Exception

Every generated SDK should define a `TransactionError` exception/error type that wraps an `ErrorResponse`:

```
class TransactionError extends Error:
    response: ErrorResponse

    constructor(response: ErrorResponse):
        super(format_error_message(response))
        self.response = response
```

The `ErrorResponse` contains:
- `error_codes`: List of `ResponseCode` objects (category + code)
- `provider_errors`: Raw error messages from the PSP
- `full_provider_response`: The complete PSP error response

---

## PSP-Specific Patterns

### Adyen

- **Declines via 200**: Adyen returns HTTP 200 for refused payments. Check `resultCode`.
- **Error code location**: `refusalReasonCode` field (numeric string)
- **Error message**: `refusalReason` field
- **HTTP errors**: Standard 401/403/422/500

```json
{
  "source_field": "refusalReasonCode",
  "categories": {
    "cvc_declined": { "codes": ["24"] },
    "expired_card": { "codes": ["6"] }
  }
}
```

### Checkout.com

- **Declines via 200**: Checkout.com can return HTTP 200 with `status: "Declined"` and a `response_code`.
- **Error code location**: Two sources — `response_code` (numeric) for declines, `error_codes` array for validation errors
- **HTTP errors**: 401/403/422/429/500

```json
{
  "source_field": "response_code",
  "categories": {
    "expired_card": { "codes": ["20054", "30033", "card_expired"] },
    "cvc_declined": { "codes": ["20082", "20124", "cvv_invalid"] }
  }
}
```

---

## Implementation Checklist

- [ ] Handle both decline responses (2xx) and HTTP errors (4xx/5xx)
- [ ] Extract error code from the path specified in `error_mappings.source_field`
- [ ] Search all categories for matching error codes
- [ ] Fall back to `http_errors` mapping when code not found in categories
- [ ] Default to "unknown" when no mapping matches
- [ ] Preserve the full PSP response in `ErrorResponse.full_provider_response`
- [ ] Include raw provider error messages in `ErrorResponse.provider_errors`
- [ ] Wrap errors in a `TransactionError` exception/error type
- [ ] Handle JSON parse failures gracefully (create generic error)
