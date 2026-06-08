---
name: jwt
description: Comprehensive JWT vulnerability testing covering claim tampering, signature bypass, algorithm confusion, weak secrets, and header parameter injection.
---

# JWT Security Testing Skill

## When to Use This Skill

Use when testing:
- Authorization headers with Bearer tokens
- JWT stored in cookies
- OAuth/OIDC implementations
- API authentication using JWT
- Single Sign-On (SSO) systems
- Microservices with JWT tokens

---

## JWT Structure

Every JWT has 3 parts separated by dots:

```
Header.Payload.Signature

Example:
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyfQ.SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

### Header
```json
{
  "alg": "RS256",
  "typ": "JWT",
  "kid": "key-1",
  "jku": "https://...",
  "jwk": {...}
}
```

### Payload (Claims)
```json
{
  "sub": "1234567890",
  "name": "John Doe",
  "role": "user",
  "admin": false,
  "iat": 1516239022,
  "exp": 1516242622
}
```

### Signature
Cryptographic signature using algorithm from header.

---

## The 3 Main Categories

### 📁 Category 1: BASIC JWT ATTACKS (60% vulnerable)
- Claim Tampering
- alg=none Bypass
- Weak Secret Cracking

**Expected bounty:** $500-$5,000

**Files:**
- 1.1_claim-tampering.md
- 1.2_alg-none.md
- 1.3_weak-secret.md

---

### 📁 Category 2: HEADER PARAMETER INJECTION (30% vulnerable)
- JWK Injection
- JKU Injection
- KID Injection

**Expected bounty:** $1,000-$5,000+

**Files:**
- 2.1_jwk-injection.md
- 2.2_jku-injection.md
- 2.3_kid-injection.md

---

### 📁 Category 3: ADVANCED ATTACKS (10% vulnerable)
- Algorithm Confusion
- Key Confusion Advanced

**Expected bounty:** $2,000-$10,000+

**Files:**
- 3.1_algorithm-confusion.md
- 3.2_key-confusion-advanced.md

---

## Quick Decision Tree

```
Found JWT token?
│
├─ Try CATEGORY 1 (Basic Attacks) first
│  ├─ Modify "role" claim
│  ├─ Change "admin" to true
│  └─ Success? → Privilege escalation
│
├─ Try CATEGORY 2 (Header Injection) if Category 1 fails
│  ├─ Try JWK injection
│  ├─ Try JKU injection
│  └─ Success? → Token forgery
│
└─ Try CATEGORY 3 (Advanced) if Category 2 fails
   ├─ Try algorithm confusion
   └─ Success? → Full bypass
```

---

## Common JWT Algorithms

| Algorithm | Type | Risk |
|-----------|------|------|
| HS256 | HMAC | Secret cracking, algorithm confusion |
| HS384 | HMAC | Secret cracking, algorithm confusion |
| HS512 | HMAC | Secret cracking, algorithm confusion |
| RS256 | RSA | Algorithm confusion |
| RS384 | RSA | Algorithm confusion |
| RS512 | RSA | Algorithm confusion |
| ES256 | ECDSA | Generally safe |
| none | None | CRITICAL - No signature! |

---

## Common Claims to Target

| Claim | Purpose | Test |
|-------|---------|------|
| sub | User ID | Change to another user |
| role | User role | Change user→admin |
| admin | Admin flag | Change false→true |
| permissions | Permissions | Add admin permissions |
| exp | Expiration | Extend to year 9999 |
| aud | Audience | Try different value |

---

## 5-Phase Methodology

### Phase 1: Discovery
Find JWT tokens in:
- Authorization header
- Cookies
- localStorage/sessionStorage
- URL parameters
- Request body

### Phase 2: Decode
Use jwt.io, Burp JWT Editor, or jwt_tool

### Phase 3: Analyze
Check header, payload, signature

### Phase 4: Test Modifications
Try tampering with claims

### Phase 5: Escalate & Exploit
Prove impact with admin access

---

## Tools Needed

- **Burp Suite** (JWT Editor extension)
- **jwt.io** (Online decoder)
- **jwt_tool** (Python)
- **hashcat** (Secret cracking)

---

## Real Bounty Examples

### Example 1: Claim Tampering → $2,000
```
Finding: "role" modified to "admin"
Impact: Admin access gained
Bounty: $2,000
```

### Example 2: Algorithm Confusion → $5,000
```
Finding: RS256→HS256 using public key as secret
Impact: Full token forgery
Bounty: $5,000
```

### Example 3: JKU Injection → $3,500
```
Finding: JKU points to attacker server
Impact: Arbitrary token creation
Bounty: $3,500
```

---

## Bounty Summary

- Basic attacks: $500-$5,000
- Header injection: $1,000-$5,000+
- Advanced attacks: $2,000-$10,000+

**Average:** $2,000-$4,000 per vulnerability

---

## Start Testing

1. Find JWT token
2. Pick a category
3. Follow the guide
4. Document findings
5. Report bounty!
