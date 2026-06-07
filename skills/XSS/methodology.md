# XSS Methodology

## Phase 1 - Reflection Discovery

Inject:

xss12345

Observe:

- Response body
- DOM
- Attributes
- JavaScript

---

## Phase 2 - Context Identification

Determine:

- HTML
- Attribute
- JavaScript
- URL
- CSS
- Template Literal

---

## Phase 3 - Payload Selection

Choose payloads based on context.

Never use generic payloads blindly.

---

## Phase 4 - Filter Analysis

Test:

<

>

"

'

`

/

Observe encoding behavior.

---

## Phase 5 - Exploitation

Assess:

- Session theft
- CSRF bypass
- Account takeover
- Stored propagation