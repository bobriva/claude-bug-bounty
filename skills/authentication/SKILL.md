---
name: authentication
description: Systematically test login, registration, password reset, MFA/2FA, session management, and OAuth/SSO/OIDC flows for authentication logic flaws, username enumeration, brute force, rate-limit/lockout bypass, session fixation, redirect_uri bypass, PKCE downgrade, scope escalation, and full authentication bypass or account takeover, for authorized bug bounty and pentest work. Covers enumeration via timing/response/status differences, rate-limit bypass headers, parameter-pollution bypass, MFA response/status manipulation, session fixation/rotation, password-reset poisoning via Host header injection, redirect_uri bypass (prefix/@-confusion/localhost/path-traversal), open-redirect-to-OAuth-token-theft chaining, state/CSRF and PKCE testing, and OAuth account-linking takeover via unverified email — plus a validation gate and report template that kill false positives. Use for login, password reset, 2FA/MFA, sessions, "Login with Google/GitHub/Facebook"/SSO, or writing up a finding for HackerOne, Bugcrowd, or YesWeHack.
---

# Authentication Testing

Same closed loop as the API security skill: **map → test → validate → report**. Authentication bugs are some of the highest-paying findings in bug bounty (often straight to account takeover), but they're also where AI-assisted hunters generate the most noise — "no rate limit after 10 requests" and "username enumeration via error message" get submitted constantly and rejected constantly, because on their own they rarely prove real-world impact. This skill is built to push past that noise to the findings that actually convert into payouts.

## Core principle: leads, not bugs

Treat every signal here — a slightly different error message, a slightly longer response time, a 4xx you can flip to a 200 — as a **lead to chase, not a bug to report**. Before writing anything up, ask: *what's the most boring, intended explanation for this? Have I actually completed the attack end-to-end, or just observed a hint that it might be possible?* Most authentication "vulnerabilities" submitted to programs are partial observations dressed up as findings — a real one ends with you actually logged into an account you shouldn't have access to, or a concrete password/token in hand.

## Before you start: scope and ethics check

This is the one skill category where running the "obvious" test can itself violate program rules:

- **Check whether automated credential attacks are allowed.** Many programs explicitly prohibit real brute-force/credential-stuffing against live accounts, even in scope. If unclear, ask the user, or default to demonstrating the *mechanism* (e.g., "rate limiting is absent — here's 30 attempts against my own test account with no block, extrapolate the risk") rather than actually compromising an account that isn't yours.
- **Use accounts you/the user control.** Username enumeration, brute force, and session attacks should be proven against the user's own test accounts (ideally two: a low-privilege "victim" and a separate "attacker" identity), not real third-party users, unless the program has explicitly authorized that.
- **Confirm what's in scope**: which login surfaces (web, mobile API, SSO, partner integrations) and what's excluded.

## The methodology

| Phase | Goal | Reference |
|---|---|---|
| 1. Map the auth surface | Every place identity is established or re-checked | This file, below |
| 2. Enumeration & brute force | Usernames, passwords, lockout, rate limiting | `references/enumeration-and-bruteforce.md` |
| 3. Logic flaws & bypass | Missing/duplicate params, alternate paths, header tricks | `references/logic-flaws-and-bypass.md` |
| 4. MFA & session | 2FA bypass, fixation, rotation, hijacking | `references/mfa-and-session.md` |
| 5. Password reset | Reset poisoning via Host header, token issues | `references/password-reset.md` |
| 6. OAuth/SSO | Detection, redirect_uri bypass, open-redirect chaining, state/PKCE, scope escalation, account-linking takeover | `references/oauth-testing.md` |
| 7. Validate | Run every candidate through the gate below | This file, "Validation gate" |
| 8. Report | Concrete, chained, fast to triage | This file + `references/payloads-and-reporting.md` |

## Phase 1: map the authentication surface

Before testing anything, list every place this application establishes or re-verifies identity — there are usually more than the obvious login form:

- Primary login (web), and any **separate mobile-app or legacy login endpoint** — these frequently implement weaker checks than the current web flow.
- Registration (and whether it leaks account existence — "email already in use" is itself a username-enumeration oracle).
- Password reset / forgot-password.
- MFA/2FA enrollment, verification, and disable/backup-code flows.
- SSO/OAuth login (which providers, and how accounts get linked/created).
- "Remember me" / persistent-login cookies.
- Any internal "login as" / impersonation feature for support staff.
- Session re-verification points: sensitive actions that should but might not re-check the current credential (changing email, password, payment method, disabling 2FA).
- If the target is JWT-based, this skill hands off to the `api-security` skill's `jwt-testing.md` for the token-structure attacks (alg confusion, none-alg, kid/jku injection) — don't duplicate that here, just confirm the token format first.

## Validation gate — run this before calling anything a finding

Every "no" here means: keep digging, downgrade to "suspected," or drop it.

1. **Did you complete the attack end-to-end?** A successful login as an account you shouldn't control, a password you cracked and used, or a session you hijacked and replayed — not "the response looked different" or "the request was accepted."
2. **Did it happen on a real account under realistic conditions?** Testing against your own test account with full knowledge of the setup is fine for proving the mechanism; the writeup still needs to make clear this would work against an arbitrary victim, and ideally show it does (with a second account standing in for the victim).
3. **Is the impact authentication-relevant, not cosmetic?** A different error message alone is, at most, a Low/Informational enumeration note — it only becomes a real finding when chained into something with consequence (brute force, account takeover, targeted phishing setup).
4. **Was this allowed under program scope/rules?** Re-check before running anything resembling brute force or credential stuffing against accounts that aren't the user's own.
5. **Is it a duplicate?** Username enumeration, "no rate limit on login," and "session doesn't expire" are some of the most over-submitted bug classes that exist — check disclosed reports first.
6. **Would a triager agree this grants unauthorized access, in under a minute of reading?** If your steps require five preconditions before the bypass works, tighten the report or reconsider the severity claim.

### Patterns that usually die at this gate (don't submit these alone)

- Username enumeration via a timing/response/status difference, with no follow-on brute-force or account-compromise demonstration
- "No rate limit observed" after a handful of manual attempts, with no actual brute-force result
- Session cookie lacking `Secure`/`HttpOnly` flags, with no demonstrated theft/hijack
- Long session timeout/expiry, with no hijacking demonstrated
- 2FA code technically reusable, with no second successful login actually performed using the reused code
- OAuth missing a `state` parameter, with no completed CSRF-driven account-linking demonstrated
- Password reset link containing a token in the URL, with no demonstrated leak path (referrer, logs, or your own host-header capture) actually shown to work
- A `redirect_uri` bypass technique accepted by the authorization server, with no actual code/token captured on attacker-controlled infrastructure
- PKCE missing or downgradable, with no demonstrated interception/reuse of an actual authorization code
- An open redirect found somewhere near an OAuth login, with no completed chain showing the code/token actually landing on attacker-controlled infrastructure

### Patterns that become valid once chained

| Weak alone | Chain it with | Becomes |
|---|---|---|
| Username enumeration | A working brute-force/credential-stuffing path against that confirmed username | Full account compromise |
| No rate limit on login/OTP | A completed brute-force run that actually lands a valid credential or OTP | Critical — demonstrated ATO |
| Session fixation (ID doesn't rotate post-login) | An injection point to set the victim's session ID (XSS, response header injection) | Full account takeover |
| Host header reflected into password-reset email | A realistic capture path for the resulting token (your own listener, or a shown referrer leak) | Account takeover via reset poisoning |
| OAuth account linking by email, unverified | Ability to register/control that email on the OAuth provider's side | Cross-provider account takeover |
| 2FA "success" field/status code manipulable | Demonstrated full login completed past the 2FA gate using the manipulated response | Critical MFA bypass |
| Race condition on a 2FA cancel/verification link | Multiple parallel requests shown to all stay valid (Burp "send group in parallel" / Turbo Intruder) | MFA bypass via race condition |
| Correctly-restricted `redirect_uri` (whitelisted domain) | An unrelated open redirect found on that same whitelisted domain | Full OAuth code/token theft — often the highest-paying chain in this skill |
| PKCE missing or downgradable (S256 → plain) | A way to intercept the authorization code in transit (network position, logs, referrer leak) | Account takeover without ever holding a client_secret |
| `redirect_uri` prefix/`@`-confusion bypass accepted | Code/token actually captured on attacker-controlled infrastructure via the bypassed URL | Confirmed token theft, not just "validation looks loose" |

Spend the extra 15 minutes trying to chain a weak signal before writing it up alone — this is where most of the payout difference between a Low and a Critical actually comes from.

## Writing it up

Same standard as the API security skill: lead with impact, not preamble; one tight chain beats three speculative paragraphs. Full template, severity guidance, and tone notes are in `references/payloads-and-reporting.md` — reuse it as-is for consistency across reports.

**Title formula:** `[Mechanism] in [exact flow] allows [actor] to [impact]`
Good: `Session fixation in /login allows pre-authentication session ID to remain valid post-login, enabling full account takeover via forced session`
Bad: `Authentication bug found`

## Reference files in this skill

- `references/enumeration-and-bruteforce.md` — username enumeration techniques, brute-force/rate-limit bypass headers and tricks
- `references/logic-flaws-and-bypass.md` — missing/duplicate parameters, alternate endpoints, header-based bypass, injection in auth fields
- `references/mfa-and-session.md` — 2FA/MFA bypass techniques, session fixation/rotation/hijacking
- `references/password-reset.md` — password reset poisoning via Host header injection, token issues
- `references/oauth-testing.md` — OAuth/OIDC detection, redirect_uri bypass, open-redirect chaining, state/CSRF, PKCE, scope escalation, account-linking takeover
- `references/payloads-and-reporting.md` — ready-to-use payloads, CVSS quick-reference, full report template
