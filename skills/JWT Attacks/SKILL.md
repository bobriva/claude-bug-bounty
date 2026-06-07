---
name: jwt
description: Detect and exploit JWT vulnerabilities including claim tampering, alg=none, weak secrets, algorithm confusion, JWK injection, JKU injection, and KID injection.
---

# JWT Security Testing

Use this skill when:

- Authorization header contains Bearer token
- JWT cookie exists
- OAuth/OIDC uses JWT
- API authentication uses JWT

Objectives:

1. Decode token
2. Analyze claims
3. Test signature validation
4. Test header parameter abuse
5. Test key confusion
6. Escalate privileges

Read:

- methodology.md
- claim-tampering.md
- algorithm-confusion.md