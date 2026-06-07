---
name: information-disclosure
description: Systematic methodology for discovering information leakage that can be chained into higher severity vulnerabilities.
---

# Information Disclosure Methodology

Information disclosure is not always a standalone vulnerability.

Treat leaked information as:

- Reconnaissance
- Attack surface expansion
- Exploitation assistance

Goal:

Convert leaked information into:

- IDOR
- SSRF
- RCE
- Authentication Bypass
- Business Logic Abuse
- Privilege Escalation

Mindset:

Never stop at the disclosure itself.

Ask:

"What does this information allow me to do next?"