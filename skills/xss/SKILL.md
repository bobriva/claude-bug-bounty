---
name: xss
description: "Advanced Cross-Site Scripting (XSS) detection, exploitation, and impact assessment. Covers Reflected XSS, Stored XSS, DOM XSS, CSP bypasses, dangling markup attacks, and stored worms. Zero false positives methodology."
tags: [xss, client-side-injection, web-security, high-impact, bounty-worthy]
---

# Cross-Site Scripting (XSS) - Advanced Bug Hunting Skill

**Use this skill when:**
- User input is reflected in responses
- User content is stored and displayed
- JavaScript dynamically processes data
- DOM manipulation occurs
- Frontend frameworks exist
- API responses contain user data
- HTML rendering from user input

**Impact:** CRITICAL to HIGH
- Account takeover (session/credential theft)
- Session hijacking
- CSRF token theft + action execution
- Admin panel compromise
- Stored XSS worms (affect all users)
- Privilege escalation
- Malware distribution
- Phishing attacks

---

## Quick Start Checklist

```
DISCOVERY PHASE
[ ] Identify all input points (GET, POST, headers, etc.)
[ ] Identify all output points (responses, DOM, etc.)
[ ] Map user input to output reflection
[ ] Identify execution contexts
[ ] Determine if input stored or reflected

TESTING PHASE
[ ] Test reflected XSS (basic)
[ ] Test stored XSS (persistence)
[ ] Test DOM XSS (client-side)
[ ] Test event handler execution
[ ] Test context-specific payloads
[ ] Test filter bypass techniques

VALIDATION PHASE
[ ] Verify JavaScript execution
[ ] Confirm user control of payload
[ ] Assess reproducibility
[ ] Determine attack impact
[ ] Check CSP bypass if present

EXPLOITATION PHASE
[ ] Session/credential theft
[ ] CSRF token extraction
[ ] Account takeover demonstration
[ ] Admin access demonstration
[ ] Stored XSS propagation potential
```

---

## Read Next

- `core-xss-concepts.md` - XSS fundamentals and contexts
- `reflection-detection.md` - Finding XSS injection points
- `payloads-by-context.md` - Context-specific exploit payloads
- `filtering-bypass.md` - Bypassing input/output filters
- `csp-bypass-techniques.md` - Content Security Policy evasion
- `stored-xss-hunting.md` - Stored XSS discovery and escalation
- `dom-xss-analysis.md` - DOM-based XSS detection
- `false-positive-mitigation.md` - Zero false positive validation
- `exploitation-techniques.md` - Real-world impact demonstration
- `case-studies.md` - Real bounty examples with amounts
- `tool-integration.md` - Burp, automation, custom scripts
- `methodology.md` - Complete hunting workflow

---

## Key Files Structure

| File | Purpose |
|------|---------|
| core-xss-concepts.md | XSS types, contexts, mechanics |
| reflection-detection.md | Finding injection points |
| payloads-by-context.md | Payload templates per context |
| filtering-bypass.md | WAF/filter evasion |
| csp-bypass-techniques.md | CSP policy bypasses |
| stored-xss-hunting.md | Stored XSS detection |
| dom-xss-analysis.md | DOM-based XSS |
| false-positive-mitigation.md | Confidence scoring |
| exploitation-techniques.md | Impact demonstration |
| case-studies.md | Real bounty examples |
| tool-integration.md | Tools + automation |
| methodology.md | Step-by-step workflow |

---

## XSS Impact Hierarchy

### CRITICAL (CVSS 9-10) - $50K+
- Stored XSS on admin panel
- Stored XSS affecting all users (worm)
- Account takeover via session theft
- Privilege escalation
- Admin account compromise

### HIGH (CVSS 7-8.9) - $15K-50K
- Stored XSS on user profile
- Stored XSS in comments/messages
- DOM XSS with CSRF bypass
- Session hijacking with credential theft

### MEDIUM (CVSS 5-6.9) - $3K-15K
- Reflected XSS (requires click)
- Self-XSS (requires user interaction)
- Limited context XSS
- DOM XSS (without cascading impact)

### LOW (CVSS <5) - $100-3K
- Reflected XSS in error message
- Self-XSS without escalation
- XSS in non-critical feature

---

## Bounty Multipliers

| Factor | Multiplier | Example |
|--------|-----------|---------|
| Type | 1-2× | Stored > Reflected |
| Scope | 1-5× | All users > single user |
| Impact | 1-10× | ATO > info leak |
| Chainable | 1-3× | Can chain with other vulns |
| Automation | 1-5× | Can automate attack |

**Example:** Reflected XSS ($3K) × Stored (2×) × All Users (5×) = $30K+

