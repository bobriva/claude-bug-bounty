# Schema Recovery Reference

## 1. Standard Introspection

### Minimal Probe (Test if introspection works)

```json
{"query":"{__schema{queryType{name}}}"}
```

### Full Schema Dump (Use this when introspection is open)

```json
{
  "query": "query IntrospectionQuery { __schema { queryType { name } mutationType { name } subscriptionType { name } types { ...FullType } directives { name description locations args { ...InputValue } } } } fragment FullType on __Type { kind name description fields(includeDeprecated: true) { name description args { ...InputValue } type { ...TypeRef } isDeprecated deprecationReason } inputFields { ...InputValue } interfaces { ...TypeRef } enumValues(includeDeprecated: true) { name description isDeprecated deprecationReason } possibleTypes { ...TypeRef } } fragment InputValue on __InputValue { name description type { ...TypeRef } defaultValue } fragment TypeRef on __Type { kind name ofType { kind name ofType { kind name ofType { kind name ofType { kind name ofType { kind name ofType { kind name ofType { kind name } } } } } } } }"
}
```

**In Burp:** Right-click → Extensions → InQL → Send to InQL Scanner
→ Automatically generates query/mutation templates for every type.

### Quick Schema Overview Queries

```graphql
# List all query types
{ __schema { queryType { fields { name description } } } }

# List all mutations
{ __schema { mutationType { fields { name description args { name type { name kind } } } } } }

# List all types
{ __schema { types { name kind description } } }

# Get specific type details
{ __type(name: "User") { fields { name type { name kind ofType { name } } } } }
```

---

## 2. Introspection Bypass Techniques

### Bypass A: Newline Injection (Most Reliable)

Many implementations block `__schema` with regex. GraphQL ignores whitespace — regex doesn't.

```json
{"query": "{\n  __schema\n  {\n    queryType {\n      name\n    }\n  }\n}"}
```

Or with raw newlines in the query body:
```
{
  __schema
  {queryType{name}}
}
```

### Bypass B: Fragment-Based Introspection

```graphql
query {
  __schema {
    ...schemaFields
  }
}

fragment schemaFields on __Schema {
  queryType { name }
  types { name kind }
}
```

### Bypass C: GET Method (Introspection blocked on POST only)

```bash
curl "https://target.com/graphql?query={__schema{queryType{name}}}"

# URL encoded
curl "https://target.com/graphql?query=%7B__schema%7BqueryType%7Bname%7D%7D%7D"
```

### Bypass D: Comma Injection (Regex escape)

```json
{"query":"{__schema,{queryType{name}}}"}
```

### Bypass E: Special Character After __schema

```json
{"query":"{__schema\u0009{queryType{name}}}"}
```
(Unicode tab character — some WAFs miss this)

### Bypass F: Inline Fragment

```graphql
query {
  ...on Query {
    __schema {
      queryType { name }
    }
  }
}
```

### Bypass G: Apollo Sandbox / Studio

If the target allows requests from `https://studio.apollographql.com`, introspection may be allowed from that origin only:

```http
POST /graphql HTTP/1.1
Origin: https://studio.apollographql.com
```

### Bypass H: X-HTTP-Method-Override

```http
POST /graphql HTTP/1.1
X-HTTP-Method-Override: GET
Content-Type: application/json

{"query":"{__schema{queryType{name}}}"}
```

---

## 3. Schema Recovery When Introspection Is Fully Blocked

### Method A: Field Suggestions (Clairvoyance)

GraphQL returns "Did you mean X?" errors — this leaks valid field names.

**Manual test:**
```json
{"query":"{usr{id}}"}
```
Response might be: `"Did you mean \"user\"?"` → `user` is a valid field.

**Automated with Clairvoyance:**
```bash
# Install
pip3 install clairvoyance

# Basic run
clairvoyance https://target.com/graphql -o schema.json

# With auth
clairvoyance https://target.com/graphql \
  -H "Authorization: Bearer TOKEN" \
  -o schema.json

# With custom wordlist (use escape.tech GraphQL wordlist for best results)
clairvoyance https://target.com/graphql \
  -w graphql-wordlist.txt \
  -o schema.json

# Download escape.tech wordlist
curl -s https://raw.githubusercontent.com/Escape-Technologies/graphql-wordlist/main/wordlist.txt \
  -o graphql-wordlist.txt
```

**After Clairvoyance:** Import schema.json into GraphQL Voyager for visual map.

### Method B: GraphQuail Burp Extension (Passive)

- Install GraphQuail from BApp Store
- Browse the target application normally
- Extension passively captures every GraphQL request and reconstructs schema
- View reconstructed schema in GraphiQL interface
- No introspection queries needed — reads real traffic

### Method C: JS Source / Network Traffic Mining

```bash
# Search compiled JS for GraphQL queries
curl -s https://target.com/static/main.js | grep -oP 'gql`[^`]+`'
curl -s https://target.com/static/main.js | grep -oP '"query":\s*"[^"]+"'

# In Burp: Target → Site map → Search → "query" in response bodies
# Also: look for .graphql or .gql files in source maps
```

### Method D: Error-Driven Enumeration

```bash
# Probe with SecLists GraphQL wordlist
ffuf -u https://target.com/graphql \
  -X POST \
  -H "Content-Type: application/json" \
  -d '{"query":"{FUZZ{id}}"}' \
  -w /usr/share/seclists/Fuzzing/GraphQL/graphql-wordlist.txt \
  -mc 200 \
  -fr "Cannot query field" \
  -fc 404
```

---

## 4. Schema Analysis — What to Hunt

Once you have the schema (full or partial), analyze in this order:

### Priority 1: Auth-Related Mutations

```
Look for:
- login, signin, authenticate
- register, signup, createUser, createAccount
- resetPassword, changePassword, forgotPassword
- updateEmail, changeEmail, verifyEmail
- generateToken, refreshToken, revokeToken
- enableMFA, disableMFA, verifyOTP
```

**Why:** These are the highest-value targets for auth bypass, brute force, and account takeover.

### Priority 2: Admin / Privileged Operations

```
Look for:
- admin in name: adminGetUser, getAdminData, adminPanel
- delete: deleteUser, removeAccount, purgeData
- role/permission: updateRole, grantPermission, setAdmin, promoteUser
- ban/suspend: banUser, suspendAccount
- impersonate: impersonateUser, actAs, switchUser
```

### Priority 3: Object Access by ID

```
Look for patterns like:
- user(id: ID!)
- order(orderId: Int!)
- invoice(uuid: String!)
- message(id: ID!)
- document(fileId: String!)

All → IDOR candidates
```

### Priority 4: URL/Path Arguments

```
Look for:
- url, endpoint, webhook, callback
- path, filePath, filename
- redirect, returnUrl, next
- source, origin, destination

All → SSRF / path traversal candidates
```

### Priority 5: Raw Filter/Search Arguments

```
Look for:
- filter, where, search, query (nested)
- orderBy, sortBy (SQL ORDER BY injection)
- name, email, username with string type

All → SQLi / NoSQLi candidates
```

---

## 5. Visualizing the Schema

```bash
# Option A: GraphQL Voyager (browser-based)
# 1. Paste introspection JSON at https://graphql-kit.com/graphql-voyager/
# 2. Visual graph of all types and relationships
# 3. Click nodes to see fields and arguments

# Option B: InQL in Burp
# 1. Send introspection result to InQL Scanner
# 2. View all queries/mutations organized by type
# 3. Generate example queries automatically

# Option C: GraphQL Playground (if exposed)
# Navigate to /graphiql or /playground
# Run introspection via the IDE itself
# Use Docs panel to browse schema
```
