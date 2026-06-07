# API Security Methodology

## Phase 1 - Recon

Identify:

- REST APIs
- JSON APIs
- Mobile APIs
- Internal APIs

Look for:

/api/
/v1/
/v2/
/graphql/

---

## Phase 2 - Documentation Discovery

Check:

/swagger

/swagger-ui

/swagger/index.html

/openapi.json

/openapi.yaml

/api-docs

/docs

---

## Phase 3 - Endpoint Enumeration

Identify:

Resources

Methods

Parameters

Authentication

Versioning

---

## Phase 4 - Hidden Surface Discovery

Test:

- Hidden endpoints
- Hidden parameters
- Unsupported methods
- Alternative content types

---

## Phase 5 - Vulnerability Assessment

Focus:

- IDOR
- Mass Assignment
- Privilege Escalation
- Sensitive Data Exposure