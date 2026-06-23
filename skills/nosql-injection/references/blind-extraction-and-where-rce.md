# Blind Extraction, `$where` JS Execution, and Other Backends

## Boolean-based blind extraction via `$regex`

Once injection is confirmed, `$regex` lets you recover the actual contents of any string field used in a query condition, one character at a time, using nothing but a true/false signal (a different status code, response length, or content):

```json
{"username": "admin", "password": {"$regex": "^a"}}
{"username": "admin", "password": {"$regex": "^b"}}
{"username": "admin", "password": {"$regex": "^c"}}
```
Whichever returns the "match" signal, extend the prefix and repeat:
```json
{"password": {"$regex": "^ab"}}
{"password": {"$regex": "^ac"}}
```
This works against any string field that participates in a queryable condition — not just `password`. Password-reset tokens, API keys, session secrets, and 2FA secrets are all viable targets if they ever appear in a query you can influence. This is exactly the chain behind real disclosed reports where blind regex extraction against an internal lookup API recovered password-reset tokens and 2FA secrets directly, leading to full admin account takeover and, from there, remote code execution once admin-level functionality was reachable — treat any string secret field reachable via this technique as a serious lead, not a curiosity.

Automate the character-by-character process rather than doing it by hand — `nosqli` (see `payloads-and-reporting.md`) handles this directly from a captured request.

## Unknown-field enumeration

If you don't know a sensitive field's name, you can still discover the application's data model:

```javascript
this.username
this.password
this.email
this.role
this.apiKey
this.token
this.secret
this.isAdmin
```
Each of these (used inside a `$where` context, or via fields probed with `$exists`) tests whether the field exists on the document. Once you've found a candidate field name, switch to the `$regex` technique above to recover its actual value.

## Timing-based (blind) extraction via `$where`

`$where` evaluates a JavaScript expression server-side — and, critically, it's enabled by default on many MongoDB deployments even in recent versions; disabling it requires an explicit `security.javascriptEnabled: false` server-side setting that many teams never set.

**Confirm the primitive first**, with a clear, repeatable delay:
```json
{"$where": "sleep(5000)"}
```
A consistent ~5 second delay across several repeated requests (compared against a baseline with no delay payload) confirms genuine server-side JavaScript execution — a single slow response proves nothing on its own; network jitter produces false positives constantly, so always compare multiple trials of both the delay payload and a no-delay control.

**Extract data via conditional timing**, once confirmed:
```json
{"$where": "this.password.match(/^a/) && sleep(5000)"}
```
If the field matches, the query sleeps; if not, it returns immediately. Narrow the regex the same way as the boolean `$regex` technique above, just using response time instead of response content as the signal.

## Bulk exfiltration via `mapReduce`

Character-by-character extraction is slow when you need many records or many fields. `mapReduce` (another JS-execution context, subject to the same `javascriptEnabled` setting as `$where`) can be used to exfiltrate data in bulk rather than one field at a time, by writing a custom map function that emits the data you want to recover into a result set you can then read back. Treat any endpoint that exposes `mapReduce`, `$accumulator`, or `$function` to user-influenced input the same way you'd treat `$where` — confirm JS execution with a timing probe first, then escalate to actual data recovery rather than command execution.

## How far `$where`/JS execution can realistically escalate

On current, supported, properly-patched MongoDB deployments, treat `$where`/`mapReduce` JS execution as a **data confidentiality and DoS** risk (bulk read access, intentional resource-exhaustion via expensive loops) — not as a direct path to host-level remote code execution. Full RCE via MongoDB's server-side JavaScript engine has historically been demonstrated (publicly documented against MongoDB versions predating 2.4, via low-level engine internals), but that required a specific, long-since-patched legacy version and genuine binary-exploitation work (memory corruption, shellcode) far beyond constructing an injection payload — don't claim RCE based on JS-execution confirmation alone; claim what you actually demonstrated (data access, timing-based extraction, or a sustained denial-of-service), and only escalate the RCE claim if you have an actual working chain to show for it.

The more relevant modern version of "`$where` leads to RCE" is at the **ODM/library** layer, not the database engine: see `operator-injection-and-bypass.md`'s filter-bypass section — recent real-world CVEs against popular Node.js MongoDB ODMs involved an initial patch blocking `$where` at the top level of a query object, bypassed by nesting it inside an `$or` clause that the sanitizer never recursed into, which restored full JS-execution (and from there, RCE) risk even on a "patched" version. If you're testing a Node.js/Mongoose-backed target, check the ODM version and whether this specific nesting bypass applies before concluding `$where` is safely blocked.

## CouchDB: check for unauthenticated "Admin Party" access

CouchDB instances prior to version 3.0 ship in "Admin Party" mode by default: no authentication required at all, every request processed with full admin rights. Many production deployments running an older pulled container image, or frozen on 2.x for compatibility reasons, still expose this. Always check the CouchDB HTTP API unauthenticated before assuming a login is required:

```bash
curl http://<target>:5984/_all_dbs
curl http://<target>:5984/<dbname>/_all_docs?include_docs=true
curl http://<target>:5984/_users/_all_docs?include_docs=true
```
The `_users` database specifically stores credential documents (hashed passwords) for every registered user — full read access here is a complete credential-database disclosure, not a partial one, and `_all_dbs` combined with `_all_docs` on each result maps the entire data layer in two or three requests.

## DynamoDB and Firebase: different testing surface

These don't use the same operator-injection primitives as MongoDB — there's no `$where`/`$ne` to inject into a query string in the same way, because client SDKs typically build structured API calls rather than string-interpolated queries. The relevant risk for these is almost always **authorization**, not syntax injection:
```
- DynamoDB: check whether IAM policies / fine-grained access control
  actually restrict which items a given identity can read or write —
  this is an access-control testing problem (see the `api-security`
  skill's BOLA/BFLA testing for the general approach), not a
  string-injection one.
- Firebase Realtime Database / Firestore: check security rules by
  attempting direct REST/SDK reads and writes to paths/collections
  outside what the current user should access, and check for
  publicly-readable/writable rules left at their permissive defaults.
```
Don't force-fit MongoDB-style operator payloads onto these backends — confirm which engine you're actually facing first (see `detection-and-syntax-injection.md`) and pivot testing approach accordingly.

## What makes a blind-extraction or `$where` finding confirmed, not theoretical

A real recovered value (a password, a token, a secret) that you independently verify is correct — by using it (logging in with it, using it on a reset endpoint) or by cross-checking it against a value you already know in a test account — and, for timing-based tests, a clearly reproducible delay across multiple trials with a control comparison. "The response time seemed longer" once is not a finding.
