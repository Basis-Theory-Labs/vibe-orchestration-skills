# /build-demo

Generate a web-based interactive demo app from an existing generated SDK and mapping files. The demo is written in the same language as the SDK.

## Usage

```
/build-demo [output-dir]
```

- `output-dir`: Where to write the demo app. Defaults to `./output/demo-app`.

## Process

### Step 1: Locate SDK, Discover PSPs, and Confirm

1. **Find the SDK.** Look in the `output/` directory for a generated SDK (the default output location of `/generate-sdk`).
   - Scan `output/*/` for any SDK directory. Look for the entry point file (e.g., `src/index.ts`, `src/main.py`, `src/lib.rs`, `src/main/`, etc.) to identify the language.
   - If exactly one is found, use it as `{sdk-path}`.
   - If multiple SDK directories are found, list them and ask the user which one to use.
   - If none are found, ask the user: *"I couldn't find a generated SDK in the output/ directory. Where did you generate it?"* Use the path they provide as `{sdk-path}`.
   - If no SDK exists anywhere, tell the user to run `/generate-sdk` first and stop.
2. **Identify the SDK language.** Determine the language from the SDK's file extensions and project structure (e.g., `.ts` = TypeScript, `.py` = Python, `.go` = Go, `.cs` = C#). The demo server will be written in this same language.
3. **Discover providers from the SDK.** Read the SDK's provider directory/module to find which provider classes are exported. The provider names determine which PSPs the demo supports — do NOT hardcode PSP names.
4. **Cross-reference with mapping files.** List `mappings/*.json` and match each mapping's `psp.name` against the discovered SDK providers. Only include PSPs that have both a mapping file and a generated provider class.
5. **Read each provider's config interface.** For each discovered provider, read its source file to extract the config field names and types. These must be used exactly as-is in the generated code (respecting the SDK's naming convention — e.g., camelCase for TypeScript, snake_case for Python).
6. Confirm the output directory, SDK language, and discovered PSPs with the user.

### Step 2: Load Mapping Data

For each included PSP (discovered in Step 1), read its `mappings/{psp}.json` and extract:

- **`psp.display_name`** — Tab label in the UI
- **`psp.name`** — Internal key
- **`authentication.config_fields`** — For `.env.example` generation (name, env_var, description, required)
- **`source_types`** — Which of the 5 source types are supported (check which keys exist)
- **`setup.required_env_vars`** and **`setup.optional_env_vars`** — For `.env.example`

The provider config interfaces (field names, types) were already extracted in Step 1.

### Step 3: Generate the Server

Generate an HTTP server in the **same language as the SDK**. Keep it minimal — use the language's standard library or the most common lightweight HTTP library. No heavy frameworks.

**Language-specific guidance:**

| SDK Language | Server Approach |
|---|---|
| TypeScript | Built-in `http` module, run with `tsx` |
| Python | Built-in `http.server` or Flask/FastAPI (minimal) |
| Go | Built-in `net/http` |
| C# | Minimal API (`WebApplication.CreateBuilder`) |
| Java/Kotlin | Built-in `com.sun.net.httpserver` or similar |
| Ruby | WEBrick or Sinatra (minimal) |
| Rust | `axum` or `actix-web` (minimal) |

Use your judgement for the most idiomatic minimal approach for the language. The goal is a single-file server (or as close to it as practical) with no unnecessary dependencies.

**Import the SDK** via relative path or local project reference from the demo output directory to `{sdk-path}`. Calculate the correct path based on where both directories are located.

**Read environment variables** at startup using the language's idiomatic approach (e.g., `process.env` in Node, `os.environ` in Python, `os.Getenv` in Go).

**Conditionally initialize providers** based on which env vars are set. For each PSP, check that its required env vars are present before creating the provider. This lets the demo work with partial configuration. The config field names in the provider constructor must match the SDK's interface exactly — read the provider source to get the correct names.

**Routes:**

| Method | Path | Description |
|--------|------|-------------|
| `GET` | `/` | Serve `index.html` from the same directory |
| `GET` | `/api/config` | Return JSON: available PSPs, BT status, test card defaults |
| `POST` | `/api/tokenize` | Body: `{ number, expiryMonth, expiryYear, cvc }` — create a BT card token |
| `POST` | `/api/token-intent` | Body: `{ number, expiryMonth, expiryYear, cvc }` — create a BT token intent |
| `POST` | `/api/authorize` | Body: `{ psp, amount, currency, source }` — call `client.authorize()` |
| `POST` | `/api/capture` | Body: `{ psp, transactionId, amount, currency }` — call `client.capture()` |
| `POST` | `/api/refund` | Body: `{ psp, transactionId, amount, currency }` — call `client.refund()` |
| `POST` | `/api/cancel` | Body: `{ psp, transactionId }` — call `client.cancel()` |
| `GET` | `/api/get?psp={psp}&id={id}` | Call `client.get()` |

**`POST /api/tokenize` implementation:**

Creates a Basis Theory card token by calling the BT API directly via HTTP (no BT SDK dependency):

```
POST https://api.basistheory.com/tokens
Header: BT-API-KEY: {BT_API_KEY env var}
Body:
{
  "type": "card",
  "data": {
    "number": "4111111111111111",
    "expiration_month": 3,
    "expiration_year": 2030,
    "cvc": "737"
  }
}
```

Returns `{ id, type, maskedNumber }` where `maskedNumber` = `•••• {last4}`. If `BT_API_KEY` is not set, return an error response.

**`POST /api/token-intent` implementation:**

Same pattern but posts to `https://api.basistheory.com/token-intents`.

**`/api/config` response shape:**
```json
{
  "providers": [
    {
      "name": "adyen",
      "displayName": "Adyen",
      "configured": true,
      "sourceTypes": ["raw_pan", "basis_theory_token", "processor_token"]
    }
  ],
  "basisTheory": {
    "configured": true
  },
  "testCards": {
    "adyen": { "number": "4111111111111111", "expiryMonth": "03", "expiryYear": "2030", "cvc": "737" },
    "stripe": { "number": "4242424242424242", "expiryMonth": "12", "expiryYear": "2030", "cvc": "123" },
    "checkout": { "number": "4242424242424242", "expiryMonth": "12", "expiryYear": "2029", "cvc": "100" }
  }
}
```

Test card data comes from what works in each PSP's sandbox. If a `demo` script or test file exists in the SDK, use those same card numbers.

**Error handling:** Wrap every API route in error handling. On error, return `{ error: true, message, details }` with appropriate status codes. If the error is a `TransactionError` (or equivalent), include the full response.

**Server listens on port 3000** (configurable via `PORT` env var). Log a startup message showing which PSPs are configured.

### Step 4: Generate `index.html`

Generate a single HTML file with inline `<style>` and `<script>` (no external deps, no build step). Use vanilla JS (no framework). This file is the same regardless of the SDK's language — it's always HTML/CSS/JS talking to the server's JSON API.

**Tab architecture — two kinds of tabs:**

The tab bar contains PSP tabs AND a Basis Theory utility tab. They share the same tab bar but switch between two different panel views:

```
[Basis Theory]  [Adyen]  [Stripe]  [Checkout]
```

- **PSP tabs** — Show the payment form (source type, fields, authorize, actions)
- **Basis Theory tab** — Show the token/intent creation tool (card fields, create buttons, token list)

The Basis Theory tab is always first. It is styled differently (e.g., orange accent) to visually distinguish it as a utility, not a PSP. If `BT_API_KEY` is not configured, grey it out like unconfigured PSPs.

**Layout when a PSP tab is selected (two-column):**

```
+----------------------------------------------------------+
|  PSP Orchestration Demo                                   |
+----------------------------------------------------------+
|  [BT]  [Adyen]  [Stripe]  [Checkout]                     |
+----------------------------------------------------------+
|  Payment Setup          |  Transaction Log                |
|                         |                                 |
|  Source: [raw_pan v]    |  1. authorize  [authorized]     |
|  Card:   [4111...]      |     id: Z8ZG...                 |
|  Expiry: [03] / [2030]  |  2. capture   [received]        |
|  CVC:    [737]          |     id: QDCV...                 |
|  Amount: [10.00] [USD]  |                                 |
|  [Authorize]            |                                 |
|                         +---------------------------------+
|  Actions                |  Raw Response                   |
|  [Get] [Capture]        |  {                              |
|  [Refund] [Cancel]      |    "pspReference": "...",       |
|                         |    "resultCode": "Authorised"   |
+-------------------------+---------------------------------+
```

**Layout when the Basis Theory tab is selected:**

```
+----------------------------------------------------------+
|  PSP Orchestration Demo                                   |
+----------------------------------------------------------+
|  [BT]  [Adyen]  [Stripe]  [Checkout]                     |
+----------------------------------------------------------+
|  Create Token           |  Created Tokens                 |
|                         |                                 |
|  Card:   [4111...]      |  Token                          |
|  Expiry: [03] / [2030]  |  01839f09-f5fc-4aa4  [copy]    |
|  CVC:    [737]          |  •••• 1111                      |
|                         |                                 |
|  [Create Token]         |  Token Intent                   |
|  [Create Token Intent]  |  a7db6419-badb-a94c  [copy]    |
|                         |  •••• 4242                      |
+-------------------------+---------------------------------+
```

**Features:**

1. **Tab Bar** — Basis Theory tab first (orange accent), then one tab per PSP from `/api/config`. Unconfigured tabs (PSPs without env vars, or BT without `BT_API_KEY`) are greyed out. On load, auto-select the first configured PSP tab (not the BT tab).

2. **Basis Theory Tab Panel** — When selected, replaces the payment form with a token creation tool:
   - **Card fields**: Number, expiry month, expiry year, CVC. Pre-filled with a default test card (`4242424242424242`, `12`, `2030`, `123`).
   - **[Create Token]** button — `POST /api/tokenize`, adds result to the token list
   - **[Create Token Intent]** button — `POST /api/token-intent`, adds result to the token list
   - **Created tokens list** (right column, replaces transaction log) — Each entry shows:
     - Label: "Token" or "Token Intent"
     - ID (truncated) with a **[copy]** button that copies the full ID to clipboard
     - Masked card number (e.g., `•••• 1111`)
     - Raw response viewable on click (uses the same JSON viewer)
   - The token list persists across tab switches so users can create a token, switch to a PSP tab, and paste it

3. **Source Type Dropdown (PSP tabs)** — Populated from the selected PSP's `sourceTypes`. Each source type shows only its native input fields:
   - `raw_pan`: Card number, expiry month, expiry year, CVC
   - `basis_theory_token`: Single text input for Token ID
   - `basis_theory_token_intent`: Single text input for Token Intent ID
   - `network_token`: DPAN, expiry month, expiry year, cryptogram, ECI
   - `processor_token`: Single text input for Processor Token ID

4. **Test Card Pre-fill** — When a PSP tab is selected and source is `raw_pan`, auto-fill card fields from the `/api/config` testCards data for that PSP.

5. **Amount Fields** — Amount (decimal, e.g., `10.00`) and currency (text input defaulting to `USD`). The UI multiplies by 100 before sending to the server (minor units).

6. **Authorize Button** — Sends `POST /api/authorize` with the source matching the selected type. Always enabled (user is responsible for filling in valid data).

7. **Action Buttons** — Get, Capture, Refund, Cancel. All disabled until an authorize succeeds. Each uses the stored transaction ID.

8. **Transaction Log (PSP tabs)** — Append-only list showing each operation with:
   - Operation name (authorize, get, capture, refund, cancel)
   - Status badge: green for success (`authorized`, `captured`, `received`, `refunded`, `voided`), red for failures (`declined`, `error`), yellow for pending/action_required
   - Truncated transaction ID
   - Click to show full raw response in the JSON viewer

9. **Raw Response Viewer** — `<pre>` block with JSON syntax highlighting (basic: strings in green, keys in blue, numbers in orange). Shared between PSP and BT tabs — shows the response for the selected log/token entry.

10. **Loading States** — Disable the clicked button and show a spinner while requests are in flight.

11. **Error Display** — On API error, add a red entry to the transaction log (or token list) showing the error message.

**Styling:**
- Clean, professional look with a monospace font for the JSON viewer
- Use CSS custom properties for the color scheme:
  - `--bg: #0a0a0a`, `--surface: #141414`, `--border: #262626`, `--text: #e5e5e5`, `--muted: #737373`
  - `--green: #22c55e`, `--red: #ef4444`, `--yellow: #eab308`, `--blue: #3b82f6`, `--orange: #f97316`
- Dark theme by default
- Basis Theory tab uses `--orange` as its accent color to differentiate from PSP tabs (which use `--blue`)
- Responsive: stack columns on narrow screens

### Step 5: Generate Project Files

Generate the appropriate project/dependency files for the SDK's language:

| SDK Language | Project File | Run Command |
|---|---|---|
| TypeScript | `package.json` with `tsx` dep, start script | `npm start` |
| Python | `requirements.txt` or `pyproject.toml` | `python server.py` |
| Go | `go.mod` with replace directive for local SDK | `go run .` |
| C# | `.csproj` with project reference to SDK | `dotnet run` |
| Java | `pom.xml` or `build.gradle` | `mvn exec:java` / `gradle run` |
| Ruby | `Gemfile` | `ruby server.rb` |
| Rust | `Cargo.toml` with path dependency to SDK | `cargo run` |

Keep it minimal — only include dependencies actually needed (the SDK + HTTP server if not using stdlib).

### Step 6: Generate `.env.example`

Auto-generate from all mapping files' `authentication.config_fields`. Group by PSP with comments:

```bash
# ── Basis Theory ───────────────────────────────────
# API key for tokenizing cards and proxy requests
# Get one at https://portal.basistheory.com
BT_API_KEY=

# ── Adyen ──────────────────────────────────────────
# Adyen API key from Customer Area -> Developers -> API credentials
ADYEN_API_KEY=
# Adyen merchant account name
ADYEN_MERCHANT_ACCOUNT=

# ── Stripe ─────────────────────────────────────────
# Stripe secret API key (sk_test_... for sandbox)
STRIPE_SECRET_KEY=

# ── Checkout.com ───────────────────────────────────
# ... etc
```

Use the `description` field from each config field as the comment. `BT_API_KEY` appears in multiple mappings — deduplicate it and place it first in its own "Basis Theory" section since it's used for both tokenization and proxy flows across all PSPs.

### Step 7: Install and Present

1. Run the language's dependency install command (e.g., `npm install`, `pip install -r requirements.txt`, `go mod tidy`) in the output directory
2. Check if `.env` already exists in the output directory — if not, check if `output/sample-app/.env` or similar exists and offer to copy it
3. Print a summary showing generated files and run instructions appropriate to the language

## Key Rules

- **PSP discovery is dynamic.** Read the SDK's provider exports to determine which PSPs are supported — do NOT hardcode PSP names like "adyen", "stripe", "checkout". The demo must work with whatever PSPs the SDK was generated for.
- **The demo language matches the SDK language.** If the SDK is Python, the server is Python. If the SDK is Go, the server is Go. The only exception is `index.html` which is always HTML/CSS/JS.
- The SDK path (`{sdk-path}`) is determined at runtime in Step 1. All imports and file reads must use this path.
- The server should be minimal — use the language's standard library HTTP server or the most common lightweight option. No heavy frameworks.
- The UI MUST be a single HTML file with inline CSS and JS — no build step, no external dependencies.
- Import/reference the SDK via relative path or local project reference from the demo output directory to `{sdk-path}`. Do not hardcode paths.
- Provider initialization must be conditional — only initialize PSPs whose required env vars are set. This lets the demo work with partial configuration.
- Test card numbers must match what each PSP's sandbox actually accepts. If a demo script or test file exists in the SDK, use those same card numbers.
- The amount in the UI is entered as a decimal (e.g., `10.00`) but sent to the SDK as minor units (e.g., `1000`). The UI handles the conversion.
- All generated files should have a comment indicating they were generated.
- The `source` object sent to the server must match the SDK's source type. The server maps between the JSON body format and the SDK types.
- The `/api/tokenize` and `/api/token-intent` endpoints call the Basis Theory API directly via HTTP — do NOT add a Basis Theory SDK dependency. The only header needed is `BT-API-KEY`.
- The tokenize/token-intent request body sends `expiration_month` and `expiration_year` as integers (not strings) to match the BT API contract.
- **The Basis Theory tab and PSP tabs are fully decoupled.** The BT tab is a standalone utility for creating tokens/intents. The PSP tabs consume token IDs as opaque strings in the `basis_theory_token` / `basis_theory_token_intent` source type fields. There is no automatic flow between them — the user copies a token ID from the BT tab and pastes it into a PSP tab.
- Each source type in the PSP tab shows ONLY its native fields. `basis_theory_token` shows a single token ID text input — it does NOT show card fields or a tokenize button. That's what the BT tab is for.
- The created tokens list in the BT tab persists in JS memory across tab switches so the user can reference them when they switch to a PSP tab.

## Example Interaction

**SDK found in output:**
```
User: /build-demo

Claude: I'll generate an interactive demo app for the PSP Orchestration SDK.

Found SDK at output/python-sdk/ (Python). Scanning providers...
- Providers: AdyenProvider, StripeProvider, CheckoutProvider
- Matching mappings: adyen.json, stripe.json, checkout.json

I'll generate a Python demo server to ./output/demo-app/. Sound good?

User: Yes

Claude: [reads mapping files + provider configs, generates server.py, index.html, requirements.txt, .env.example]
[runs pip install]

Demo app generated:
  server.py        — HTTP server (9 routes, 3 PSPs, BT tokenization)
  index.html       — Interactive UI (BT tab + PSP tabs)
  requirements.txt — Dependencies
  .env.example     — Credentials template (BT + 3 PSPs)

To run:
  cd output/demo-app
  cp .env.example .env   # fill in your credentials
  python server.py       # opens http://localhost:3000
```

**Multiple SDKs found:**
```
User: /build-demo

Claude: I found multiple SDKs in output/:
  1. output/typescript-sdk/
  2. output/python-sdk/

Which SDK should I use for the demo?

User: python-sdk

Claude: Found SDK at output/python-sdk/ (Python). Scanning providers...
```

**No SDK found:**
```
User: /build-demo

Claude: I couldn't find a generated SDK in the output/ directory.
Where did you generate the SDK?

User: It's at ~/projects/my-payment-sdk

Claude: Found SDK at ~/projects/my-payment-sdk/ (Go). Scanning providers...
- Providers: StripeProvider, BraintreeProvider
- Matching mappings: stripe.json, braintree.json

I'll generate a Go demo server to ./output/demo-app/. Sound good?
```
