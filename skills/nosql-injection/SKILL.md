---
name: nosql-injection
description: Detect and exploit NoSQL injection — MongoDB operator injection ($ne, $gt, $regex, $where, $exists, $in), PHP/form-encoded array-bracket injection, syntax/boolean injection, blind regex and timing-based ($where + sleep) data extraction, authentication bypass, unknown-field enumeration, and CouchDB's unauthenticated "Admin Party" default — for authorized bug bounty and pentest work. Covers JSON-body and URL-encoded injection vectors, operator-filter bypass when $ne is blocked, character-by-character blind extraction of passwords/tokens/secrets, mapReduce-based bulk exfiltration, the modern ODM-level $where-nested-under-$or filter-bypass pattern (Mongoose CVE-2024-53900/CVE-2025-23061), and a validation gate plus report template that kill false positives before reaching a triager. Use for any login/search/filter endpoint backed by MongoDB, CouchDB, DynamoDB, or another NoSQL store, or JSON-body APIs in general — or writing up a NoSQL injection finding for HackerOne, Bugcrowd, or YesWeHack.
---

# NoSQL Injection Testing

Same loop as the other skills in this set: **map → test → validate → report**. NoSQL injection survives in 2026 specifically because it doesn't look like SQL injection — generic scanners and WAF rules tuned for `' OR 1=1` miss `{"$ne": null}` entirely, which makes this a consistently under-tested, high-yield category on any target running a JSON API over MongoDB (by far the most common backend you'll encounter) or a legacy CouchDB instance.

## Core principle

NoSQL injection works because the application lets the **shape** of the input control the **shape** of the query, not just its value. A login check written as `db.users.findOne({username: req.body.username, password: req.body.password})` assumes both fields will always be strings — if you can instead send an object (`{"$ne": null}`) where a string was expected, you're not injecting query syntax the way SQL injection does, you're injecting an entirely different and fully-valid query operator that the database happily executes. Every technique in this skill is a variation of that one idea: find a field whose *type*, not just its value, is attacker-controlled.

## Before you start: identify the backend and the injection vector

```
1. Backend indicators: JSON-accepting APIs, Node.js/Express stack
   hints, BSON-flavored error messages, ObjectId-shaped identifiers
   (24-char hex strings) in URLs/responses — all point to MongoDB,
   the dominant target for this skill.
2. Injection vector: does the endpoint accept Content-Type: application/json
   (you control the body's structure directly), or only
   application/x-www-form-urlencoded / query-string parameters? The
   second case still works via PHP-style bracket notation — see
   `references/operator-injection-and-bypass.md`.
3. Sensitivity of the target field: login/auth fields are highest
   value (see "Objectives" below), but search, filter, and "forgot
   password" lookup fields are frequently just as injectable and
   less tested.
```

## The methodology

| Phase | Goal | Reference |
|---|---|---|
| 1. Recon & vector ID | Backend fingerprinting, JSON vs. form-encoded injection vector | This file, above |
| 2. Syntax & boolean detection | Fuzz strings, true/false conditions, null-byte truncation | `references/detection-and-syntax-injection.md` |
| 3. Operator injection | `$ne`/`$gt`/`$regex`/`$in`/`$exists` auth bypass and filter bypass | `references/operator-injection-and-bypass.md` |
| 4. Blind extraction & `$where` | Boolean/timing-based character extraction, mapReduce exfiltration, JS-execution risk | `references/blind-extraction-and-where-rce.md` |
| 5. Validate | Run every candidate through the gate below | This file, "Validation gate" |
| 6. Report | Concrete proof, real data recovered, fast to triage | This file + `references/payloads-and-reporting.md` |

## Validation gate — run this before calling anything a finding

Every "no" means: keep digging, downgrade to "suspected," or drop it.

1. **Did the injected operator actually change query logic, not just survive parsing?** A 200 OK with an object payload accepted is not proof — you need a result that differs from what a non-injected request returns (a different user logged in, more records returned, a true boolean signal reproduced consistently).
2. **For auth bypass: did you actually get authenticated as an account, with a real session/token, ideally one that isn't yours?** "The request didn't error" is not login. Confirm by using the resulting session to access an authenticated endpoint.
3. **For blind extraction: is the signal reproducible across repeated trials, and did you recover an actual, verifiable value?** One "different-looking" response is noise; a regex-narrowed character sequence that you then confirm independently (e.g., it successfully logs in, or matches a value obtained another way) is a finding.
4. **For `$where`/timing-based tests: is the delay specifically tied to your injected condition, repeatable, and clearly distinguishable from normal network jitter?** Run the true and false condition several times each and compare distributions, not single samples.
5. **Is the field/endpoint actually sensitive?** Operator injection accepted on a public, non-sensitive search filter with no unauthorized data exposure is a much weaker finding than the same bug on a login or password-reset lookup.
6. **Is it a duplicate?** `$ne`-based MongoDB login bypass is one of the most commonly reported NoSQL findings in existence — check disclosed reports, and don't expect novelty alone to drive severity; demonstrated impact does.

### Patterns that usually die at this gate (don't submit these alone)

- An operator payload (`{"$ne": null}`, `{"$gt": ""}`) accepted with a 200 response, with no confirmation you're actually authenticated as a real account
- A single boolean test that looked different once, not reproduced on a second run
- A `$where` timing payload where the "delay" is within normal response-time variance, not a clear multi-second outlier across repeated trials
- Operator injection that only affects a field with no sensitive consequence (e.g., a `sort` or `limit` parameter accepting an object with no observable behavior change)
- Claiming "RCE possible via `$where`" with no actual demonstrated JavaScript execution (a confirmed `sleep()` delay, at minimum) backing the claim

### Patterns that become valid once chained

| Weak alone | Chain it with | Becomes |
|---|---|---|
| `{"username": {"$ne": null}, "password": {"$ne": null}}` logs into *some* account | `{"$regex"}` or `{"$in": ["admin"]}` targeting a specific known username | Confirmed admin-account takeover, not just "any account" |
| Blind `$regex` extraction works against a string field | The field happens to be a password-reset token or 2FA secret, fully recovered character-by-character | Full account takeover — mirrors real disclosed reports where exactly this chain led from blind NoSQLi to admin RCE |
| `$where` accepts a JS expression and `sleep()` confirms execution | `mapReduce`-based exfiltration to pull many records' worth of a field in fewer requests than character-by-character regex would need | Practical bulk data extraction, not just a single-field proof of concept |
| CouchDB instance responds unauthenticated on its HTTP API | `_users/_all_docs` actually returns credential documents | Full credential-database disclosure, not just "looks unauthenticated" |
| ODM-level sanitization blocks bare `$where` | Nesting `$where` inside an `$or` clause that the sanitizer only checks at the top level | Filter bypass restoring full JS-execution risk even after a vendor patch — confirm against the specific ODM/version in use |

Spend the extra time chaining a working primitive (auth bypass, blind extraction) into something with concrete, named impact (a specific account, a specific recovered secret) before writing it up — "I can manipulate the query" alone consistently scores lower than "I recovered this user's password reset token and took over their account."

## Writing it up

**Title formula:** `NoSQL injection in [exact endpoint] allows [actor] to [impact]`
Good: `NoSQL injection in POST /api/login allows unauthenticated attacker to bypass authentication via $ne operator and log in as any user, including admin`
Bad: `NoSQL injection found`

Full payload cheat-sheet, tooling, severity guidance, and report template: `references/payloads-and-reporting.md`.

## Reference files in this skill

- `references/detection-and-syntax-injection.md` — backend fingerprinting, fuzz strings, boolean/syntax detection, null-byte truncation
- `references/operator-injection-and-bypass.md` — operator injection (JSON and PHP/form-encoded vectors), authentication bypass, filter-bypass variations
- `references/blind-extraction-and-where-rce.md` — boolean/timing blind extraction, unknown-field enumeration, `$where` JS execution risk, mapReduce exfiltration, CouchDB Admin Party, other backends
- `references/payloads-and-reporting.md` — payload cheat-sheet, tools (nosqli, NoSQLMap), CVSS quick-reference, full report template
