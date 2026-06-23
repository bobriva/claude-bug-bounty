# Payloads, Severity Reference, and Report Template

## Mass assignment / hidden field payloads

Send these as additions to a normal PATCH/PUT/POST body on any user-controlled object (user profile, order, account settings):

```json
{ "isAdmin": true }
{ "is_admin": true }
{ "role": "admin" }
{ "roles": ["admin"] }
{ "permissions": ["admin"] }
{ "scope": "admin" }
{ "verified": true }
{ "approved": true }
{ "status": "approved" }
{ "accountType": "enterprise" }
{ "userType": "internal" }
{ "internal": true }
{ "debug": true }
```

## Hidden parameter wordlist (query string or body)

```
isAdmin  is_admin  admin  debug  internal  test  bypass  override
godmode  sudo  role  roles  permission  permissions  scope  scopes
status  approved  verified  userType  accountType  impersonate_user_id
act_as  on_behalf_of  force  unsafe  legacy  v1  beta
```

## HTTP methods to test on every discovered endpoint

```
GET  POST  PUT  PATCH  DELETE  OPTIONS  HEAD
```

## Content-Types to test on write operations

```
application/json
application/xml
application/x-www-form-urlencoded
multipart/form-data
text/plain
```

## API version/path indicators (recon + fuzzing)

```
/api/   /v1/   /v2/   /v3/   /beta/   /internal/   /legacy/   /graphql
```

## Documentation paths

```
/swagger  /swagger-ui  /swagger-ui.html  /swagger/index.html
/openapi.json  /openapi.yaml  /api-docs  /docs  /redoc  /rapidoc
/v2/api-docs  /v3/api-docs
```

## SSRF probe targets

```
http://169.254.169.254/latest/meta-data/           # AWS metadata
http://metadata.google.internal/computeMetadata/v1/ # GCP metadata (needs Metadata-Flavor: Google header)
http://169.254.169.254/metadata/instance            # Azure metadata (needs Metadata: true header)
http://127.0.0.1:6379/                              # Redis, if internal services are reachable
http://127.0.0.1:9200/                              # Elasticsearch
```

Use an out-of-band listener (interactsh, Burp Collaborator) when you only need to confirm the request fired, before escalating to actually pulling metadata content.

---

## CVSS 3.1 quick reference for common API findings

| Finding | Typical CVSS range | Severity |
|---|---|---|
| BOLA, read-only (PII exposure) | ~5.3–6.5 | Medium |
| BOLA, write/delete capability | ~7.1–8.1 | High |
| BFLA → reaches admin function | ~8.1–9.8 | High–Critical |
| Mass assignment → privilege escalation | ~7.2–8.8 | High |
| Mass assignment, no observable privilege/data change | n/a | Not a finding — see validation gate |
| JWT `alg: none` accepted | ~9.1–9.8 | Critical |
| JWT algorithm confusion (forge any token) | ~9.1–9.8 | Critical |
| GraphQL introspection alone | n/a | Informational — not a finding |
| GraphQL missing field-level auth (PII exfil) | ~6.5–8.6 | Medium–High |
| SSRF reaching internal service only | ~5.3–6.8 | Medium |
| SSRF reaching cloud metadata + credentials extracted | ~9.0–9.8 | Critical |
| Rate limiting missing, demonstrated brute-force success | ~7.5–8.8 | High |
| Rate limiting "missing" with no completed abuse | n/a | Not a finding |

CVSS factors worth remembering when justifying a score to a triager:
- **Attack Vector**: almost always Network for API findings.
- **Privileges Required**: None if unauthenticated, Low if any valid account works, High if it needs an already-privileged account.
- **User Interaction**: None for most API bugs (no victim click needed) — this is part of why API bugs often score higher than equivalent UI bugs.
- **Scope/Impact**: be honest about Confidentiality/Integrity/Availability — don't claim "High" across all three if you only demonstrated one.

## Full report template

```markdown
**Title**: [Vuln class] in [exact endpoint] allows [actor] to [impact] [scope of victims]

## Summary
2-3 plain sentences: what the bug is, exactly where it lives, what an attacker
can actually do with it. No preamble, no "I am pleased to report."

## Steps to Reproduce
1. Log in as Account A ([role/org/tier])
2. Send: [exact method, path, headers, body — copy-paste ready]
3. Observe: [exact response demonstrating the issue]
4. [If applicable] Repeat using Account B's identifier while authenticated as
   Account A → confirms cross-account/cross-tenant access

## Proof of Concept
- Request/response pair (raw HTTP, or Burp/Caido export)
- Screenshot or short screen recording if the impact is easier to show visually

## Impact
An attacker with [access level] can [exact action], resulting in [concrete
harm — leaked PII, modified records, privilege escalation, financial loss].
Quantify where possible: "affects all N tenants," "exposes field X for every
user," "allows unlimited redemption of a single-use coupon code."

## Suggested Fix
One or two sentences, specific to this endpoint — e.g. "Verify the requesting
user's session owns or has been explicitly granted access to `organization_id`
server-side before returning invoice data; do not rely on the client-supplied
ID alone."

## Severity Assessment
CVSS 3.1: X.X ([Severity label])
Attack Vector: Network | Privileges Required: [None/Low/High] |
User Interaction: None | Scope: [C/I/A impact actually demonstrated]
```

### Tone notes (avoid "AI slop" that costs credibility with triagers)

- Open with the impact or the bug, not throat-clearing ("I am writing to report...").
- Write like you're explaining to a competent developer colleague, not a textbook.
- Use "I" and active voice — "I found that..." not "It was discovered that...".
- Don't add an "Executive Summary," "Methodology," or "Recommendations" section to a simple, single-endpoint bug — that formality reads as padding on anything below a multi-step chain.
- One concrete example beats three abstract sentences about "potential" impact.
- Keep the whole report under roughly 600 words; triagers skim, and a long report on a simple bug reads as the reporter not understanding their own finding.

### If severity gets pushed back on

| Triager says | You can counter with (if true) |
|---|---|
| "Requires authentication" | "Only a free/standard account is needed — no special role or invite" |
| "Limited impact" | State the actual scope: number of users/tenants affected, data type exposed |
| "Already known / duplicate" | Ask for the report number; if none exists, say so and ask for re-review |
| "By design" | Ask for the documentation stating this is intended behavior |

Only push back once, with facts, then accept the program's call — repeatedly arguing a small severity delta burns the relationship for future reports more than it's worth.
