# TE.TE

Both support Transfer-Encoding.

One side ignores obfuscated header.

Examples:

Transfer-Encoding : chunked

Transfer-Encoding:[tab]chunked

Transfer-Encoding: xchunked

Transfer-Encoding
 : chunked

---

Goal

Trigger parser disagreement.