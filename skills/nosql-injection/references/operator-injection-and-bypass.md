# Operator Injection & Authentication Bypass

## The core mechanism

MongoDB queries are JSON objects, and comparison/logical operators (`$ne`, `$gt`, `$regex`, etc.) are themselves just keys in that object. If an application builds a query directly from user input without enforcing that the input is a plain string — e.g. `db.users.findOne({username: req.body.username, password: req.body.password})` — and accepts `Content-Type: application/json`, you control the *shape* of `req.body.username`, not just its value. Sending an object instead of a string introduces a real MongoDB operator that the database evaluates server-side.

## JSON-body injection (direct)

```json
{"username": {"$ne": null}, "password": {"$ne": null}}
```
`$ne: null` matches any stored value that isn't literally `null` — true for essentially every real account. Sent to a login endpoint, this returns the first matching document in the collection, with no credentials needed and no usernames to guess. It's often an admin account if the collection was populated in insertion order, but treat that as a lead to confirm, not an assumption — check who you actually logged in as.

## PHP / form-urlencoded injection (the URL-encoded equivalent)

When the application only accepts form-encoded or query-string input, you can still construct the same attack, because PHP (and several other frameworks) automatically convert bracket notation into nested arrays/objects:

```
# Normal request
POST /login
username=admin&password=secret

# Injected — PHP turns password[$ne]=x into {"password": {"$ne": "x"}}
POST /login
username=admin&password[$ne]=x
```
This works even when there's no visible JSON anywhere in the application's normal traffic — the bracket-notation trick is the vector, not the Content-Type. Test it on every form-encoded field that might reach a NoSQL query, not just on endpoints that look JSON-native.

## Operator variations (use when one is filtered)

If `$ne` specifically is blocked or stripped, these achieve the same effect through different logic — try each in turn:

```json
{"username": {"$gt": ""}, "password": {"$gt": ""}}
{"username": {"$regex": ".*"}, "password": {"$regex": ".*"}}
{"username": {"$exists": true}, "password": {"$exists": true}}
{"username": {"$in": ["admin", "administrator", "root"]}, "password": {"$ne": ""}}
```
`$gt: ""` is true for any non-empty string. `$regex: ".*"` matches anything. `$exists: true` is true whenever the field is present at all, regardless of its value. `$in` with a list of likely usernames lets you target a specific account directly rather than accepting whichever document happens to match first — combine this with `$ne`/`$gt` on the password field for a single-request targeted bypass attempt.

## Filter/sanitization bypass patterns

If you've confirmed the application or its ODM (e.g., Mongoose) attempts to strip `$`-prefixed keys but isn't fully recursive:

```
- Test whether sanitization only inspects top-level keys: nest the
  operator one level deeper than the sanitizer checks — e.g. inside
  an $or array, where each array element is its own object that a
  shallow, top-level-only sanitizer may never recurse into.
  {"$or": [{}, {"password": {"$ne": null}}]}
- Test whether sanitization is applied to the whole body but skips
  array values, nested objects, or specific known field names.
- Test alternate encodings of the $ character if the filter does a
  naive string match: unicode-escaped keys, or wrapping the operator
  inside a differently-cased or whitespace-padded key, depending on
  how the parser/sanitizer tokenizes keys.
```
This exact class of bypass — nesting a blocked operator one level deeper than a patch's sanitizer checks — has been the root cause of real, recently-disclosed CVEs against popular Node.js MongoDB ODMs, where an initial patch blocked the operator at the top level only for the bypass to resurface once nested under a logical operator the sanitizer didn't recurse into. Always test nested placements specifically after an operator appears to be filtered, rather than concluding the endpoint is hardened.

## Confirming an authentication bypass is real, not theoretical

You need a session/token issued by the server as a result of your injected request, and you need to use that session against an authenticated endpoint to confirm it actually works — and ideally confirm *which* account you're now authenticated as (check a profile/me endpoint) rather than assuming it's the admin. A 200 response with no further verification is not yet a confirmed bypass.
