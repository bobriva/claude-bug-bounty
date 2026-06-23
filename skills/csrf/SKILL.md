---
name: csrf
description: Detect and exploit Cross-Site Request Forgery (CSRF) and bypass common defenses — missing/predictable/non-session-bound tokens, double-submit cookie weaknesses, naive Referer/Origin validation, and SameSite Lax/Strict restrictions (GET-based state changes, client-side redirect gadgets, the 2-minute cookie-refresh window, sibling-subdomain trust) — for authorized bug bounty and pentest work. Covers Content-Type/method tricks that avoid CORS preflight (text/plain, form-urlencoded, charset switching), token leakage via unrelated endpoints or caching, clickjacking chained with "reflected but not saved" token errors, CORS-misconfiguration-assisted token theft, and multi-stage CSRF chains into account takeover (email/password change, MFA removal, admin actions, financial transactions) — plus a validation gate and report template. Use for any state-changing endpoint (account settings, password/email change, admin actions, payments) or writing up a CSRF finding for HackerOne, Bugcrowd, or YesWeHack.
---

# CSRF Testing

Same loop as the other skills in this set: **map → test → validate → report**. CSRF is one of the most over-reported bug classes in bug bounty — "the form has no token" gets submitted constantly with no working PoC behind it — and also one where a seemingly airtight defense (token *and* SameSite *and* Referer check) still falls in a meaningful fraction of real-world cases, if you push past the first failed bypass attempt instead of concluding "protected."

## Core principle: a CSRF finding is a working HTML page, not an observation

The only thing that actually proves CSRF is a self-contained exploit page that, when visited by an authenticated victim with no further interaction beyond what you've documented, causes the target action to execute. "There's no visible CSRF token in this form" is a lead. A page you hosted, tested against your own second session standing in for the victim, that actually fired the request and actually changed the target's state, is a finding.

## Before you start: map sensitive actions and the authentication model

```
1. List every state-changing endpoint: account settings, email/password
   change, MFA enrollment/removal, admin actions (role/permission
   changes, user creation), financial actions (transfers, purchases,
   billing), and anything else that mutates data based on the
   logged-in session.
2. Determine the authentication model per endpoint: session cookie
   (CSRF-relevant), bearer token/JWT in a header (generally NOT
   CSRF-relevant, since browsers don't auto-attach it cross-site),
   or both. Don't spend time on endpoints that only accept a
   header-delivered token with no cookie fallback.
3. For each cookie-authenticated state-changing endpoint, note what
   defenses appear present: CSRF token (form field or header),
   SameSite attribute on the session cookie, Referer/Origin checks.
```

## The methodology

| Phase | Goal | Reference |
|---|---|---|
| 1. Map & triage | Sensitive actions, auth model, apparent defenses | This file, above |
| 2. Token & header bypass | Missing/weak token validation, naive Referer/Origin checks | `references/token-and-header-bypass.md` |
| 3. SameSite & request-construction bypass | Cookie restriction gaps, Content-Type/method tricks for cross-origin requests | `references/samesite-and-request-tricks.md` |
| 4. Chaining | Clickjacking, CORS misconfig, XSS-assisted token theft, multi-stage chains | `references/clickjacking-and-chaining.md` |
| 5. Validate | Run every candidate through the gate below | This file, "Validation gate" |
| 6. Report | Working PoC, real impact, fast to triage | This file + `references/payloads-and-reporting.md` |

If the target authenticates primarily via OAuth/SSO, the OAuth-flow-specific CSRF (missing/unvalidated `state` parameter enabling account-linking abuse) is covered in the `authentication` skill's `oauth-testing.md` — don't duplicate that testing here, just confirm which model is in play first.

## Validation gate — run this before calling anything a finding

Every "no" means: keep working the bypass, downgrade to "suspected," or drop it.

1. **Do you have a working exploit page, actually tested?** Host it (Burp's exploit server, or your own domain), visit it from a second authenticated session standing in for the victim, and confirm the target action actually executed — not just that the request was crafted correctly in theory.
2. **Did you check all three defense layers, not just the obvious one?** A form with no visible token can still be protected by SameSite or a header-based Origin check; conversely a token-protected form can still be bypassed via SameSite/Referer gaps if the token check itself is weak. Confirm what's actually enforced before claiming a bypass — and don't stop at the first defense layer that blocks you; check whether a different layer is the weak one.
3. **Is the action genuinely sensitive?** Email/password change, MFA removal, admin actions, and financial transactions are high value. A preference toggle or a non-sensitive GET request is real but low severity — say so honestly rather than inflating it.
4. **Is user interaction realistic?** A PoC that requires the victim to be already logged in, click once, and nothing else is strong. One that depends on a narrow timing window (e.g., the SameSite Lax 2-minute cookie-refresh trick) or multiple precise actions is still valid but should state the precondition plainly — it affects severity.
5. **Is it a duplicate?** "Missing CSRF token on a low-value settings page" is one of the most commonly submitted, commonly duplicate bug classes that exists. Check disclosed reports before assuming novelty changes your framing.
6. **Would a triager agree, from your PoC alone, that an unmodified victim session would actually trigger this?** If reproducing it requires you to explain away several "but actually here's why this still works," tighten the PoC instead of the explanation.

### Patterns that usually die at this gate (don't submit these alone)

- "No CSRF token visible in the HTML form" with no actual cross-origin PoC tested, and no check of SameSite/Origin as alternate defenses
- A working PoC against a non-sensitive, low-impact action (e.g., changing a display preference) reported with inflated severity language
- Login CSRF or logout CSRF alone, with no further chain into something that matters (these are real but routinely scored Low/Informational without a follow-on impact)
- A SameSite bypass that only works within an unrealistic precondition you can't actually demonstrate happening for a real user (e.g., assuming the victim's cookie was set in the last 90 seconds, with no plausible reason it would be)
- Referer/Origin bypass demonstrated against an endpoint that turns out to also require the actual CSRF token, which you didn't also defeat

### Patterns that become valid once chained

| Weak alone | Chain it with | Becomes |
|---|---|---|
| Email-change CSRF | The existing password-reset flow, using the now-attacker-controlled email | Full account takeover |
| Token "reflects but doesn't save" on invalid/blank submission | Clickjacking (missing `X-Frame-Options`/`frame-ancestors`) to get the victim's own real click on the visible confirm button | Bypass of an apparently solid token check |
| CSRF token leaked via an unrelated endpoint (cached JS file, debug response, analytics beacon) | A standard CSRF PoC using the leaked token value | Full bypass despite token protection existing |
| CORS misconfiguration (reflects arbitrary `Origin` with `Access-Control-Allow-Credentials: true`) | A `fetch()`-based PoC that first reads the victim's own CSRF token via the misconfigured CORS, then replays it | One-shot exploit against a page that looks fully token-protected |
| `SameSite=Lax` cookie on a state-changing **GET** endpoint | Simple top-level navigation (`<script>location = '...'</script>` or even a plain link) | CSRF despite SameSite being set at all |
| `SameSite=Strict`, but an open redirect or XSS exists on a sibling subdomain | A same-site (not same-origin) request triggered from that sibling domain | Bypasses Strict, since the browser still considers it same-site |

Spend the extra time checking whether a "blocked" attempt can be re-routed through a different defense gap before concluding the endpoint is solid — in this category, the bugs that pay are almost always the ones where the first three obvious bypasses failed and the fourth, less obvious one didn't.

## Writing it up

**Title formula:** `CSRF in [exact endpoint] allows [actor] to [impact], bypassing [defense bypassed]`
Good: `CSRF in /api/account/email allows attacker to change a victim's email with no token, leading to account takeover via password reset`
Bad: `CSRF vulnerability found`

Always attach the actual PoC HTML and describe exactly which defense(s) were present and how each was defeated — a triager re-running your PoC file is the fastest path to confirmation. Full report template and severity guidance: `references/payloads-and-reporting.md`.

## Reference files in this skill

- `references/token-and-header-bypass.md` — CSRF token weaknesses (missing/predictable/cross-user/non-session-bound, double-submit cookie via session fixation), Referer/Origin naive validation bypass
- `references/samesite-and-request-tricks.md` — SameSite Lax/Strict bypass techniques, Content-Type/method tricks to construct cross-origin requests without preflight
- `references/clickjacking-and-chaining.md` — clickjacking-assisted bypass, CORS-misconfiguration token theft, XSS-assisted token theft, multi-stage CSRF chains, real disclosed-report patterns
- `references/payloads-and-reporting.md` — ready-to-use PoC templates, severity guidance by impact type, full report template
