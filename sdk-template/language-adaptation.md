# Language Adaptation Guide

How to adapt the unified payment SDK patterns to any programming language.

The SDK templates (`unified-types.md`, `client-pattern.md`, `error-handling.md`) describe *what* to generate. This guide describes *how* to make it idiomatic for the target language.

---

## Naming Conventions

Every language has a dominant casing style. Apply it consistently across all generated code.

| Convention | Typical Languages |
|---|---|
| `snake_case` | Python, Ruby, Rust, Elixir |
| `camelCase` | TypeScript, JavaScript, Java, Kotlin, Dart |
| `PascalCase` | C#, Go (exported), Swift |

**Questions to answer:**
- What casing does the language use for local variables? Fields? Methods? Types? Constants?
- Are acronyms cased differently (e.g., `HTTP` vs `Http` vs `http`)?
- Does the language distinguish exported vs unexported by casing (Go)?

Convert all unified model field names (`amount_value`, `source_type`, `holder_name`) to the target convention. The mapping file field names stay as-is — they're PSP API names, not SDK names.

---

## Type Systems

Languages vary widely in how they express types. Map the unified models accordingly.

### Enums

- **Full enum support** (Python, TypeScript, Java, C#, Rust, Swift, Kotlin): Use native enums with string values.
- **Iota/const blocks** (Go): Use a type alias + const block.
- **No enums** (older JS, Lua): Use a frozen object or module constants.

### Optionals

- **Nullable types** (TypeScript `?`, C# `?`, Kotlin `?`): Use the native nullable syntax.
- **Option/Maybe** (Rust `Option<T>`, Haskell `Maybe a`): Wrap optional fields.
- **Pointers** (Go `*string`): Use pointer types for optional fields.
- **None/nil** (Python, Ruby): Use `Optional[T]` type hints or just allow `None`.

### Generics

- The `metadata` field (`map<string, string>`) and `full_provider_response` (`object`) need language-appropriate generic/dynamic types.
- Use `Dict[str, str]`, `Record<string, string>`, `map[string]string`, `HashMap<String, String>`, etc.

---

## Data Models

Choose the most idiomatic construct for the language.

| Approach | When to Use |
|---|---|
| **Dataclasses / records** | Python (`@dataclass`), Kotlin (`data class`), Java 16+ (`record`) |
| **Structs** | Go, Rust, C, Swift |
| **Interfaces / type aliases** | TypeScript (request/response shapes) |
| **Classes with builders** | Java (pre-16), C# with complex construction |
| **Plain objects + type guards** | JavaScript without TypeScript |

**Questions to answer:**
- Does the language favor immutability by default? If so, make models immutable.
- Is there a builder pattern convention for objects with many optional fields?
- Should models have serialization/deserialization methods, or does the language handle that via reflection or codegen?
- Does the language separate "data transfer" types from "behavior" types?

---

## Error Handling

The unified model defines a `TransactionError` that wraps an `ErrorResponse`. Adapt to the language's error paradigm.

| Paradigm | Languages | Approach |
|---|---|---|
| **Exceptions** | Python, Java, C#, Ruby, Kotlin | `TransactionError` extends the base exception class |
| **Result types** | Rust (`Result<T, E>`), Swift (`throws`), Kotlin (`Result<T>`) | Return `Result<TransactionResponse, TransactionError>` |
| **Error returns** | Go (`value, error`) | Return `(*TransactionResponse, error)` tuples |
| **Either/union** | TypeScript, Haskell | Return a discriminated union or Either type |

Regardless of paradigm, the error must carry the full `ErrorResponse` (unified codes + provider errors + raw response).

---

## HTTP Clients

Choose the standard or most widely-adopted HTTP library for the language.

**Questions to answer:**
- Does the language have a built-in HTTP client (Go `net/http`, Python `urllib`)?
- Is there a dominant ecosystem choice (Python `requests`/`httpx`, Java `OkHttp`, Rust `reqwest`)?
- Does the HTTP client support async natively, or only sync?
- How does the client handle JSON serialization/deserialization?

The provider needs two HTTP paths:
1. **Direct requests** — standard HTTP to the PSP's API
2. **Proxy requests** — routed through the Basis Theory proxy (different URL, extra headers)

Both paths use the same payload; only the URL and headers differ.

### Content-Type Handling

PSPs use different content types. The mapping file's `psp.content_type` field specifies which format is used.

| Content-Type | PSPs | Notes |
|---|---|---|
| `application/json` | Adyen, Checkout.com | Standard JSON body |
| `application/x-www-form-urlencoded` | Stripe | Requires flattening nested objects |

**Form-encoded request handling:**

When `content_type` is `application/x-www-form-urlencoded`, the HTTP layer must:
1. Flatten nested objects to bracket notation: `{ "source": { "type": "card" } }` becomes `source[type]=card`
2. Flatten arrays with indexed brackets: `{ "payment_method_types": ["card"] }` becomes `payment_method_types[0]=card`
3. URL-encode the resulting key-value pairs
4. Set the `Content-Type` header to `application/x-www-form-urlencoded`

This flattening must be recursive — deeply nested objects like `payment_method_options[card][three_d_secure][cryptogram]` must be supported.

---

## Async Patterns

Match the language's concurrency model.

| Pattern | Languages |
|---|---|
| `async/await` | Python, TypeScript, JavaScript, C#, Rust, Kotlin, Swift |
| Goroutines + channels | Go |
| Futures / promises | Java (`CompletableFuture`), Scala (`Future`) |
| Callbacks | Older Node.js patterns (avoid if possible) |
| Synchronous | Ruby, PHP (typically) |

If the language supports both sync and async, prefer async for the provider (HTTP calls are I/O-bound). Consider whether a sync wrapper is needed.

---

## Project Layout

Follow the language's standard project structure.

**Questions to answer:**
- Does the language use one-class-per-file (Java, C#) or modules with multiple types (Python, TypeScript, Go)?
- Is there a standard directory convention (Go flat packages, Java deep package hierarchies)?
- Where do provider implementations go — same package, sub-package, or separate module?

**General layout:**
```
{output-dir}/
  {models-file}         # All unified types, enums, request/response models
  {client-file}         # Provider interface + PaymentClient wrapper
  {providers}/          # One file per PSP (or inline if language prefers flat)
    {psp-name}{ext}     # e.g., adyen.py, adyen.go, Adyen.java
  {error-file}          # TransactionError (if separate from models)
  {package-config}      # Language-specific manifest
```

Flatten or nest based on what's idiomatic. A Go SDK might put everything in one package. A Java SDK might use `com.example.payments.providers` sub-packages.

---

## Package Scaffolding

Every language has a manifest or config file. Generate it.

| Language | Manifest | Extras |
|---|---|---|
| Python | `pyproject.toml` or `setup.py` | `__init__.py` for packages, `py.typed` marker |
| TypeScript | `package.json` | `tsconfig.json` |
| Go | `go.mod` | No extra files needed |
| Java | `pom.xml` or `build.gradle` | Package directories matching namespace |
| Rust | `Cargo.toml` | `lib.rs` entry point |
| C# | `.csproj` | Namespace matching directory structure |

Include the HTTP client dependency in the manifest. Don't include test frameworks or dev dependencies unless asked.
