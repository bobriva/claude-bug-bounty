# MFA/2FA Bypass & Session Management

## MFA/2FA bypass techniques

Test these against the actual 2FA verification step, after a normal first-factor login:

**1. Response manipulation**
Intercept the verification response and flip the result before it reaches the client logic that decides whether to proceed:
```
Original response:  {"success": false, "verified": false}
Modified response:  {"success": true,  "verified": true}
```
Only relevant when the *client* is trusted to act on this field rather than the server independently gating access on the next request — confirm by checking whether the next authenticated request actually succeeds, not just whether the UI changed screens.

**2. Status code manipulation**
```
Original: HTTP/1.1 401 Unauthorized   (or 403)
Modified: HTTP/1.1 200 OK
```
Same caveat — only matters if a subsequent real authenticated action then succeeds.

**3. Forced browsing past the 2FA gate**
Navigate directly to the post-2FA / dashboard URL without submitting a code. If blocked, retry with the `Referer` header set to the 2FA verification page's URL.

**4. Code leakage**
- Check the *triggering* request's response (the one that sends the code) for the code itself accidentally included.
- Check JS files loaded around the 2FA step for the code or its generation logic exposed client-side.

**5. Code reusability and missing expiry**
Use the same OTP twice. Wait past the stated expiry window and try it again. Either succeeding is a finding once you've completed an actual second login with the reused/expired code.

**6. Brute-forcing the OTP**
If there's no rate limiting on the verification step (check the techniques in `enumeration-and-bruteforce.md` first to rule out a bypassable limiter), a 4-6 digit numeric OTP is brute-forceable in a practical number of requests. Burp's Turbo Intruder is the standard tool for doing this fast enough that the OTP's validity window doesn't expire first.

**7. Backup code abuse**
Apply the same response/status manipulation and brute-force techniques to the backup-code flow specifically — it's frequently implemented separately from the primary 2FA check and inherits none of its protections.

**8. Trivial/placeholder codes**
Try `000000`, `123456`, or an empty/null code outright — some implementations have a leftover test/bypass value.

**9. Race conditions**
Fire several requests in parallel (Burp "Send group in parallel" or Turbo Intruder) at:
- The 2FA verification endpoint itself, with different guesses, to see if rate limiting only counts sequential requests.
- A "cancel this 2FA request" / "this wasn't me" link sent via email or push — if multiple such links are issued (e.g., one per device or one per retry) and only the clicked one gets invalidated, the others may remain valid indefinitely.

**10. Clickjacking on the disable-2FA page**
If the page that lets a user disable their own 2FA can be framed (no `X-Frame-Options`/`frame-ancestors`), check whether a clickjacking attack against a logged-in victim could be used to disable their 2FA without their knowledge — this needs to be chained with a realistic delivery mechanism to be more than theoretical.

**11. Session established pre-2FA, reused**
If the server issues *any* session token before the 2FA step completes, check whether that pre-2FA token already grants authenticated access on its own, by replaying it directly against an authenticated endpoint without ever submitting a code.

### Confirming, not just observing

Every technique above needs to end with **an actual authenticated action succeeding** that should have required passing 2FA. A manipulated response that merely changes what the UI displays, without a subsequent real authenticated request succeeding, is not a finding.

---

## Session management

**Session fixation** — does the session identifier issued *before* login remain valid and tied to the same account *after* login?
```
1. Visit the login page (unauthenticated), capture the session cookie value.
2. Log in normally.
3. Check whether the post-login session cookie is the same value.
4. If unchanged: an attacker who can set a victim's pre-login session ID
   (via a subdomain, XSS, or a session-ID-in-URL pattern) can pre-establish
   a known session, then wait for the victim to log into that same session.
```

**Session rotation** — beyond login, does the session ID rotate on privilege changes (e.g., a password change, becoming an admin, re-authenticating)? Failing to rotate after a password change is particularly notable: it means a session hijacked *before* the legitimate user changed their password remains valid *after*, defeating the point of the password change.

**Session invalidation on logout** — capture a session token, log out, then replay the captured token against an authenticated endpoint. If it still works, logout isn't actually invalidating server-side state (only clearing the client-side cookie).

**Concurrent session limits** — log in from two clients with the same credentials; check whether the app allows unlimited concurrent sessions when it shouldn't, or whether logging in fixed-factor-only from a second device silently extends trust without re-verifying MFA on either.

**Session token predictability** — if tokens are sequential, timestamp-based, or otherwise low-entropy (inspect several issued in a row), they may be guessable; this is a different bug class from fixation and needs its own proof (an actual guessed/predicted token from a different account, used successfully).

### What makes a session finding confirmed, not theoretical

A specific captured/predicted/fixated session token that you replayed and that granted access — ideally to a second account standing in for "the victim" — not just "the cookie looked unusual" or "rotation wasn't observed in one test."
