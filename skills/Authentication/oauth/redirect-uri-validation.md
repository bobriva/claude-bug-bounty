# Redirect URI Testing

## Basic Manipulation

Add:

- Query parameters
- Fragments
- Additional paths

Example:

https://trusted.com/callback?test=1

---

## Prefix Validation Bypass

Examples:

https://trusted.com.evil.com

https://trusted.com@evil.com

---

## Duplicate Parameters

redirect_uri=trusted
redirect_uri=evil

---

## Localhost Abuse

localhost.evil.com

127.0.0.1.evil.com

---

## Directory Traversal

/oauth/callback/../../redirect

---

## Open Redirect Chaining

Combine:

Valid redirect_uri
+
Open Redirect

Goal:

Leak code/token