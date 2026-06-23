# Clickjacking-Assisted Bypass & CSRF Chaining

Most of the highest-paying CSRF reports aren't "no token at all" — they're a seemingly solid defense defeated by combining it with something else. This file covers the chaining techniques and includes patterns drawn from real disclosed reports, to calibrate what "good" looks like.

## Clickjacking-assisted token bypass

If a state-changing form rejects an invalid/missing token but still **renders the submitted values back into the page** with a visible confirm/retry button — rather than rejecting cleanly with no further action available — and the page lacks `X-Frame-Options`/`Content-Security-Policy: frame-ancestors`, you can chain the two:

```
1. Your CSRF page silently submits the target form with the values
   you want (e.g., a new email address) and an invalid/blank token,
   in a hidden iframe.
2. The server responds with an error, but the page now shows the
   attacker-chosen values pre-filled, with a "Submit again" / "Confirm"
   button — and critically, this resubmission no longer needs a fresh
   token, or re-checks a token that's now present in that rendered page.
3. Your page frames that exact response and uses standard clickjacking
   (an invisible/transparent overlay positioned over the visible
   button) to get the victim's own genuine click to land on it.
4. The victim's real click completes the action — the server sees a
   legitimately-clicked submission with a valid-looking token, because
   it is one; you just orchestrated which values were in it.
```

This defeats token validation that looks airtight in isolation, purely because of how the error-handling path differs from the success path. Always test what the *error* response looks like for a deliberately-broken submission, not just whether the happy path is protected.

## CORS-misconfiguration-assisted token theft

If an endpoint reflects an arbitrary `Origin` header back in `Access-Control-Allow-Origin` *and* sets `Access-Control-Allow-Credentials: true`, a malicious page can read authenticated responses cross-origin — including an endpoint that returns the victim's own CSRF token:

```javascript
fetch('https://target.com/api/get-csrf-token', { credentials: 'include' })
  .then(r => r.json())
  .then(data => {
    return fetch('https://target.com/api/account/email', {
      method: 'POST',
      credentials: 'include',
      headers: {
        'Content-Type': 'application/json',
        'X-CSRF-Token': data.csrf_token
      },
      body: JSON.stringify({ email: 'attacker@evil.com' })
    });
  });
```
This turns a page that's "fully protected" by a strict, server-validated, session-bound CSRF token into a one-shot exploit, because the attacker simply reads the victim's real token first via the CORS hole, then uses it normally. Always check CORS configuration on any endpoint that returns a CSRF token in its response body, not just on the state-changing endpoint itself.

## Token leakage via an unrelated surface

CSRF tokens that are strict and session-bound everywhere they're *supposed* to appear can still leak through a surface nobody thought to check. A real disclosed pattern: a CSRF-protecting token value also happened to be embedded, unintentionally, inside an unrelated cached JavaScript file served from the same origin (a worker/service-worker script that included account-specific stats data for an entirely different feature). Because that file was reachable and the token value sat in it as a string, an attacker who could get a victim to load it could extract the token and use it to forge a normal CSRF request — completely bypassing the strength of the token mechanism itself.

When auditing a target with strong token protection, specifically check:
```
- Any JS/JSON file the app serves that embeds per-user data (analytics
  configs, feature-flag payloads, service workers, "account stats"
  bundles) for an accidentally-included token value
- Whether the token appears in any URL (and would therefore leak via
  Referer to a third party loading on the same page)
- Whether the token is cached by a CDN/proxy in a way that could
  serve one user's token to another (web cache deception)
```

## XSS-assisted CSRF (and the reverse: CSRF-assisted XSS delivery)

If a stored or reflected XSS exists anywhere on the same origin, it trivially defeats every CSRF defense covered in this skill — token, SameSite, Referer/Origin — because the malicious request now originates from genuinely same-origin JavaScript with full access to read the real token and the browser correctly treats it as same-site/same-origin in every respect. When auditing a target that has even a minor, hard-to-escalate XSS finding elsewhere, note explicitly that it can be chained into a full CSRF bypass on every form, and consider scoring/reporting it with that combined impact in mind. The reverse direction also occurs: a login/logout CSRF used purely to force a victim onto an attacker-prepared account where a stored-XSS payload then fires — useful context for why "just" a login CSRF can matter when chained deliberately.

## Multi-stage CSRF chains

Some of the highest-severity disclosed CSRF reports aren't one request — they're two or three forged requests fired in sequence from the same exploit page, each one a legitimate action on its own, that together add up to full account takeover (for example: forge a request that changes a security question's answer, then forge a second request that uses the now-attacker-known answer to reset the password). When you find one CSRF-able state-changing endpoint, immediately check whether any *other* sensitive endpoint can be chained right after it in the same exploit page — most CSRF PoCs can fire several forged requests sequentially with no extra victim interaction beyond the original page load.

## Login CSRF — when it's worth reporting on its own

Forcing a victim to be logged into an attacker-controlled account (rather than logging an attacker into the victim's) is lower-impact by default, but becomes meaningful when chained: it can be used to capture data the victim subsequently enters (search history, payment details typed into "their" session which is actually the attacker's), or to deliver a stored-XSS payload that fires once the victim is in that session, or simply to plant tracking data linking the victim's real-world identity to an account the attacker controls. Report it as a chain with a stated downstream effect, not as a bare "login CSRF exists."

## What makes a chained finding confirmed, not theoretical

Every link in the chain needs its own end-to-end proof — the clickjacking overlay actually positioned and tested against a real click, the CORS-read token actually used successfully in a follow-up forged request, the leaked token actually extracted from the real file and actually accepted by the server. A chain is only as strong as its weakest unverified link; don't write up a 3-step chain where step 2 was only theorized.
