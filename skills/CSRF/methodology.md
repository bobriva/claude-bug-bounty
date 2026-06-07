# CSRF Methodology

## Phase 1 - Find Sensitive Actions

Examples:

- Email change
- Password change
- Profile update
- Role modification
- Payment actions
- API key generation

---

## Phase 2 - Authentication Analysis

Check:

- Session cookies
- JWT
- API Tokens

Determine whether browser automatically sends credentials.

---

## Phase 3 - CSRF Protection Analysis

Check:

- CSRF Tokens
- SameSite Cookies
- Referer Validation
- Origin Validation

---

## Phase 4 - Generate PoC

Use:

Burp Generate CSRF PoC

or

Manual HTML Form

---

## Phase 5 - Validate Impact

Can attacker trigger action?

Can attacker gain account control?

Can attacker affect administrators?