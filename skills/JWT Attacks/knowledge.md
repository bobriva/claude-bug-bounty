# JWT Knowledge

Core Components

Header

Payload

Signature

---

Common Algorithms

HS256

HS384

HS512

RS256

ES256

---

High Risk Vulnerabilities

Claim Tampering

alg=none

Weak Secret

Algorithm Confusion

JWK Injection

JKU Injection

KID Injection

---

Common Tools

Burp JWT Editor

hashcat

jwt_tool

jwt.io

---

Hunting Mindset

1. Decode

2. Modify Claims

3. Test Signature Validation

4. Test Header Parameters

5. Escalate Privileges