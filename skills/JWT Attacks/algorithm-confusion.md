# Algorithm Confusion

Also Known As

Key Confusion

---

Scenario

Server expects:

RS256

Attacker sends:

HS256

Server public key becomes HMAC secret.

---

Workflow

1. Obtain public key

2. Convert key format

3. Change alg to HS256

4. Sign token using public key

5. Test acceptance

---

Impact

Authentication bypass

Privilege escalation

User impersonation