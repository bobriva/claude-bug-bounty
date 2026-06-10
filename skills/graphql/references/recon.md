# Recon Reference — Endpoint Discovery & Fingerprinting

## 1. Endpoint Discovery

### Common Paths (Try All)

```
# Standard
/graphql
/graphiql
/graphql/console
/graphql/playground

# API-prefixed
/api/graphql
/api/graphiql
/v1/graphql
/v2/graphql
/v3/graphql
/v1/api/graphql

# Alternative formats
/graphql.php
/graphql.json
/graphql/api
/graphql/graphql
/query
/gql
/data

# Explorer / IDE interfaces (dev tools left in prod)
/v1/explorer
/graphiql.php
/console
/playground
/altair
/api/explorer
/graphql/explorer

# Hasura-specific
/v1/graphql
/v2/graphql
/v1/relay

# WPGraphQL (WordPress)
/?graphql
/graphql (also via POST to homepage)
```

### Discovery with Tools

```bash
# ffuf endpoint scan
ffuf -w /usr/share/seclists/Discovery/Web-Content/graphql.txt \
  -u https://target.com/FUZZ \
  -mc 200,301,302,400,401,403 \
  -H "Content-Type: application/json"

# Look in JS source files — GraphQL endpoints often hardcoded
curl -s https://target.com/ | grep -oP '"[^"]*graphql[^"]*"'
curl -s https://target.com/static/main.js | grep -oP 'https?://[^\s"]+graphql[^\s"]*'

# Check .js bundles for endpoint leaks
# In Burp: search across all JS responses for "graphql" or "__typename"
```

### Universal Probe (Confirm GraphQL Exists)

Send to any suspected path — if any of these work, you found it:

```http
POST /graphql HTTP/1.1
Host: target.com
Content-Type: application/json

{"query":"{__typename}"}
```

**Positive responses:**
```json
{"data":{"__typename":"Query"}}      ← confirmed
{"data":{"__typename":"RootQuery"}}  ← confirmed
{"errors":[{"message":"..."}]}       ← confirmed (error = GraphQL is there)
```

**Negative (not GraphQL):**
```
404, 200 with HTML, plain text error
```

### GET Method Test

```bash
# Test GET support (important for CSRF)
curl "https://target.com/graphql?query={__typename}"

# With URL encoding
curl "https://target.com/graphql?query=%7B__typename%7D"
```

### Form-Encoded Test (CSRF prerequisite)

```http
POST /graphql HTTP/1.1
Host: target.com
Content-Type: application/x-www-form-urlencoded

query=%7B__typename%7D
```

---

## 2. Engine Fingerprinting

Knowing the engine tells you which CVEs apply, which bypasses work, and how the server behaves.

### Automated Fingerprinting

```bash
# graphw00f — best tool for this
pip install graphw00f
graphw00f -d -t https://target.com/graphql

# Output example:
# [*] Checking if GraphQL is available at https://target.com/graphql...
# [*] Found GraphQL at https://target.com/graphql
# [*] Attempting to fingerprint...
# [+] Detected: Apollo Server (NodeJS)
```

### Manual Fingerprinting by Error Messages

Send invalid queries and read the error text:

```http
POST /graphql HTTP/1.1
Content-Type: application/json

{"query": "query { thisFieldDefinitelyDoesNotExist }"}
```

**Engine identification by error format:**

| Error Message Pattern | Engine |
|---|---|
| `"Cannot query field \"X\" on type \"Y\""` | Apollo Server (Node.js) |
| `Field 'X' doesn't exist on type 'Y'` | GraphQL Ruby |
| `Cannot query field 'X' on type 'Y'.` | graphql-core (Python) |
| `Unknown field.` (minimal) | Hasura |
| `field X is undefined` | Graphene (Python) |
| `error parsing 'query'` | GraphQL PHP |
| `Syntax Error` with line/column | GraphQL.js reference impl |

### Fingerprint by Response Headers

```
Server: nginx + X-Powered-By: Express → likely Apollo (Node)
Server: gunicorn → likely Graphene (Python)
X-Hasura-* headers → Hasura
Content-Type: application/json → standard
Content-Type: application/graphql+json → newer spec
```

### Why Engine Matters

```
Apollo Server < 3.x → introspection on by default in dev
Apollo Server >= 4.x → hideSchemaDetailsFromClientErrors option
Hasura → powerful admin endpoint at /v1/metadata (if exposed = Critical)
GraphQL Ruby → different introspection bypass techniques
graphql-java → known N+1 issues, different error format
WPGraphQL → WordPress-specific attack surface (user enumeration, CPT exposure)
```

---

## 3. Authentication Scope Testing

```bash
# Test 1: Unauthenticated introspection
curl -s -X POST https://target.com/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{__schema{queryType{name}}}"}' | jq .

# Test 2: Unauthenticated __typename
curl -s https://target.com/graphql?query={__typename}

# Test 3: Compare authenticated vs unauthenticated schema
# → Different results = auth enforced
# → Same results = auth not enforced on schema discovery
```

---

## 4. WebSocket / Subscription Endpoint Detection

GraphQL subscriptions use WebSocket — separate endpoint, separate auth rules.

```bash
# Common subscription endpoints
/graphql/subscriptions
/subscriptions
/ws
/graphql-ws

# Test with wscat
npm install -g wscat
wscat -c wss://target.com/graphql/subscriptions \
  -H "Authorization: Bearer TOKEN"

# Send subscription init
{"type":"connection_init","payload":{}}
{"type":"start","id":"1","payload":{"query":"subscription { newMessage { content } }"}}
```

**Why subscriptions matter:**
- Often have weaker auth than regular queries
- May expose real-time data without authorization check
- Persistent connections can be used for DoS
- Sometimes bypass rate limiting entirely
