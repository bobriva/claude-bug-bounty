# Host Header Cache Poisoning

## Goal

Store malicious response in cache.

---

## Indicators

Cache-Control

CDN

Reverse Proxy

---

## Test

Inject Host variations.

Observe:

Cache HIT

Cache MISS

Response differences

---

## Impact

Persistent XSS

Open Redirect

Victim Redirection

Content Manipulation