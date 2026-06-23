# API Recon & Discovery

Goal: before testing a single vulnerability class, know the *real* attack surface — every endpoint, parameter, method, content-type, and version the server actually responds to, not just what's documented or visible in the UI.

## 1. Documentation discovery

Check these paths directly (and via the brute-force step below if they're not at the obvious location):

```
/swagger
/swagger-ui
/swagger-ui.html
/swagger/index.html
/openapi.json
/openapi.yaml
/api-docs
/docs
/redoc
/rapidoc
/v2/api-docs
/v3/api-docs
```

If you find a spec, don't just read it — **diff it against reality**. Pull every path the frontend JS actually calls and compare against the documented operations. Paths that exist in the JS but not in the spec (or vice versa) are exactly where security review tends to be thinnest.

## 2. JavaScript recon

Modern SPAs ship their entire API surface inside the JS bundle.

```bash
# Pull all JS files referenced by the app
katana -u https://target.com -d 3 -silent | grep '\.js$' | sort -u

# Beautify minified bundles for readability
js-beautify bundle.min.js > bundle.pretty.js

# Pull candidate endpoints/paths out of JS
grep -oE '"(/api/[a-zA-Z0-9/_-]+)"' bundle.pretty.js | sort -u
grep -oE '"(/v[0-9]+/[a-zA-Z0-9/_-]+)"' bundle.pretty.js | sort -u

# Look for feature flags — flags toggled client-side often aren't enforced server-side
grep -iE 'feature.?flag|FF_[A-Z_]+|BETA_|INTERNAL_' bundle.pretty.js
```

Things worth flagging from JS recon: undocumented admin routes, API keys/secrets accidentally bundled, business logic implemented client-side (validation that should be server-side), and feature flags that imply a backend capability exists even if the UI hides it.

## 3. Endpoint brute-forcing

`kiterunner` is purpose-built for APIs — it understands REST path conventions (resource/{id}/sub-resource patterns) rather than just brute-forcing static filenames, so it finds API routes `ffuf` with a generic wordlist often misses.

```bash
# Kiterunner against a routes wordlist (assetnote's routes-large is the standard one)
kr scan https://api.target.com -w routes-large.kite --delay 200ms -t 20s -x 4 -j 1

# ffuf for general path/file discovery, with auto-calibration to cut noise
ffuf -u https://api.target.com/FUZZ -w api-endpoints-wordlist.txt -mc 200,201,301,302,401,403 -ac

# Subdomain-level API discovery (api.*, internal-api.*, etc.)
ffuf -u https://FUZZ.target.com -w subdomains.txt -ac
```

Always include `401` and `403` in matched codes, not just `200` — a 403 confirms the route exists and is doing *something*, which matters for method/role testing later. A route that 404s for every method is a dead end; a route that 403s only for some methods is worth a second look.

## 4. Hidden parameter discovery

```bash
# Arjun — hidden GET/POST parameter discovery
arjun -u https://api.target.com/user/profile -m GET -oT params.txt
arjun -u https://api.target.com/user/update -m POST --stable -oT params.txt

# ffuf as a parameter fuzzer
ffuf -u "https://api.target.com/api/endpoint?FUZZ=test" -w burp-parameter-names.txt -ac -mc 200

# x8 (Rust, fast, good for large parameter wordlists)
x8 -u https://api.target.com/endpoint -w params-large.txt
```

In Burp/Caido, **Param Miner** does the same passively/actively across many requests at once and is often faster to set up than scripting Arjun across an entire collection.

High-yield parameter names to always include in custom wordlists (these consistently show up in real mass-assignment and BFLA findings):

```
isAdmin  is_admin  admin
role  roles  permissions  scope  scopes
userType  accountType  status  approved  verified
internal  debug  test  sudo  bypass  override
godmode  impersonate_user_id  act_as  on_behalf_of
```

## 5. Method and content-type matrix

For every endpoint you've found, don't assume the documented verb is the only one that works:

```bash
for method in GET POST PUT PATCH DELETE OPTIONS HEAD; do
  echo "== $method =="
  curl -s -o /dev/null -w "%{http_code}\n" -X "$method" \
    -H "Authorization: Bearer $TOKEN" "https://api.target.com/endpoint/123"
done
```

Also vary `Content-Type` on write operations — some frameworks parse `application/xml` or `multipart/form-data` through a different code path than `application/json`, and that alternate path sometimes skips validation the JSON path has.

```
application/json
application/xml
application/x-www-form-urlencoded
multipart/form-data
text/plain
```

## 6. Version and shadow-API discovery

```
/v1/   /v2/   /v3/   /beta/   /internal/   /legacy/
```

Old API versions are one of the highest-signal, lowest-competition places to look: the current version usually gets re-reviewed every time auth logic changes, but `/api/v1/` often doesn't, because "nobody uses it anymore" — except it's frequently still live and reachable. The same applies to the API a mobile app uses, which is often a different (and less-audited) version than the one the web app calls. Pull the mobile app's APK/IPA traffic via a proxy (Burp/Caido + a rooted device or emulator, or Frida for certificate pinning bypass) to map that surface separately.

## Indicators worth grepping for across all of the above

```
/api/   /v1/   /v2/   /internal/   /admin/   /graphql
```
