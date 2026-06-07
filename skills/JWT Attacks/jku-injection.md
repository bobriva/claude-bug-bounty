# JKU Injection

Goal

Force server to fetch attacker-controlled JWK Set.

---

Header Parameter

jku

---

Workflow

Host malicious JWKS.

Point jku to attacker server.

Sign token.

Submit token.

---

Indicators

Outbound request.

Accepted token.

---

Impact

Full JWT forgery.