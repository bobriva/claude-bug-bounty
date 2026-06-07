# Password Reset Poisoning

## Goal

Inject attacker-controlled domain into reset links.

---

## Indicators

Password reset emails.

Absolute URLs.

Host-derived links.

---

## Test

Change:

Host

X-Forwarded-Host

Observe email contents.

---

## Impact

Account takeover.

Credential theft.

Session compromise.