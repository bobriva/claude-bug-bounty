---
name: csrf
description: Detect and exploit Cross-Site Request Forgery vulnerabilities and bypass common CSRF defenses.
---

# CSRF Skill

Use this skill when:

- Session cookies are used
- Sensitive actions exist
- State-changing requests exist
- Account management functions exist

## Objectives

1. Identify state-changing endpoints
2. Determine authentication model
3. Assess CSRF protections
4. Test token validation
5. Test SameSite behavior
6. Test Referer validation
7. Assess impact

## High Value Findings

- Account Takeover
- Email Change
- Password Change
- MFA Removal
- Admin Actions
- Financial Transactions

Refer to:

- methodology.md
- token-bypass.md
- samesite-bypass.md
- referer-bypass.md