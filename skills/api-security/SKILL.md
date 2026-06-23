---
name: api-security
description: Systematically map and test REST, GraphQL, and mobile API attack surfaces against the OWASP API Security Top 10 (BOLA/IDOR, Broken Authentication & JWT flaws, mass assignment/BOPLA, unrestricted resource consumption, BFLA, SSRF, security misconfiguration, shadow/zombie API versions) for authorized bug bounty and pentest engagements. Covers endpoint/parameter discovery (Swagger/OpenAPI, JS recon, ffuf, kiterunner, Arjun, Param Miner), cross-account authorization testing, JWT algorithm confusion, GraphQL introspection/batching abuse, plus a validation gate and report template that kill false positives before they reach a triager. Use whenever the user is hunting bugs on a REST/JSON/GraphQL/mobile/internal API, testing authorization or IDOR, auditing an OpenAPI spec, fuzzing parameters, or writing up/scoring an API finding for HackerOne, Bugcrowd, or YesWeHack — even if they just paste a request and ask "is this exploitable?"
---

# API Security Testing

This skill turns API testing into a closed loop: **map → test → validate → report**. The loop matters more than any single payload — most rejected bug bounty reports aren't wrong about the vulnerability class, they're wrong about *impact*, *scope*, or they were never verified by hand. This skill is built so that doesn't happen.

## Core principle: leads, not bugs

When you (Claude) are asked to "find vulnerabilities" in an API, you will be tempted to declare something a "Critical IDOR" the moment a status code looks interesting. Resist this. Experienced hunters who use AI heavily report that roughly 80% of AI-claimed findings collapse the moment someone asks "are you sure this is exploitable, and how would you prove it by hand?" Adopt that posture by default:

- Produce **prioritized leads** ("this endpoint is suspicious because X, here's the fastest way to confirm or kill it"), not confident vulnerability claims.
- For every lead you raise, immediately ask yourself the counter-questions: *Could there be a check I haven't seen? What's the simplest benign explanation? How do I confirm this with one concrete request/response pair?*
- Never report something you (or the user) have not reproduced end-to-end with an actual request and response. "Should theoretically work" is not a finding.
- Distinguish clearly, every time, between "confirmed" (you have the request/response proving impact) and "suspected" (pattern looks off, not yet tested).

This isn't caution for its own sake — it's what actually produces money-earning reports instead of a pile of N/As that damage the user's signal/reputation on the platform.

## Before you start: scope and context

Don't start firing requests. Confirm first (ask the user if unknown):

- **Scope**: which domains/hosts/API versions are explicitly in-scope, and what's explicitly excluded or forbidden (no automated scans above N req/s, no testing with real user PII, no destructive actions, etc.).
- **Accounts**: do you have credentials for **at least two accounts** (ideally in two different orgs/tenants if the app is multi-tenant)? Most high-value API findings (BOLA, BFLA, mass assignment) require comparing behavior *across* identities — a single account can't prove authorization is broken.
- **Auth mechanism**: session cookie, API key, OAuth bearer, JWT? This changes which reference file matters (see JWT section below).
- **Existing recon**: has the user already collected endpoints/JS files/an OpenAPI spec? Don't redo work that's already been done — read it first.

If this is a fresh target with no prior recon, do recon first (Phase 1) before touching vulnerability classes.

## The five-phase loop

| Phase | Goal | Reference |
|---|---|---|
| 1. Map the surface | Find every endpoint, parameter, version, and content-type the API actually accepts — not just what's documented | `references/recon-and-discovery.md` |
| 2. Test by vulnerability class | Work through the OWASP API Top 10 systematically against the mapped surface | `references/vulnerability-classes.md` |
| 2a. JWT (if used) | Algorithm confusion, weak secrets, claim tampering, header injection | `references/jwt-testing.md` |
| 2b. GraphQL (if present) | Introspection, batching/aliasing abuse, missing field-level auth, depth-based DoS | `references/graphql-testing.md` |
| 3. Validate | Run every candidate finding through the gate below before it's called a "bug" | This file, "Validation gate" section |
| 4. Report | Write it up so a tired triager can reproduce it in under 5 minutes | This file + `references/payloads.md` for ready-made test bodies |

Read the relevant reference file *when you reach that phase* — don't load everything up front.

## Phase 1 (quick view): map before you test

You cannot test what you haven't found. At minimum:

1. **Documentation discovery** — check `/swagger`, `/swagger-ui`, `/openapi.json`, `/openapi.yaml`, `/api-docs`, `/docs`, `/redoc`. If an OpenAPI/Swagger spec exists, diff it against what the frontend actually calls — undocumented operations are common and under-tested.
2. **JS recon** — API clients ship the full route map in their JS bundles. Pull every `/api/`, `/v1/`, `/v2/`, `/internal/`, `/graphql` reference out of the JS, including ones the UI never links to.
3. **Endpoint brute force** — use `kiterunner` (purpose-built for API route discovery; it understands REST conventions, not just static files) and `ffuf` against API-specific wordlists for anything the above missed.
4. **Hidden parameter discovery** — use `Arjun` or Burp's `Param Miner` against every known endpoint. Sensitive parameter names worth always testing: `isAdmin`, `role`, `permissions`, `userType`, `status`, `approved`, `verified`, `internal`, `debug`, `is_admin`, `bypass`, `override`, `impersonate_user_id`.
5. **Method and content-type matrix** — for every discovered endpoint, don't assume only the "obvious" verb works. Try `GET, POST, PUT, PATCH, DELETE, OPTIONS, HEAD` and `application/json, application/xml, application/x-www-form-urlencoded, multipart/form-data`. A 403 on POST doesn't mean PATCH is also blocked.
6. **Version discovery** — `/v1/`, `/v2/`, `/v3/`, `/beta/`, `/internal/`. Old versions are frequently left live with weaker auth than the current one — this is one of the single highest-yield API recon steps because nobody re-audits the version nobody uses anymore.

Full commands and tool flags: `references/recon-and-discovery.md`.

## Phase 2 (quick map): OWASP API Security Top 10 (2023)

| # | Class | One-line test | Detail |
|---|---|---|---|
| 1 | BOLA (Broken Object Level Authorization) | Swap the object ID in the request between Account A and Account B; does the server check ownership? | `vulnerability-classes.md` |
| 2 | Broken Authentication | Token replay across services, weak/missing JWT validation, no re-auth on sensitive changes | `jwt-testing.md` |
| 3 | BOPLA (Broken Object Property Level Authorization — merges old "Excessive Data Exposure" + "Mass Assignment") | Diff the GET response against what PATCH/POST accepts; inject fields that exist in the model but aren't in the documented request schema | `vulnerability-classes.md` |
| 4 | Unrestricted Resource Consumption | No rate/size/pagination limits — test cost-heavy or bulk operations | `vulnerability-classes.md` |
| 5 | BFLA (Broken Function Level Authorization) | Can a low-privilege account hit an admin-only operation directly? | `vulnerability-classes.md` |
| 6 | Unrestricted Access to Sensitive Business Flows | Can a flow (e.g. purchase, voting, account creation) be automated/abused at scale without the friction the business intended? | `vulnerability-classes.md` |
| 7 | SSRF | Any parameter that makes the server fetch a URL (webhooks, imports, avatar-by-URL, PDF generation) | `vulnerability-classes.md` |
| 8 | Security Misconfiguration | Verbose errors, debug endpoints left on, permissive CORS, missing security headers on API responses | `vulnerability-classes.md` |
| 9 | Improper Inventory Management | Shadow/zombie API versions, undocumented internal/mobile-only endpoints still reachable | Phase 1 above |
| 10 | Unsafe Consumption of APIs | The app trusts a third-party API's response/redirects more than user input — test what happens when that upstream response is attacker-influenced | `vulnerability-classes.md` |

BOLA alone accounts for roughly 40% of real-world API attacks and has held the #1 spot on this list since 2019 — when you're short on time, start there.

## Validation gate — run this before calling anything a finding

All of these need a **yes** before you write a report. Any **no** means: keep digging, downgrade it to "suspected," or drop it.

1. **Can I show the exact request and response that prove it?** If you can't paste a concrete HTTP request/response pair (or screenshot/video) that demonstrates the behavior, it isn't a finding yet — it's a hypothesis.
2. **Does this happen to a real victim doing nothing unusual?** "An attacker who already has X, Y, and Z" with several stacked preconditions is weak. A clean BOLA needs nothing more than a second account and a changed ID.
3. **Is the impact concrete?** Reading another user's email/PII, modifying another tenant's data, or escalating privilege is impact. "Inconsistent error message" or "theoretically could leak data" is not, on its own.
4. **Is it actually in scope?** Recheck the program's scope page for this exact host/path. Many API findings get killed simply because the endpoint lives on an excluded subdomain or third-party integration.
5. **Have you ruled out it being already public/known?** Check the program's disclosed reports (Hacktivity, Bugcrowd Crowdstream) and changelog for the same endpoint. Duplicates are the single most common reason for a "wasted" report.
6. **Did you test with two independent accounts (and two tenants, if multi-tenant)?** A single-account test cannot prove an authorization bug — it can only prove the feature works for its owner.
7. **Would a tired triager reading this at 5pm on a Friday agree it's a bug in under one minute?** If the explanation needs three paragraphs of setup before the impact is clear, the report needs to be tightened, not the bar lowered.

### Patterns that usually die at this gate (don't submit these alone)

- GraphQL introspection being enabled, with no follow-up query that actually pulls sensitive data
- A hidden/debug parameter existing, with no demonstrated behavior change when set
- Rate limiting "missing" with no actual brute-force/abuse proof completed
- An object ID that's guessable but the endpoint consistently returns 401/403/404 for other users' IDs in your own testing
- A verbose error message with no sensitive data inside it
- Mass assignment payload accepted by the API with no observable privilege or data change afterward

### Patterns that become valid once chained

| Weak alone | Chain it with | Becomes |
|---|---|---|
| Open redirect in an API param | OAuth `redirect_uri` reuse | Auth-code/token theft |
| Verbose debug endpoint | Leaked internal token/credential, demonstrated to reach something | Real credential exposure |
| BOLA (read-only) | Same ID pattern on a PUT/PATCH/DELETE endpoint | BOLA with write impact (higher severity) |
| SSRF reaching only an internal IP | Cloud metadata endpoint (`169.254.169.254`) with extracted credentials | Critical — cloud account compromise |
| Mass assignment accepted | Field that actually maps to a privilege/role column, confirmed via a follow-up authenticated request as that role | Privilege escalation |

When a finding is weak alone, your job is to spend 15 more minutes trying to chain it before discarding it — not to write it up as-is and hope.

## Writing it up

Once a finding passes the gate, write a report a triager can act on in minutes, not a school essay. Don't open with "I am pleased to report a critical vulnerability" — triagers recognize AI-generated padding immediately and it costs credibility before they've read the bug.

**Title formula:** `[Vuln class] in [exact endpoint] allows [actor] to [impact] [scope of victims]`
Good: `BOLA in GET /api/v1/organizations/{id}/invoices allows any authenticated user to read any other organization's invoices`
Bad: `IDOR vulnerability found in API`

**Structure:**
```
## Summary
2-3 plain sentences: what it is, where it is, what an attacker can actually do.

## Steps to Reproduce
1. Log in as Account A (role/org X)
2. Send: [exact request — method, path, headers, body]
3. Observe: [exact response demonstrating the bug]
4. Repeat with Account B's ID in Account A's session → confirms cross-account access

## Proof of Concept
Request/response pair, or short screen recording.

## Impact
An attacker with [access level] can [exact action], resulting in [concrete harm].
Quantify if you can: "affects all N tenants", "exposes field X for every user".

## Suggested fix
One or two sentences — e.g. "Verify the requesting user owns or has been granted access to `organization_id` server-side before returning invoice data."
```

Keep the whole report under ~600 words. Two test accounts in the steps (not one account testing itself), an honest severity claim that matches what you actually demonstrated, and a fix suggestion that's specific to this endpoint — not a lecture on access control in general.

For severity, see the CVSS quick-reference and report template details in `references/payloads.md`; for class-specific report phrasing (JWT, GraphQL, mass assignment) see the respective reference file.

## Tool quick-reference

| Tool | Purpose |
|---|---|
| `kiterunner` | API-aware route brute-forcing (understands REST path structures, much faster signal than generic `ffuf` for this) |
| `ffuf` | General-purpose fuzzing — paths, parameters, headers |
| `Arjun` / Burp `Param Miner` / `x8` | Hidden parameter discovery |
| Burp Suite or Caido | Intercept, replay, diff requests across accounts |
| `jwt_tool` / Burp JWT Editor | JWT structural attacks (alg confusion, none-alg, claim tampering) |
| InQL / GraphQL Voyager | GraphQL schema exploration from introspection, query generation |
| `nuclei` | Fast template-based sweep for known misconfigurations once the surface is mapped |

Full commands, flags, and example payloads: `references/recon-and-discovery.md` and `references/payloads.md`.

## Reference files in this skill

- `references/recon-and-discovery.md` — endpoint/parameter discovery commands and wordlists
- `references/vulnerability-classes.md` — full testing methodology per OWASP API Top 10 category
- `references/jwt-testing.md` — JWT-specific attack techniques
- `references/graphql-testing.md` — GraphQL-specific attack techniques
- `references/payloads.md` — ready-to-use request bodies, parameter lists, CVSS quick-reference, and the full report template
