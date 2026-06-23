---
name: sql-injection
description: Detect, validate, and exploit SQL injection across all contexts — error-based, boolean-blind, UNION-based, time-based, OAST/out-of-band, and second-order — across MySQL, PostgreSQL, MSSQL, Oracle, and SQLite, for authorized bug bounty and pentest work. Covers DBMS fingerprinting, column-count discovery, per-engine time-delay and OAST primitives (xp_dirtree, UTL_HTTP, LOAD_FILE UNC paths, dblink, DNS-label-length-aware chunked exfiltration), WAF/filter bypass (inline comments, case randomization, encoding, sqlmap tamper scripts), second-order injection across a store-then-execute boundary, and escalation to RCE (xp_cmdshell, COPY...PROGRAM, INTO OUTFILE+UDF) with a responsible stop-at-minimal-proof principle — plus a validation gate and report template that kill false positives before reaching a triager. Use for any login, search, filter, sort, report, or API parameter that reaches a database, or writing up a SQL injection finding for HackerOne, Bugcrowd, or YesWeHack.
---

# SQL Injection Testing

Same loop as the other skills in this set: **map → test → validate → report**. SQL injection is the oldest bug class in this entire skill set and still one of the highest-paying, precisely because the easy cases (a bare `'` causing a 500) get found and fixed constantly while the harder cases — blind, second-order, behind a WAF, escalated to RCE — survive specifically because they take more than five minutes to confirm properly. Always verify manually before reporting, even when a scanner or sqlmap flags something; automated tools produce false positives on response-difference heuristics constantly.

## Before you start: map inputs broadly

```
GET/POST parameters    JSON body fields       XML body fields
Cookies                 Custom headers          Sort/filter/order params
URL path segments       Multipart form fields   WebSocket messages (if used)
```
Don't stop at the obvious search box — sort/filter/pagination parameters, "export to CSV/PDF" report generators, and anything that builds a dynamic `ORDER BY`/`WHERE` clause from user input are disproportionately under-tested and frequently injectable even on otherwise well-hardened targets.

## The methodology

| Phase | Goal | Reference |
|---|---|---|
| 1. Detection & fingerprinting | Syntax/error/boolean probes, identify the DBMS | `references/detection-and-fingerprinting.md` |
| 2. UNION & error-based extraction | Column count, direct data extraction via UNION or error messages | `references/union-and-error-based.md` |
| 3. Blind & OAST extraction | Boolean/time-based character extraction, per-DBMS OAST primitives, second-order | `references/blind-and-oast-extraction.md` |
| 4. WAF bypass & RCE escalation | Filter/WAF evasion, stacked queries, xp_cmdshell/COPY PROGRAM/UDF chains | `references/waf-bypass-and-rce-chains.md` |
| 5. Validate | Run every candidate through the gate below | This file, "Validation gate" |
| 6. Report | Real extracted data, reproducible PoC, fast to triage | This file + `references/payloads-and-reporting.md` |

## Validation gate — run this before calling anything a finding

Every "no" means: keep digging, downgrade to "suspected," or drop it.

1. **Did you recover real, verifiable evidence — not just an unusual response?** A SQL-specific error message naming the query/table, an actual extracted value (DB version string, a real username), a UNION column count that produces clean (not garbled) output, or an OAST interaction that actually arrived. A generic 500 with no SQL-specific content is a lead, not a finding.
2. **For boolean-blind: is the true/false signal reproducible across multiple trials with a clearly defined, consistent indicator** (specific content present/absent, a specific row count) — not a vague "the page looked different once"?
3. **For time-based: did you run a control comparison?** Both the delay-inducing and baseline payloads, several times each, compared as distributions — a single slow response proves nothing given normal jitter.
4. **For OAST: did the interaction actually arrive on infrastructure you control**, ideally with real exfiltrated data inside it (not just "a payload was sent toward a suspected sink")?
5. **For second-order: did you confirm execution happens specifically at the second (later/different-context) operation**, with proof tied to that operation — not just that you stored a value containing SQL syntax?
6. **If escalating to RCE: did you stop at minimal, non-destructive proof** (a benign command's output via a pingback/file write, not a reverse shell or persistent backdoor)? See the WAF-bypass/RCE reference file for the same responsible-scope principle used throughout this skill set.
7. **Is it a duplicate?** SQLi on login forms and search boxes is one of the most over-reported bug classes in existence — check disclosed reports, and let demonstrated impact (not novelty) drive your severity claim.

### Patterns that usually die at this gate (don't submit these alone)

- A generic 500 error from a single `'` with no SQL-specific error text and no boolean/time follow-up confirming it
- A boolean payload that "looked different" once, with no reproduction
- Claiming UNION extraction works without first confirming the actual column count — garbled or partial output from a wrong column count is not a finding
- An OAST payload sent with no confirmed interaction in the Collaborator/interactsh log
- "WAF bypass confirmed" because the payload wasn't blocked, with no actual data extracted behind it
- DBMS/version claims guessed from generic error phrasing rather than actually extracted via a confirmed technique

### Patterns that become valid once chained

| Weak alone | Chain it with | Becomes |
|---|---|---|
| Slow, one-character-at-a-time boolean-blind extraction | UNION-based extraction once column count and a reflected column are confirmed | Fast bulk data extraction instead of a multi-hour blind crawl |
| Time-based signal only (slow, hard to scale) | A confirmed OAST channel (`xp_dirtree`, `UTL_HTTP`, `LOAD_FILE` UNC, `dblink`) | Reliable, fast exfiltration and a more convincing PoC than a timing graph |
| Read-only data extraction | Confirmed stacked-query support (`;` accepted, a second statement executes) | Escalation to data modification, authentication bypass via `UPDATE`, or RCE (see `waf-bypass-and-rce-chains.md`) |
| Injection confirmed in a stored value with no immediate effect | The value later reaching a different, unsanitized query context (an admin report, a batch job, a search index rebuild) | Second-order SQLi — often higher severity because it's reachable from a context the original submitter doesn't control |
| WAF blocking obvious payloads | Tamper/encoding bypass (inline comments, case randomization, hex/Unicode encoding) plus actual extracted data behind it | Confirmed bypass with real impact, not just "the request wasn't blocked" |

## Writing it up

**Title formula:** `SQL injection in [exact endpoint/parameter] allows [actor] to [impact]`
Good: `Boolean-blind SQL injection in GET /api/orders?sort= allows authenticated user to extract arbitrary database contents, confirmed via OAST DNS callback`
Bad: `SQLi found`

Full payload cheat-sheet per DBMS, sqlmap usage notes, severity guidance, and report template: `references/payloads-and-reporting.md`.

## Reference files in this skill

- `references/detection-and-fingerprinting.md` — syntax/error/boolean detection, parameter discovery across contexts, DBMS fingerprinting
- `references/union-and-error-based.md` — column-count discovery, UNION-based extraction, error-based extraction per DBMS
- `references/blind-and-oast-extraction.md` — boolean/time-based blind extraction, per-DBMS OAST primitives, DNS-length-aware exfiltration, second-order detection
- `references/waf-bypass-and-rce-chains.md` — WAF/filter bypass techniques, sqlmap tamper scripts, stacked queries, xp_cmdshell/COPY PROGRAM/UDF RCE chains
- `references/payloads-and-reporting.md` — payload cheat-sheet, CVSS quick-reference, full report template
