---
name: graphql
description: >
  Expert GraphQL security testing for bug bounty. Trigger this skill whenever
  a target exposes GraphQL — visible or hidden. Indicators: /graphql path exists,
  Apollo/GraphiQL playground present, requests with {"query":"..."} in Burp,
  Content-Type: application/json on API endpoints, "errors" array in JSON responses,
  "__typename" in JS source, any mention of GraphQL in JS bundles or source maps.
  Also trigger when REST API endpoints behave unusually (hidden GraphQL behind REST
  wrapper is common). Covers the full chain: discovery → fingerprinting → schema
  recovery → authorization testing → injection → abuse → impact escalation.
---

# GraphQL Security Testing

## Quick Orientation

Read this file first. Follow the decision tree to load the right reference.

**Reference files:**
- `references/recon.md` — Endpoint discovery, engine fingerprinting, method probing
- `references/schema.md` — Introspection, bypass techniques, Clairvoyance, schema analysis
- `references/authorization.md` — IDOR, broken access control, auth bypass, privilege escalation
- `references/injection.md` — SQLi, NoSQLi, SSRF, XSS, SSTI, command injection via resolvers
- `references/abuse.md` — Alias abuse, batching, rate limit bypass, brute force, CSRF, DoS
- `references/payloads.md` — All payloads organized by attack type, copy-paste ready

---

## Phase 1: Discovery & Fingerprinting

**Load `references/recon.md`**

Always start here. Do not assume the endpoint is where you expect it.

```
Key questions to answer:
1. Where is the GraphQL endpoint? (common + hidden paths)
2. Which HTTP methods does it accept? (GET, POST, form-encoded?)
3. Which GraphQL engine is running? (Apollo, Hasura, GraphQL-Ruby, Graphene, etc.)
4. Is GraphiQL / Playground exposed? (dev interface = higher attack surface)
5. Does it require authentication to query?
```

---

## Phase 2: Schema Recovery

**Load `references/schema.md`**

This is the most important phase — schema = map of the entire API.

```
Decision tree:
├── Introspection enabled?
│   └── YES → Run full introspection, visualize with Voyager, extract all
│              queries/mutations/subscriptions/types
│
├── Introspection disabled?
│   ├── Try bypass techniques (newline, whitespace, GET method, fragment trick)
│   ├── Check field suggestions ("Did you mean...") → run Clairvoyance
│   └── Check GraphQuail extension in Burp (passive schema reconstruction)
│
└── Suggestions also disabled?
    └── Manual field guessing + SecLists graphql.txt wordlist + error analysis
```

---

## Phase 3: Authorization Testing (Highest Bounty Value)

**Load `references/authorization.md`**

GraphQL has NO built-in auth — all access control is developer-implemented.
This means every query and mutation must be individually tested.

```
Priority targets from schema:
1. User-related queries (getUser, me, users, userById)
2. Financial objects (orders, invoices, transactions, payments)
3. Admin mutations (deleteUser, updateRole, createAdmin, banUser)
4. Private data (messages, addresses, tokens, secrets)
5. Cross-tenant data (workspaces, organizations, teams)
```

---

## Phase 4: Injection Testing

**Load `references/injection.md`**

Every argument that touches a backend system is an injection candidate.

```
Injection targets in GraphQL:
- String arguments → SQLi, NoSQLi, XSS, SSTI
- URL arguments → SSRF
- File path arguments → Path Traversal
- Command arguments → RCE
- Template arguments → SSTI
```

---

## Phase 5: Feature Abuse

**Load `references/abuse.md`**

GraphQL features that are legitimate but abusable:

```
Alias abuse → rate limit / 2FA bypass, mass brute force
Array batching → same bypass, also DoS
CSRF via GET or form encoding → account takeover
Query depth abuse → DoS
Subscription abuse → persistent connection DoS
```

---

## Phase 6: Decision Tree — What to Test First

```
GraphQL endpoint found
│
├── Is introspection open?
│   ├── YES → Full schema dump → map all mutations → go to Phase 3+4+5
│   └── NO  → Schema bypass → Clairvoyance → partial schema → continue
│
├── Is GraphiQL / Playground exposed?
│   └── YES → Dev environment → likely fewer restrictions, test harder
│
├── Does endpoint accept GET requests?
│   └── YES → CSRF possible → test all state-changing mutations via GET
│
├── Does it accept application/x-www-form-urlencoded?
│   └── YES → CSRF without CORS bypass needed
│
├── Are there user/object ID arguments?
│   └── YES → IDOR → enumerate IDs across accounts
│
├── Are there mutations for email/password/role changes?
│   └── YES → Critical auth bypass targets
│
├── Are there URL/path arguments in mutations/queries?
│   └── YES → SSRF candidates
│
└── Are there filter/search string arguments?
    └── YES → SQLi / NoSQLi candidates
```

---

## Severity & Impact Guide

| Finding | Severity | Bounty Range |
|---|---|---|
| IDOR on financial data (transactions, payments) | Critical | $5K–$25K |
| Auth bypass to admin mutations | Critical | $5K–$20K |
| Introspection + admin route discovery + exploit | High/Critical | $3K–$15K |
| SSRF via GraphQL resolver | High/Critical | $3K–$10K |
| SQLi via GraphQL argument | High/Critical | $5K–$20K |
| Alias abuse → 2FA bypass | High | $2K–$8K |
| Batching → account brute force | High | $2K–$5K |
| CSRF via GET mutation | High | $2K–$8K |
| Stored XSS via mutation argument | High | $1K–$5K |
| Introspection exposure (standalone) | Low/Info | $100–$500 |
| GraphiQL exposed in prod | Low/Medium | $200–$1K |

---

## Tools Quick Reference

```bash
# Engine fingerprinting
graphw00f -d https://target.com/graphql

# Automated security audit
graphql-cop -t https://target.com/graphql
graphql-cop -t https://target.com/graphql -H "Authorization: Bearer TOKEN"

# Schema recovery (introspection disabled)
clairvoyance https://target.com/graphql -o schema.json
clairvoyance https://target.com/graphql -H "Authorization: Bearer TOKEN" -o schema.json

# Schema visualization
# InQL (Burp extension) → auto-generates schema + query templates
# GraphQL Voyager → paste introspection JSON → interactive graph

# Brute force / fuzzing via aliases
CrackQL --target https://target.com/graphql --query-file bruteforce.graphql

# Interactive exploitation
graphql-map -u https://target.com/graphql --method POST

# Automated scan
nuclei -t graphql -u https://target.com/
```
