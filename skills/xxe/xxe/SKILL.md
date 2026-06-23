---
name: xxe
description: Detect and exploit XML External Entity (XXE) injection — file disclosure, SSRF (including cloud metadata access), blind XXE via out-of-band exfiltration and error-based extraction, XInclude abuse when only a single value is controllable, billion-laughs entity-expansion DoS, and XXE reached via file uploads (SVG, DOCX/XLSX/PPTX) or content-type smuggling into endpoints that don't advertise XML at all — for authorized bug bounty and pentest work. Covers the external-DTD parameter-entity OOB exfiltration technique, local-DTD repurposing for error-based extraction when OOB is blocked, FTP/base64 fallbacks for exfiltrating files with newlines or bad characters, UTF-7/UTF-16 encoding bypass for keyword filters, the Java jar:// protocol, and escalation toward RCE — plus a validation gate and report template. Use for any XML/SOAP API, SVG or Office-document upload, SAML flow, RSS/feed parser, or XML-configuration ingestion point, or writing up an XXE finding for HackerOne, Bugcrowd, or YesWeHack.
---

# XXE Testing

Same loop as the other skills in this set: **map → test → validate → report**. XXE rarely shows up in brand-new JSON-only APIs, but it persists in exactly the places that don't get re-audited often — legacy SOAP endpoints, SAML assertion parsing, file-upload pipelines that accept XML-based formats (SVG, DOCX/XLSX/PPTX) without anyone remembering those are XML, and "JSON" APIs that turn out to tolerate `Content-Type: text/xml` anyway because nobody locked that down. Most of the value in this category comes from finding XML parsers in places nobody thinks to look, not from a clever payload against an obvious target.

## Before you start: find every XML parser, not just the obvious ones

```
Obvious:    XML/SOAP APIs, .xml file uploads, RSS/Atom feed ingestion
Less obvious: SVG image uploads (SVG is XML), DOCX/XLSX/PPTX uploads
              (Office Open XML formats are zipped XML), SAML
              authentication flows, any REST endpoint that *might*
              tolerate an alternate Content-Type even if it normally
              receives JSON or form-encoded data
```
Every programming language's XML library (libxml in PHP, the JDK's built-in parsers in Java, lxml/xml.etree in Python, .NET's System.Xml) ships with external entity resolution enabled by default in many configurations — the vulnerability is usually an omission (nobody hardened the parser), not a deliberate feature, which is exactly why it shows up in unexpected corners of an application that nobody specifically built "for XML."

## The methodology

| Phase | Goal | Reference |
|---|---|---|
| 1. Discovery & basic XXE | Find XML parsers (including via content-type smuggling), confirm file read/SSRF | `references/detection-and-basic-xxe.md` |
| 2. Blind & OOB exfiltration | OAST detection, external-DTD exfiltration, error-based extraction, DoS | `references/blind-xxe-and-oob-exfiltration.md` |
| 3. Upload-based & XInclude | SVG/DOCX/XLSX/PPTX uploads, single-value injection points, Java jar:// | `references/upload-based-and-xinclude.md` |
| 4. Filter bypass & RCE escalation | Encoding bypass, expect:// wrapper, path-traversal/RCE chaining | `references/waf-bypass-and-rce-chains.md` |
| 5. Validate | Run every candidate through the gate below | This file, "Validation gate" |
| 6. Report | Real data recovered, reproducible PoC, fast to triage | This file + `references/payloads-and-reporting.md` |

## Validation gate — run this before calling anything a finding

Every "no" means: keep digging, downgrade to "suspected," or drop it.

1. **Did you recover real file content, confirm a real SSRF callback, or capture a real OOB interaction with actual exfiltrated data inside it?** A bare XML parsing error with no sensitive content and no follow-up OOB confirmation is a lead, not a finding.
2. **For blind XXE: did the out-of-band interaction carry real data**, via the external-DTD parameter-entity technique — not just a bare ping confirming the DOCTYPE was processed at all? A confirmed-but-empty OAST hit is real (state it as "confirmed XXE, exfiltration not yet demonstrated") but is a different, lower claim than full file disclosure.
3. **For SSRF via XXE: did the backend request actually reach the intended target**, confirmed via response content, timing, or an OAST interaction — not just that you declared an external entity pointing at it?
4. **For upload-based XXE: did you confirm the uploaded file was actually parsed by a vulnerable component end-to-end** (an OAST hit or file content tied specifically to processing your upload), not just that the upload was accepted?
5. **For DoS (billion laughs/entity expansion): did you confirm measurable resource impact while staying within authorized testing bounds?** Don't take down a production system to prove a DoS exists — describe the mechanism and a controlled, limited demonstration instead.
6. **If escalating toward RCE: did you stop at minimal, non-destructive proof?** Same responsible-scope principle used throughout this skill set — confirm the primitive, don't deploy a persistent foothold.
7. **Is it a duplicate?** Classic SOAP/legacy-API XXE is well-trodden ground on older targets; check disclosed reports, and let actual demonstrated impact (file content, SSRF target reached, real exfiltrated data) drive severity rather than novelty.

### Patterns that usually die at this gate (don't submit these alone)

- A generic XML parsing error with no sensitive content and no OOB follow-up attempted
- "The DOCTYPE was accepted with no error" with no entity resolution ever actually confirmed
- An SSRF-via-XXE payload sent with no confirmed callback or response difference proving the backend request fired
- A crafted malicious SVG/DOCX/XLSX uploaded successfully, with no confirmation the file was ever actually parsed by a vulnerable XML component (vs. just stored, or processed by a hardened parser/library that ignores DTDs)
- Claiming RCE based on a `jar://`/`expect://` payload being accepted with no actual command output or file-write confirmed

### Patterns that become valid once chained

| Weak alone | Chain it with | Becomes |
|---|---|---|
| OAST ping confirms blind XXE exists | External-DTD parameter-entity exfiltration technique, actually pulling real file content into the interaction log | Confirmed file disclosure, not just "injection exists" |
| SSRF via XXE reaching only `localhost`/internal hosts | The cloud provider's metadata endpoint specifically (`169.254.169.254`), with credentials actually extracted | Cloud credential exposure — often Critical instead of Medium |
| Out-of-band connections blocked entirely | A local DTD file already present on the server (common ones ship with most Linux XML/SVG/GNOME packages), repurposed for error-based extraction | Blind XXE remains exploitable even with no outbound network path at all |
| Direct XML endpoint hardened against XXE | An SVG/DOCX/XLSX upload feature elsewhere in the same application, sharing a less-hardened parsing library | Reaching the same underlying vulnerability through an indirect, less-audited path |
| Java `jar://` protocol confirmed reachable | A path-traversal-writable temp-file location from the archive-extraction step, combined with a separate template-injection/deserialization sink | Escalation toward RCE — note this is a distinct, separately-confirmed chain, not "XXE" on its own |

## Writing it up

**Title formula:** `XXE in [exact endpoint/feature] allows [actor] to [impact]`
Good: `Blind XXE via SVG avatar upload allows authenticated user to read arbitrary server files via out-of-band DNS/HTTP exfiltration`
Bad: `XXE vulnerability found`

Full payload cheat-sheet, severity guidance, and report template: `references/payloads-and-reporting.md`.

## Reference files in this skill

- `references/detection-and-basic-xxe.md` — finding XML parsers (including content-type smuggling), basic file read, SSRF via XXE
- `references/blind-xxe-and-oob-exfiltration.md` — OAST detection, external-DTD OOB exfiltration, error-based extraction, billion laughs DoS
- `references/upload-based-and-xinclude.md` — SVG/DOCX/XLSX/PPTX upload XXE, XInclude for single-value injection points, Java jar://
- `references/waf-bypass-and-rce-chains.md` — UTF-7/UTF-16 encoding bypass, expect:// wrapper, RCE escalation chains
- `references/payloads-and-reporting.md` — payload cheat-sheet, CVSS quick-reference, full report template
