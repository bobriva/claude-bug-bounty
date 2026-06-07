# Reflected XSS

## Definition

User input is reflected immediately in the response.

---

## Discovery

Inject:

xss12345

Locate reflection.

---

## Common Sources

Query parameters

POST parameters

Headers

Path segments

---

## Basic Payloads

<script>alert(1)</script>

<img src=1 onerror=alert(1)>

---

## Validation

Verify JavaScript execution.