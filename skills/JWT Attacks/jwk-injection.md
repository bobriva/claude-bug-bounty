# JWK Injection

Goal

Embed attacker-controlled key.

---

Header Parameter

jwk

---

Workflow

Generate RSA keypair.

Embed public key.

Sign token.

Submit token.

---

Indicators

Server accepts embedded key.

Arbitrary JWT creation.

Authentication bypass.