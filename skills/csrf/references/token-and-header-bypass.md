# CSRF Token & Header-Based Defense Bypass

## Token validation weaknesses

Work through these against any endpoint that includes a CSRF token, in order:

```
1. Remove the token entirely. Some implementations only validate the
   token *when present* and silently skip validation when it's missing.

2. Send an empty/blank token value. Distinct from omitting it — some
   code paths treat "" differently from "absent."

3. Reuse a previously-valid (but now stale/expired) token.

4. Swap in a token issued for a *different* user/session — confirms
   whether the token is actually bound to the session presenting it,
   or just checked for "is this a token-shaped value that exists
   somewhere in our system."

5. Replace the token with a same-length random string. Some
   implementations only check length/format, not the actual value.

6. Check predictability: collect several tokens in a row (new
   session each time) and look for sequential, timestamp-derived, or
   otherwise low-entropy patterns rather than genuinely random values.
```

**The "reflected but not saved" trap (test this even when the token check looks strict):** on some implementations, submitting an invalid/blank token causes the server to render the submitted values back into the page (to "show what you tried to change") alongside an error like "CSRF token invalid, please resubmit" — without actually having saved anything yet. If the page then has a visible "submit again" / "confirm" button that, when clicked by the legitimately-authenticated user, completes the action without requiring a fresh valid token, you can deliver this as a two-stage attack: your CSRF page submits the invalid-token request, then frames the resulting confirmation page and uses clickjacking to get the victim to click "confirm" themselves. See `clickjacking-and-chaining.md` for the framing technique.

## Double-submit cookie weaknesses

A "double-submit cookie" pattern checks only that a token value in the request body matches a token value in a cookie — with no server-side record of which tokens are legitimate. This is weaker than a synchronizer token tied to server-side session state:

```
1. Confirm the pattern: does the server appear to validate the token
   purely by comparing request-body value to cookie value, with no
   apparent server-side lookup?
2. If so, and if you have any way to set a cookie in the victim's
   browser for the target origin (a session-fixation bug, a
   subdomain that can set cookies for the parent domain, or a
   response-header-injection bug elsewhere on the site), you can
   choose the token value yourself: set the victim's CSRF cookie to
   a value you pick, then submit a CSRF request using that same
   value in the body.
3. Confirm end-to-end: the victim's browser must actually carry your
   chosen cookie value at the time of the forged request.
```

## Referer header validation bypass

```
1. Remove the Referer header entirely:
   <meta name="referrer" content="no-referrer">
   (or content="never" — both suppress the header on outgoing requests
   from the page hosting your PoC)
   Test whether validation only fires when a Referer is present at all.

2. If an allowlist/substring check is used, test classic bypasses:
   https://trusted.com.attacker.com/        (trusted.com as a prefix
                                              of the attacker's own
                                              domain — passes a naive
                                              "contains" or "starts
                                              with" check)
   https://attacker.com/trusted.com/        (trusted.com appended as
                                              a path segment, to beat
                                              a naive "contains" check)

3. Test whether the check distinguishes scheme/port — submitting from
   http://trusted.com when only https://trusted.com is expected, or
   vice versa, sometimes still passes.
```

## Origin header validation bypass

`Origin` is generally a more reliable signal than `Referer` (browsers don't allow it to be suppressed the way `Referer` can be, and it's sent on POST/PUT/DELETE even when stricter), but implementations still get the check wrong:

```
- Missing-Origin fallback: some servers only check Origin when it's
  present, and certain request types/older browser behavior can omit
  it — test whether removing it (where the request path allows) skips
  validation entirely.
- Null origin: requests from a sandboxed iframe, a data: URI, or a
  redirect chain in some configurations send Origin: null — test
  whether "null" is accepted as if it were a legitimate value.
- Same allowlist/substring weaknesses as Referer above — a regex or
  substring check against Origin is just as exploitable as one
  against Referer.
```

## What makes a token/header bypass finding confirmed, not theoretical

An actual cross-origin request, sent from a page you hosted (not just curl with a forged header, which doesn't prove a *browser* will reproduce it), that the server accepted and acted on — checked against a real before/after state on the target account. Confirming a header *can* be spoofed via curl is necessary groundwork, but the finding requires showing the browser-driven attack page actually works.
