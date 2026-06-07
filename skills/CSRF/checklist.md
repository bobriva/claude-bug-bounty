# CSRF Checklist

## Discovery

[ ] State-changing endpoints

[ ] Account settings

[ ] Password functions

[ ] Admin functionality

## Authentication

[ ] Session cookies

[ ] Browser credentials

## CSRF Protection

[ ] CSRF token present

[ ] Token validated

[ ] SameSite enabled

[ ] Referer validation

[ ] Origin validation

## Bypass Testing

[ ] Missing token

[ ] Invalid token

[ ] Reused token

[ ] Cross-user token

[ ] Referer removal

[ ] Referer manipulation

## Impact

[ ] Email change

[ ] Password change

[ ] MFA removal

[ ] Account takeover