# /generate-sdk

Generate a working payment SDK from mapping files in any programming language.

## Usage

```
/generate-sdk [language] [psp-names] [output-dir]
```

All arguments are optional — the skill will prompt for anything missing.

- `language`: Target language (e.g., `python`, `typescript`, `go`, `java`, `csharp`)
- `psp-names`: Comma-separated PSP names (e.g., `adyen,checkout`). Defaults to all available mappings.
- `output-dir`: Where to write the generated code. Defaults to `./output/{language}-sdk`.

## Process

### Step 1: Interactive Setup

1. List available mapping files from `mappings/*.json`
2. If `language` not provided, ask the user which language to generate
3. If `psp-names` not provided, ask which PSPs to include (or all)
4. Confirm output directory

If the user wants a PSP that doesn't have a mapping yet, offer to run `/generate-mapping-for-psp` first.

### Step 2: Load Resources

Read all required files:

```
# Schema and templates
schema/psp-mapping.schema.json
sdk-template/unified-types.md
sdk-template/client-pattern.md
sdk-template/error-handling.md
sdk-template/language-adaptation.md

# Selected mappings
mappings/{psp-name}.json (for each selected PSP)
```

### Step 3: Generate Code

Generate the following files, using `sdk-template/language-adaptation.md` to apply idiomatic conventions for the target language:

#### 3a. Unified Models

Generate from `sdk-template/unified-types.md`:
- All enums (SourceType, TransactionStatus, ErrorCategory, RecurringType)
- All request models (Amount, Source, Customer, ThreeDS, TransactionRequest, RefundRequest)
- All response models (TransactionResponse, RefundResponse, ErrorResponse)
- TransactionError exception/error type

Apply the language adaptation guide for naming conventions, type system choices, optionals, and data model constructs.

#### 3b. Provider per PSP

For each selected PSP mapping, generate a provider class following `sdk-template/client-pattern.md`:

1. **Static lookup tables** generated from the mapping:
   - `STATUS_MAPPINGS` from `status_mappings`
   - `ERROR_CATEGORY_LOOKUP` from `error_mappings.categories` (inverted: code -> category)
   - `HTTP_ERROR_MAPPINGS` from `error_mappings.http_errors`
   - `RECURRING_MAPPINGS` from `recurring.unified_to_psp`
   - `THREE_DS_REQUEST_FIELDS` from `three_ds.request_fields`

2. **Constructor** accepting all `authentication.config_fields`

3. **Source type transforms** from `source_types.{type}.request_transform`:
   - Use the mapping's dot-notation keys to build nested request objects
   - For BT token types, generate the expression template strings
   - For network tokens, use the PSP-specific field paths

4. **Operation methods** (authorize, capture, get, refund, cancel):
   - HTTP method and path from `operations.{op}.method` and `operations.{op}.path`
   - Request building from `operations.{op}.request_mapping`
   - Response parsing from `operations.{op}.response_mapping`
   - Status transformation via `status_mappings`
   - Error classification via `error_mappings`

5. **HTTP layer** with:
   - Direct requests for raw_pan, network_token, processor_token
   - Basis Theory proxy requests for basis_theory_token, basis_theory_token_intent
   - Idempotency key support

#### 3c. Client Wrapper

Generate a unified client from `sdk-template/client-pattern.md`:
- Accepts a list of providers at construction
- Routes operations by provider name
- Exposes the same authorize/capture/get/refund/cancel interface

#### 3d. Project Structure

Generate appropriate project scaffolding for the language. Use the project layout and package scaffolding sections of `sdk-template/language-adaptation.md` to determine:
- File organization (flat vs nested, one-class-per-file vs modules)
- Package manifest and config files
- Entry point and export structure

### Step 4: Write Output

1. Create the output directory structure
2. Write all generated files
3. Present a summary of what was generated:
   - Number of files
   - PSPs included
   - Key features (source types supported, operations, error handling)
4. Provide a usage example in the target language

## Key Rules

- Generated code should be idiomatic for the target language — not a mechanical translation. Use `sdk-template/language-adaptation.md` to guide language-specific decisions.
- All lookup tables (status mappings, error categories, etc.) are generated as static data from the mapping files.
- The provider's `authorize` method must handle all 5 source types via conditional branching based on `source.type`.
- Error handling must follow the patterns in `sdk-template/error-handling.md`.
- Every generated file should have a comment indicating it was generated from a mapping file.
- Do NOT hardcode PSP-specific values in the client wrapper — everything PSP-specific lives in the provider.

## Example Interaction

```
User: /generate-sdk typescript adyen,checkout ./my-payments-sdk

Claude: I'll generate a TypeScript SDK with Adyen and Checkout.com support.

Loading:
- mappings/adyen.json (42 status mappings, 17 error categories)
- mappings/checkout.json (12 status mappings, 18 error categories)
- SDK templates for TypeScript

Generating to ./my-payments-sdk/:
  models.ts          - Unified types (14 interfaces, 4 enums)
  client.ts          - PaymentClient wrapper
  providers/adyen.ts - Adyen provider (6 operations, 5 source types)
  providers/checkout.ts - Checkout.com provider (6 operations, 5 source types)
  providers/index.ts - Provider exports
  index.ts           - Package entry point
  package.json       - Package configuration

Usage:
  import { PaymentClient, AdyenProvider, SourceType } from './my-payments-sdk';

  const client = new PaymentClient([
    new AdyenProvider({ apiKey: '...', merchantAccount: '...', isTest: true })
  ]);

  const response = await client.authorize('adyen', {
    amount: { value: 1000, currency: 'USD' },
    source: { type: SourceType.BasisTheoryToken, id: 'tok_...' },
    reference: 'order-123'
  });
```
