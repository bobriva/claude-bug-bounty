# JWT-Specific Testing

JWTs are a self-contained credential format, which means the testing surface is the token's *structure*, not just the endpoints that use it. Treat every JWT-authenticated API as having this entire checklist on top of the standard BOLA/BFLA testing.

A JWT has three base64url parts separated by dots: `header.payload.signature`. The header's `alg` field tells the server how to verify the signature — and that field is exactly what most JWT attacks target, because it's attacker-controlled but trust-critical.

## 1. `alg: none`

Some libraries historically accepted an unsigned token if the header claims no algorithm is used.

```
1. Decode the JWT header and payload (base64url).
2. Set header to: {"alg":"none","typ":"JWT"}
3. Modify payload claims freely (e.g. role -> admin).
4. Re-encode header.payload, drop the signature entirely (trailing dot, empty signature).
5. Send the token. If it's accepted, signature verification isn't enforced for this alg value.
```

## 2. Algorithm confusion (RS256 → HS256)

If the server is configured to verify RS256 tokens (asymmetric: signed with a private key, verified with a public key) but the verification library is alg-agnostic, you can sometimes get it to verify an HS256 (symmetric) token using the **public key as the HMAC secret** — which you, as an attacker, may already have, since public keys are frequently exposed (JWKS endpoint, TLS cert, documentation).

```
1. Obtain the RS256 public key (check /.well-known/jwks.json, /jwks, or the app's TLS certificate).
2. Change header alg from RS256 to HS256.
3. Modify payload claims as desired.
4. Sign the new header.payload using HS256 with the RSA public key (in the correct PEM/format the library expects) as the secret.
5. Send it. If the server's verify() call is algorithm-agnostic and trusts the alg header, this validates.
```

If the public key isn't directly exposed, it can sometimes be derived mathematically from two existing valid RS256 tokens (tools: `rsa_sign2n`). Burp's **JWT Editor** extension automates most of this workflow with a GUI.

## 3. Weak / brute-forceable HMAC secret

If the app uses HS256 with a guessable secret (default values, short strings, dictionary words):

```bash
jwt_tool <token> -C -d wordlist.txt   # dictionary/brute-force the HMAC secret
hashcat -m 16500 jwt.txt wordlist.txt # GPU-accelerated alternative
```

A cracked secret means you can forge arbitrary valid tokens — this is typically a critical finding because it's full authentication bypass, not just a single-account issue.

## 4. Header-parameter injection (`kid`, `jku`, `x5u`, `jwk`)

These optional header fields tell the verifier *where* to find the key, and if the server fetches/uses them without restriction, you control the trust anchor:

- **`kid` (Key ID) injection** — if the server looks up the verification key in a database/file by `kid`, try path traversal (`kid: ../../dev/null` with an empty-string secret) or SQL injection in the lookup.
- **`jku` (JWK Set URL) injection** — point `jku` at a JWKS file you host, containing your own key pair; sign the token with your private key, and the server fetches your public key from your URL and trusts it.
- **`x5u` (X.509 URL) injection** — same idea with a certificate URL.
- **`jwk` (embedded key) injection** — some implementations accept a public key embedded directly in the JWT header and trust it instead of using a fixed server-side key.

All four follow the same logic: the server should never let attacker-supplied header fields choose which key verifies the attacker's own token.

## 5. Claims tampering (after establishing you can forge or replay valid signatures)

Once any of the above gives you a way to produce server-accepted tokens, test which individual claims actually get enforced:

```
- Change the user/subject ID claim
- Change role/permission claims
- Remove `exp` (expiration) entirely
- Remove `nbf` (not-before)
- Change `aud` (audience) to a different service's expected value
- Change `iss` (issuer)
- Add arbitrary custom claims and see if any are blindly trusted downstream
```

## 6. Cross-service token reuse

In a microservices setup, obtain a token issued for service A and replay it unmodified against service B, C, etc. If `aud` (or an equivalent service-binding claim) isn't actually checked, a token scoped to a low-trust service might work everywhere.

## Tools

| Tool | Use |
|---|---|
| Burp **JWT Editor** extension | GUI-based header/payload editing, signature recalculation, key management |
| `jwt_tool` | CLI scanning and exploitation — covers most of the above automatically (`-M` mode flags many issues) |
| `rsa_sign2n` | Derive an RSA public key from two existing RS256 tokens when it's not directly exposed |
| `hashcat` mode 16500 | GPU brute-force of HS256 secrets |

## What makes a JWT finding confirmed, not theoretical

You need a token you forged or modified yourself to actually be accepted by the server *and* to grant something a legitimately-issued token for your account wouldn't — e.g., access to another user's data, or an elevated role's functionality, demonstrated with a real follow-up request. Decoding a token and noting "this could be tampered with" is not yet a finding.
