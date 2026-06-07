# CSP Bypass

## Review CSP Header

Identify:

script-src

img-src

object-src

frame-src

nonce

hash

---

## Common Weaknesses

Unsafe external domains

Misconfigured nonces

Misconfigured hashes

Policy injection

---

## Testing

Can external requests occur?

Can images load?

Can events execute?

Can CSP directives be injected?

---

## Targets

script-src-elem

report-uri

trusted CDNs