# Username Enumeration & Brute Force

## Username enumeration

The goal: find any observable difference between "this account exists" and "this account doesn't," anywhere identity is checked — login, registration, password reset, and even some API endpoints that take a username/email as a lookup key.

**1. Error message differences**
```
Login as known-invalid user    -> "Invalid username or password"
Login as known-valid user      -> "Invalid username or password"   (looks safe...)
Registration with existing email -> "Email already in use"          (...but registration leaks it)
Password reset, unknown email  -> "If that email exists, we sent a link"
Password reset, known email    -> Same message, but check response length/timing below
```
Always test the same oracle across *all three* surfaces (login, registration, reset) — a team can fix the message on login and forget registration leaks the same information.

**2. Response length / structure differences**
Diff the raw response bytes for a valid vs. invalid identifier, even when the rendered message looks identical — whitespace, included fields, or HTML structure sometimes differ by a few bytes only for one case.

**3. HTTP status code differences**
```
Valid username, wrong password   -> 200 / 302
Invalid username                 -> 401 / 404
```
Some apps get this right for the message but leak it in the status code instead.

**4. Timing differences**
Valid usernames often take measurably longer (password hash comparison actually runs) than invalid ones (rejected before hashing). To test reliably:
```
1. In Burp Intruder, send the same login request with a list of candidate
   usernames and a fixed dummy password.
2. Add columns for "Response received" and "Response completed" time.
3. Sort by response time — an outlier that's consistently slower across
   repeated requests is a valid username.
```
The signal is sometimes faint. If it doesn't show up with a normal password, try an exceptionally long password (200+ characters) — some implementations only show the timing gap once the comparison/hashing step itself becomes measurably expensive.

**5. Account lockout as an oracle**
If lockout only triggers for accounts that exist (because there's nothing to "lock" for a username with no account), repeatedly failing login for a candidate username and then succeeding once with the *correct* password but getting a lockout message reveals the username was valid all along.

## Turning enumeration into a real finding

Enumeration alone is weak (see the validation gate in `SKILL.md`). To make it count:
1. Confirm a valid username via any technique above.
2. Move directly into the brute-force section below against that specific, confirmed-valid username.
3. If a credential is found, log in and demonstrate access — that's the actual finding; the enumeration was just the recon step that made it efficient.

## Brute force & credential stuffing

```bash
# Password spray with hydra (only against accounts you control / explicitly authorized)
hydra -L usernames.txt -P passwords.txt target.com http-post-form \
  "/login:username=^USER^&password=^PASS^:Invalid credentials"

# ffuf as a lighter-weight alternative
ffuf -u https://target.com/login -X POST \
  -d "username=victim&password=FUZZ" \
  -w passwords.txt -H "Content-Type: application/x-www-form-urlencoded" \
  -mc 200,302 -fc 401
```

Always check the program's policy on automated credential attacks before running this against anything but your own test accounts — many programs explicitly disallow it even when the login endpoint is in scope.

## Rate-limit and lockout bypass techniques

If a lockout/rate-limit exists, these are the standard ways defenders' implementations fail — test each, since apps frequently trust client-supplied headers for the "real" client IP:

```
X-Originating-IP: 127.0.0.1
X-Forwarded-For: 127.0.0.1
X-Remote-IP: 127.0.0.1
X-Remote-Addr: 127.0.0.1
X-Client-IP: 127.0.0.1
X-Host: 127.0.0.1
X-Forwarded-Host: 127.0.0.1
```

Other angles, roughly in order of how often they work in practice:

- **Increment the spoofed IP per request** (`127.0.0.1`, `127.0.0.2`, `127.0.0.3`...) rather than reusing one value — some implementations only block a given IP value after N hits, so a fresh value each time resets the count.
- **Double up the header**: `X-Forwarded-For: <real>, X-Forwarded-For: 127.0.0.1` — some parsers take the wrong element of the list.
- **Add a throwaway parameter** to make each request look unique to a gateway that rate-limits on exact URL/param match: `/login?x=1`, `/login?x=2`, ...
- **Rotate `User-Agent`** if limiting is partially keyed on client fingerprint.
- **Log in successfully between attempts** (or every few attempts) — some counters reset on any successful auth event, not just after a cooldown window.
- **Switch request format**: form-encoded → JSON, or `POST` → `PUT`, in case the limiter is only wired into one code path.
- **GraphQL batching/aliasing**, if the login mutation is exposed over GraphQL — see the `api-security` skill's `graphql-testing.md` for the technique; it can turn N HTTP-level rate-limited attempts into one HTTP request containing N attempts.

### Confirming, not just observing

"The header trick was accepted and didn't immediately error" is not proof of a bypass. Confirm by actually running enough attempts past the point where the *un*-bypassed request would have been blocked, against your own test account, and showing the difference side by side (blocked without the header, not blocked with it).
