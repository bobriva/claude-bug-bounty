# GraphQL-Specific Testing

GraphQL exposes a single endpoint (commonly `/graphql`) but the entire schema and resolver logic sit behind it — meaning the OWASP API Top 10 categories still all apply, they just manifest differently than in REST. Detect GraphQL first: a JSON body with curly-brace query syntax, a response shaped like `{"errors":[{"message":...}]}`, or an endpoint literally named `/graphql` or `/api/graphql`.

## 1. Introspection

```graphql
{ __schema { types { name fields { name type { name } } } } }
```

If this returns the full schema, you have a complete map of every query, mutation, type, and field — extremely useful for guiding the rest of testing, but **introspection being enabled is informational on its own, not a reportable bug by itself.** It only becomes interesting once it leads you to something exploitable.

If introspection is disabled, schemas can sometimes still be reconstructed via **field suggestion** (deliberately malformed queries trigger "did you mean X" error responses that leak field names one at a time) — tools like `Clairvoyance` automate this reconstruction.

Useful tooling once you have a schema (introspected or reconstructed):
- **GraphQL Voyager** — visualizes the schema's type relationships, much easier to read than raw JSON for a large schema.
- **InQL** (Burp extension or CLI) — generates ready-to-send query/mutation templates from the schema, and supports schema discovery techniques even when introspection is off.

## 2. Missing field-level authorization (the GraphQL flavor of BOLA/BOPLA)

A common pattern: the schema has both a scoped query that correctly filters by the caller's identity, and a generic `node(id:)` resolver that fetches by global ID without re-checking ownership.

```graphql
# Scoped query — correctly returns only the caller's own data
{ user(id: 1) { name email } }

# Generic node resolver — sometimes bypasses the same check
{ node(id: "dXNlcjoy") { ... on User { email phoneNumber ssn } } }
```

GraphQL global IDs are frequently base64-encoded composites like `Type:numericId` — decode them, and you can often construct IDs for other types/objects directly rather than needing to enumerate them. Test every object type the schema exposes through `node()` or any other generic-by-ID resolver, not just the obvious `User` type.

This is the single highest-value GraphQL-specific thing to test — it converts straight into a BOLA finding with the same proof requirements (two accounts, request/response showing cross-account access).

## 3. Batching and aliasing abuse (rate-limit and brute-force bypass)

GraphQL lets a client send many operations in a single HTTP request, which can sneak past rate limiting implemented at the HTTP-request level rather than the operation level.

```json
[
  {"query": "{ login(email: \"user@test.com\", password: \"pass1\") }"},
  {"query": "{ login(email: \"user@test.com\", password: \"pass2\") }"}
]
```

Or, using **aliases** within a single query to repeat the same field with different arguments:

```graphql
{
  a1: login(email: "user@test.com", password: "pass1")
  a2: login(email: "user@test.com", password: "pass2")
  a3: login(email: "user@test.com", password: "pass3")
}
```

If you can send hundreds of password/OTP guesses inside one HTTP request and the rate limiter only counted requests, this is a real bypass — confirm it by actually completing a brute force in a controlled, low-impact way (e.g., on your own test account's password) rather than just observing that the batch was accepted.

## 4. Depth/complexity-based denial of service

Deeply nested or recursive queries (querying a type that relates back to itself, many levels deep) can force the server to do exponential work for a small request.

```graphql
{
  user(id: 1) {
    friends { friends { friends { friends { friends { name } } } } } }
  }
}
```

Confirm via response time / resource impact, not just "the query was accepted" — pair this with the "Unrestricted Resource Consumption" validation bar in `vulnerability-classes.md`.

## 5. Injection inside resolver arguments

Resolvers ultimately call the same backend code REST endpoints would — so SQL/NoSQL injection, command injection, and path traversal are all still possible if a resolver passes an argument into a query/shell call unsanitized. Test resolver string arguments the same way you'd test a REST query parameter.

## What makes a GraphQL finding confirmed, not theoretical

- Introspection alone → not a finding.
- A `node()` or similar generic resolver returning another user's actual sensitive fields, demonstrated with two accounts → confirmed BOLA-equivalent finding.
- Batching "accepted" → not a finding; batching used to actually complete a brute force or rate-limit bypass with observable effect → confirmed.
- A deep query "running slowly" once → weak; demonstrated resource exhaustion (server degradation, measurable cost) → confirmed.
