# Filter Bypass & RCE Escalation

## Encoding bypass for naive keyword filters

Many defenses block XXE by blacklisting keywords (`DOCTYPE`, `ENTITY`, `SYSTEM`) in the request body — but XML parsers are required to support multiple character encodings, and most keyword filters only inspect the ASCII surface form.

**UTF-16:**
```bash
cat payload.xml | iconv -f UTF-8 -t UTF-16BE > utf16_payload.xml
```
Send with the corresponding `encoding="UTF-16"` (or BE/LE-specific) declaration in the XML prolog and the appropriate `Content-Type`/charset. The parser decodes and processes it normally; an ASCII-only filter sees only binary noise where the blacklisted keywords would be.

**UTF-7:**
```xml
<?xml version="1.0" encoding="UTF-7"?>
+ADw-+ACE-DOCTYPE+ACA-data+ACA-+AFs-...
```
UTF-7 represents the same characters using a completely different byte sequence (the example above is the encoded form of `<!DOCTYPE data [...`). This only works if the parser is configured to accept arbitrary declared encodings — but many are, by default, since rejecting non-UTF-8 input isn't something most developers think to restrict. Always include the XML prolog with the matching `encoding="UTF-7"` declaration, or the parser won't know to decode it that way.

## Hexadecimal, decimal, and CDATA encoding of entity values

When the *value* inside an entity declaration (not the keywords themselves) is being filtered for specific substrings like `file://`:
```xml
<!ENTITY xxe "&#x66;&#x69;&#x6c;&#x65;&#x3a;&#x2f;&#x2f;&#x2f;&#x65;&#x74;&#x63;&#x2f;&#x70;&#x61;&#x73;&#x73;&#x77;&#x64;">  <!-- hex-encoded file:///etc/passwd -->
<!ENTITY xxe "&#102;&#105;&#108;&#101;&#58;&#47;&#47;&#47;&#101;&#116;&#99;&#47;&#112;&#97;&#115;&#115;&#119;&#100;">  <!-- decimal-encoded -->
<!ENTITY xxe "<![CDATA[file:///etc/passwd]]>">
```

## Content-type and structural disguises

```
- Convert between JSON and XML representations of the same body (see
  `detection-and-basic-xxe.md`) when a filter only inspects requests
  it expects to be XML-shaped, missing ones that started as JSON.
- Embed the payload inside a format-specific wrapper the filter
  doesn't specifically parse for XXE — an XLIFF translation file, a
  Karaf/OSGi features.xml deployment descriptor, or any other
  XML-based config format the target happens to ingest — since
  generic XXE filters are often written against the "obvious" XML
  API shape and miss format-specific ingestion points entirely.
```

## RCE escalation patterns

XXE's path to RCE is almost always feature abuse (a powerful protocol handler or a chain into a different vulnerability class) rather than a flaw in XML parsing itself — treat each of these as a separate, distinctly-confirmed escalation, not an automatic consequence of XXE existing.

**PHP `expect://` wrapper** (only if the `expect` PHP extension is installed and enabled — uncommon, but worth checking):
```xml
<svg xmlns="http://www.w3.org/2000/svg" xmlns:xlink="http://www.w3.org/1999/xlink">
  <image xlink:href="expect://id" width="20" height="20"/>
</svg>
```
If the extension is present, this executes the given command directly. Confirm with a benign command (`id`, `whoami`) and stop there — the same responsible-scope principle used throughout this skill set.

**Jar protocol + path traversal chaining** (Java only): as covered in `upload-based-and-xinclude.md`, the `jar://` protocol downloads and extracts a remote archive to a predictable temporary path. If you can separately find or induce a path-traversal-capable write or include elsewhere in the application, the two primitives combined can sometimes escalate to writing a file to an attacker-chosen location, or including/executing the extracted content through a different vulnerable sink (template injection, deserialization). This is a multi-vulnerability chain — document each half's independent confirmation in your report, not just the final combined claim.

**XSLT-based RCE** (a related but technically distinct feature from XXE — Extensible Stylesheet Language Transformations): some XML processing pipelines also support XSLT, and Java/.NET XSLT processors have historically offered extensions that let a stylesheet invoke arbitrary methods or scripts. If the application performs any XSLT transformation on user-influenced XML/stylesheets, treat it as separate attack surface worth testing on its own — it's commonly found alongside XXE-vulnerable endpoints (the same parsing pipeline often supports both) but isn't itself an XXE technique.

**Confirming, not just attempting:** for any RCE escalation, you need an actual command's output (direct or via OAST callback) — not "the wrapper/extension appears to be present" or "the payload was accepted." Stop immediately once you have one clean, benign, non-destructive proof; don't pivot further, install persistence, or access other systems.
