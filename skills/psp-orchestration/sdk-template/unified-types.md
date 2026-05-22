# Unified Types

This document defines all enums, request models, and response models for generated SDKs. Every generated SDK must implement these types, adapting naming conventions to the target language.

---

## Enums

### SourceType

How the card data is being provided.

| Value | Description |
|---|---|
| `raw_pan` | Raw card number, expiry, CVC |
| `basis_theory_token` | Stored Basis Theory token |
| `basis_theory_token_intent` | Ephemeral Basis Theory token intent |
| `network_token` | Network token (DPAN) with cryptogram |
| `processor_token` | Token stored with the PSP |

### TransactionStatus

Unified status of a payment.

| Value | Description |
|---|---|
| `authorized` | Payment authorized, funds held |
| `declined` | Payment refused by PSP or issuer |
| `error` | Technical error occurred |
| `cancelled` | Payment cancelled/voided |
| `pending` | Awaiting completion |
| `action_required` | Additional action needed (e.g., 3DS challenge) |
| `partially_authorized` | Only part of the amount was authorized |
| `captured` | Funds captured |
| `refunded` | Payment refunded |
| `voided` | Payment voided before capture |
| `received` | Payment received, processing |

### ErrorCategory

Top-level error classification.

| Value | Description |
|---|---|
| `authentication_error` | API key or auth issues |
| `payment_method_error` | Card-related issues |
| `processing_error` | PSP or acquirer processing issues |
| `validation_error` | Request validation failures |
| `fraud_decline` | Fraud-related declines |
| `unknown` | Unclassified errors |

### ErrorCode

Specific error classifications within categories.

| Value | Category | Description |
|---|---|---|
| `card_declined` | processing_error | Generic card decline |
| `insufficient_funds` | payment_method_error | Not enough balance |
| `expired_card` | payment_method_error | Card is expired |
| `invalid_card` | payment_method_error | Invalid card number or details |
| `cvc_declined` | payment_method_error | CVC check failed |
| `blocked_card` | payment_method_error | Card is blocked or restricted |
| `fraud_detected` | fraud_decline | Suspected fraud |
| `three_ds_failed` | authentication_error | 3DS authentication failed |
| `issuer_unavailable` | processing_error | Card issuer is unreachable |
| `not_supported` | processing_error | Transaction type not supported |
| `acquirer_error` | processing_error | Acquirer-side error |
| `pin_error` | payment_method_error | PIN required or incorrect |
| `cancelled_by_shopper` | processing_error | Customer cancelled |
| `avs_declined` | processing_error | Address verification failed |
| `authentication_error` | authentication_error | Auth required or failed |
| `validation_error` | validation_error | Invalid request parameters |
| `rate_limit` | processing_error | Too many requests |
| `psp_error` | processing_error | PSP internal error |
| `unknown` | unknown | Unrecognized error |

### RecurringType

Type of recurring payment.

| Value | Description |
|---|---|
| `one_time` | Single payment, no storage |
| `card_on_file` | Cardholder-initiated with stored card |
| `subscription` | Merchant-initiated recurring |
| `unscheduled` | Merchant-initiated, non-fixed schedule |

---

## Request Models

### Amount

| Field | Type | Required | Description |
|---|---|---|---|
| `value` | integer | Yes | Amount in minor units (e.g., 1000 = $10.00) |
| `currency` | string | Yes | ISO 4217 currency code (e.g., "USD") |

### Source

| Field | Type | Required | Description |
|---|---|---|---|
| `type` | SourceType | Yes | How the card data is provided |
| `id` | string | Yes | Token ID, source ID, or card number depending on type |
| `store_with_provider` | boolean | No | Whether to store with the PSP for future use |
| `holder_name` | string | No | Cardholder name |
| `number` | string | No | Card number (raw_pan and network_token only) |
| `expiry_month` | string | No | Expiry month (raw_pan and network_token only) |
| `expiry_year` | string | No | Expiry year (raw_pan and network_token only) |
| `cvc` | string | No | CVC (raw_pan only) |
| `cryptogram` | string | No | Network token cryptogram (network_token only) |
| `eci` | string | No | ECI indicator (network_token only) |

### Address

| Field | Type | Required | Description |
|---|---|---|---|
| `address_line1` | string | No | Street address |
| `address_line2` | string | No | Unit, suite, etc. |
| `city` | string | No | City |
| `state` | string | No | State or province |
| `zip` | string | No | Postal code |
| `country` | string | No | ISO 3166-1 alpha-2 country code |

### Customer

| Field | Type | Required | Description |
|---|---|---|---|
| `reference` | string | No | Your internal customer ID |
| `first_name` | string | No | First name |
| `last_name` | string | No | Last name |
| `email` | string | No | Email address |
| `address` | Address | No | Billing address |

### StatementDescription

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | No | Statement descriptor name |
| `city` | string | No | Statement descriptor city |

### ThreeDS

| Field | Type | Required | Description |
|---|---|---|---|
| `eci` | string | No | Electronic Commerce Indicator |
| `authentication_value` | string | No | CAVV / authentication value |
| `version` | string | No | 3DS protocol version |
| `ds_transaction_id` | string | No | Directory server transaction ID |
| `directory_status_code` | string | No | Directory server status |
| `authentication_status_code` | string | No | Authentication status |
| `challenge_cancel_reason_code` | string | No | Challenge cancellation reason |
| `challenge_preference_code` | string | No | Challenge preference indicator |

### TransactionRequest

| Field | Type | Required | Description |
|---|---|---|---|
| `amount` | Amount | Yes | Payment amount |
| `source` | Source | Yes | Payment source |
| `reference` | string | No | Your unique payment reference |
| `merchant_initiated` | boolean | No | Whether this is merchant-initiated (default: false) |
| `type` | RecurringType | No | Recurring payment type |
| `customer` | Customer | No | Customer information |
| `statement_description` | StatementDescription | No | Statement descriptor |
| `three_ds` | ThreeDS | No | 3D Secure data |
| `previous_network_transaction_id` | string | No | Prior network transaction ID for recurring |
| `metadata` | map<string, string> | No | Custom key-value metadata |

### RefundRequest

| Field | Type | Required | Description |
|---|---|---|---|
| `original_transaction_id` | string | Yes | PSP transaction ID to refund |
| `reference` | string | Yes | Your unique refund reference |
| `amount` | Amount | Yes | Refund amount |
| `reason` | string | No | Refund reason |

---

## Response Models

### TransactionStatusResponse

| Field | Type | Description |
|---|---|---|
| `code` | TransactionStatus | Unified status |
| `provider_code` | string | Raw status string from the PSP |

### ResponseCode

| Field | Type | Description |
|---|---|---|
| `category` | ErrorCategory | Error category |
| `code` | ErrorCode | Specific error code |

### ProvisionedSource

| Field | Type | Description |
|---|---|---|
| `id` | string | Token ID stored with the PSP |

### TransactionSource

| Field | Type | Description |
|---|---|---|
| `type` | SourceType | Source type used |
| `id` | string | Source ID used |
| `provisioned` | ProvisionedSource? | PSP-stored token (if the PSP provisioned one) |

### TransactionResponse

| Field | Type | Description |
|---|---|---|
| `id` | string | PSP's transaction ID |
| `reference` | string | Your payment reference |
| `amount` | Amount | Authorized amount |
| `status` | TransactionStatusResponse | Unified + provider status |
| `response_code` | ResponseCode | Error classification |
| `source` | TransactionSource | Source details + provisioned token |
| `network_transaction_id` | string? | Network transaction reference for recurring |
| `full_provider_response` | object | Raw PSP response |
| `created_at` | datetime | Transaction timestamp |

### RefundResponse

| Field | Type | Description |
|---|---|---|
| `id` | string | PSP's refund ID |
| `reference` | string | Your refund reference |
| `amount` | Amount | Refunded amount |
| `status` | TransactionStatusResponse | Refund status |
| `refunded_transaction_id` | string? | Original transaction ID |
| `full_provider_response` | object | Raw PSP response |
| `created_at` | datetime | Refund timestamp |

### ErrorResponse

| Field | Type | Description |
|---|---|---|
| `error_codes` | list<ResponseCode> | Unified error codes |
| `provider_errors` | list<string> | Raw error messages from the PSP |
| `full_provider_response` | object | Raw PSP error response |
