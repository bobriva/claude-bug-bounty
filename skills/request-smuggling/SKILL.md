---
name: http-request-smuggling
description: "Advanced HTTP Request Smuggling detection, confirmation, and exploitation. Includes CL.TE, TE.CL, TE.TE, HTTP/2 downgrades, browser-powered desync, cache poisoning, and response queue poisoning. Zero false positives methodology."
tags: [request-smuggling, http-protocol, web-security, high-impact, bounty-worthy]
---

# HTTP Request Smuggling - Advanced Bug Hunting Skill

**Use this skill when:**
- Target has reverse proxy / CDN / load balancer
- HTTP/2 is detected on frontend
- Multiple backend systems exist
- Connection pooling/reuse is observed
- Unusual response timing detected
- Cache inconsistencies observed

**Impact:** CRITICAL
- Authentication bypass
- Access control bypass
- Cache poisoning
- Credential theft
- Session hijacking
- Response queue poisoning

---

## Quick Start Checklist

```
DISCOVERY PHASE
[ ] Identify proxy/load balancer (headers, timing, behavior)
[ ] Detect HTTP/2 support
[ ] Check connection reuse (keep-alive)
[ ] Identify backend diversity
[ ] Map frontend-to-backend routing

DETECTION PHASE
[ ] Test CL.TE variant
[ ] Test TE.CL variant
[ ] Test TE.TE variant
[ ] Test H2.CL variant (if HTTP/2 frontend)
[ ] Test H2.TE variant (if HTTP/2 frontend)
[ ] Test CL.0 variant (if applicable)

CONFIRMATION PHASE
[ ] Differential response analysis
[ ] Socket poisoning verification
[ ] Timing analysis (baseline vs attack)

EXPLOITATION PHASE
[ ] Header leakage (X-Forwarded-For, internal headers)
[ ] Cache poisoning chains
[ ] Access control bypass
[ ] Request capture/modification
[ ] Response queue poisoning
```

---

## Read Next

- `core-concepts.md` - Protocol fundamentals
- `detection-techniques.md` - Detailed detection methodology
- `exploitation-payloads.md` - Real-world exploit chains
- `false-positive-mitigation.md` - Confidence scoring
- `tool-integration.md` - Burp/custom tool setup
- `case-studies.md` - Real bounty examples

---

## Key Files Structure

| File | Purpose |
|------|---------|
| core-concepts.md | HTTP/1.1, HTTP/2, CL/TE mechanics |
| detection-techniques.md | Timing-based, differential response, socket poisoning |
| exploitation-payloads.md | Tested payload templates |
| false-positive-mitigation.md | Confidence scoring, validation |
| tool-integration.md | Burp macros, custom scripts |
| case-studies.md | Real CVE + HackerOne references |
| methodology.md | Step-by-step hunting process |
| advanced-techniques.md | Browser desync, H2 tunneling, edge cases |

