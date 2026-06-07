---
name: xxe
description: Detect and exploit XML External Entity vulnerabilities including file disclosure, SSRF, blind XXE, XInclude abuse, and XXE via file uploads.
---

# XXE Skill

Use this skill when:

- XML requests exist
- SOAP exists
- SVG uploads exist
- DOCX processing exists
- XML APIs exist
- XML parsers are present

## Objectives

1. Identify XML attack surface
2. Test entity resolution
3. Retrieve local files
4. Test SSRF via XXE
5. Test blind XXE
6. Test XInclude
7. Test XML-based uploads

## High Value Findings

- Arbitrary File Read
- SSRF
- Internal Service Discovery
- Credential Disclosure
- Cloud Metadata Access
- Infrastructure Enumeration

Refer to:

- methodology.md
- file-read.md
- blind-xxe.md
- xinclude.md