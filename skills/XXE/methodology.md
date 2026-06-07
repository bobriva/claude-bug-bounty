# XXE Methodology

## Phase 1 - Discovery

Identify:

- XML requests
- SOAP endpoints
- SVG uploads
- Office document uploads
- XML APIs

---

## Phase 2 - Basic XXE

Test:

External entities

Observe:

- File disclosure
- Response changes
- Error messages

---

## Phase 3 - SSRF Testing

Target:

- Internal services
- Metadata services
- Localhost services

---

## Phase 4 - Blind XXE

Use:

- OAST
- DNS callbacks
- Error-based extraction

---

## Phase 5 - Hidden Attack Surface

Test:

- XInclude
- SVG uploads
- DOCX uploads
- XML content-type smuggling