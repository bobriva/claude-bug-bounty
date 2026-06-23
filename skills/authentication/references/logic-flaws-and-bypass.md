# Authentication Logic Flaws & Bypass

These are bugs in the *logic* of how identity is checked, independent of brute force or token attacks. They tend to be high-severity when they hit, because they often mean "skip authentication entirely" rather than "guess a credential faster."

## Parameter manipulation

```
# Missing parameter — drop the password field entirely
POST /login
username=admin

# Empty / null password
POST /login
username=admin&password=

POST /login
{"username":"admin","password":null}

# Duplicate parameters — some frameworks take the first value, others the
# last, others concatenate or array-ify; the auth check might validate a
# different value than the one actually used downstream
POST /login
username=victim&username=attacker&password=attackerpass

POST /login
{"username":"victim","password":["wrongpass","correctpass"]}
```

The logic to look for: does the **validation** step check one value while the **session-issuing** step uses a different one (e.g., validates the first `username` instance but issues a session for the last)? That mismatch is the actual bug — confirm it by logging in as `victim` while supplying `attacker`'s credentials in the duplicated field and checking which account's session you receive.

## Injection in auth fields

Classic but still found regularly, especially on legacy or homegrown auth:

```
# SQL injection
username: admin' OR '1'='1
username: admin'--

# NoSQL injection (MongoDB-style operator injection)
{"username": "admin", "password": {"$ne": null}}
{"username": {"$ne": null}, "password": {"$ne": null}}
```

## Alternate authentication paths

The current, well-reviewed login form is rarely the only way in:

- **Mobile-app API login** — often a different endpoint/version than the web login, frequently with looser validation because it's reviewed less.
- **Legacy/deprecated login endpoints** found during recon (`/v1/login` next to a current `/v2/login`) — old logic sometimes never got the same fixes.
- **SSO/partner-integration login** — a separate code path that might not enforce the same lockout/MFA rules as direct login (see `password-reset-and-oauth.md`).
- **"Remember me" / persistent-login cookies** — check whether the persistent token is predictable, doesn't rotate, or grants more than the session it's meant to extend (e.g., bypasses an MFA step that a fresh login would require).
- **Internal "login as" / impersonation tooling** — if discoverable, check whether it requires a genuinely privileged role or just a guessable internal flag/header.

## Header and routing-based bypass

Some apps make authorization decisions based on headers that are meant to be set by trusted infrastructure (a load balancer, a WAF) but aren't actually validated as coming from there:

```
GET /admin-only-page
X-Custom-IP-Authorization: 127.0.0.1

GET /admin-only-page
X-Forwarded-Host: internal.target.com

POST /api/delete
X-HTTP-Method-Override: GET
```

Also test directly **forced-browsing past an auth step**: navigate straight to a post-login or post-2FA page/endpoint without completing the prior step, and if blocked, retry while setting the `Referer` header to the URL of the page that's supposed to precede it — some implementations check "did this request appear to come from the right step" instead of "did the user actually complete it."

## What makes a logic-flaw finding confirmed, not theoretical

You need to actually be authenticated as an account afterward — a session cookie issued, an authenticated page loading, a subsequent authenticated API call succeeding — using credentials/parameters that should not have been sufficient. "The request didn't error" or "the parameter was accepted" without confirming you're actually logged in as that account is not yet a finding.
