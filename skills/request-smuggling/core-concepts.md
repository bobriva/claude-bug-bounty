# Core Concepts - HTTP Request Smuggling Fundamentals

## HTTP Protocol Basics

### Content-Length (CL)
- Indicates exact number of bytes in message body
- Parser reads exactly N bytes, then treats next byte as new message start
- **Critical:** Disagreement on CL value = desynchronization

```
POST / HTTP/1.1
Host: example.com
Content-Length: 13

Hello, World!      <- Exactly 13 bytes
```

### Transfer-Encoding (TE)
- Chunks body into segments prefixed with size in hex
- Last chunk is size 0
- **Critical:** Different TE parsing = desynchronization

```
POST / HTTP/1.1
Host: example.com
Transfer-Encoding: chunked

D              <- 13 bytes (hex: D)
Hello, World!
0              <- End of chunks
```

### HTTP/2 Differences
- Binary framing (DATA frames)
- No chunked encoding (HTTP/2 handles framing differently)
- Content-Length can still exist and conflict
- Stream multiplexing allows multiple requests on same connection
- ALPN negotiation determines HTTP/1.1 vs HTTP/2

---

## The Desynchronization Triangle

```
CLIENT (Browser/Tool)
    ↓
PROXY/FRONTEND (e.g., Nginx, HAProxy, CloudFlare)
    ↓
BACKEND (e.g., Apache, Express, Django)
```

**Exploitation requires:**
1. Frontend and backend disagree on request boundaries
2. Shared connection reuse (keep-alive)
3. Attacker can inject headers/body

---

## Smuggling Variants Explained

### CL.TE (Content-Length vs Transfer-Encoding)

**Frontend:** Trusts `Content-Length`
**Backend:** Trusts `Transfer-Encoding: chunked`

```
POST / HTTP/1.1
Host: example.com
Content-Length: 13
Transfer-Encoding: chunked

0

SMUGGLED       <- Frontend: end of message (CL=13)
               <- Backend: considers this a new request
```

**Attack Impact:** 
- Inject entire new request that frontend doesn't see
- Backend processes as second request
- Next legitimate client request gets poisoned response

**Detection Signal:**
- Request appears to complete normally to client
- Backend waits for more data (timeout observable)
- Or: differential response when smuggle delivers 404 request

---

### TE.CL (Transfer-Encoding vs Content-Length)

**Frontend:** Trusts `Transfer-Encoding: chunked`
**Backend:** Trusts `Content-Length`

```
POST / HTTP/1.1
Host: example.com
Transfer-Encoding: chunked
Content-Length: 200

D              <- 13 bytes
Hello, World!
0              <- Frontend: end of chunks (TE wins)

[145 bytes of garbage/attack]  <- Backend: still waiting for 200 bytes total
```

**Attack Impact:**
- Frontend forwards only chunk data to backend
- Backend expects more data based on CL value
- Remaining garbage/attack payload becomes prefix of next request

**Detection Signal:**
- Timeout waiting for more data
- Next request gets corrupted
- Differential response from injected prefix

---

### TE.TE (Transfer-Encoding Obfuscation)

**Both:** Claim to support Transfer-Encoding, but parse differently

```
POST / HTTP/1.1
Host: example.com
Transfer-Encoding: chunked
Transfer-Encoding: cow     <- Unknown encoding, often ignored

D
Hello, World!
0

SMUGGLED
```

OR

```
Transfer-Encoding: xchunked
Transfer-Encoding:[space]chunked   <- Tab or space variation
Transfer-Encoding
 : chunked                          <- Obfuscated via CRLF
```

**Attack Impact:**
- One side ignores obfuscated TE header
- Other side respects it
- Result: parser disagreement

**Detection Signal:**
- Response inconsistencies
- May require multiple obfuscation attempts

---

### H2.CL (HTTP/2 + Content-Length)

**Frontend:** HTTP/2 (ignores CL in frame-based protocol)
**Backend:** HTTP/1.1 (trusts CL)

```
HTTP/2 REQUEST (from browser):
:method: POST
:path: /
:scheme: https
:authority: example.com
content-length: 13

Hello, World!SMUGGLED

Frontend interpretation: body = "Hello, World!SMUGGLED" (full payload)
Backend interpretation: body = "Hello, World!" (CL=13)
                       next msg = "SMUGGLED..."
```

**Attack Impact:**
- HTTP/2 frontend sends full payload
- Content-Length truncation on backend
- Excess becomes request prefix for next client

**Detection Signal:**
- Very reliable when HTTP/2 frontend + HTTP/1.1 backend confirmed
- Timing + differential response

---

### H2.TE (HTTP/2 + Transfer-Encoding Injection)

**Frontend:** HTTP/2 (removes injected TE)
**Backend:** HTTP/1.1 (processes TE if not sanitized)

Rare but possible if proxy injects TE on backend connection.

---

### CL.0 (Client-Side Desync)

**Frontend:** Respects Content-Length: 0
**Browser:** Ignores CL:0, sends full body anyway (browser bug/feature)

```
POST / HTTP/1.1
Content-Length: 0

Full body that browser sends anyway
```

**Attack Impact:**
- Browser sends payload marked as "no body"
- Backend receives it as request start
- Next request gets corrupted

**Targets:** Single-server sites, VPNs, CDNs with weak validation

---

## Socket Poisoning

When one request's body bleeds into next request's parsing:

```
ATTACK: Injects "GET /admin HTTP/1.1\r\n\r\n"
VICTIM: Sends normal "GET /profile HTTP/1.1\r\n\r\n"

Backend receives: "GET /admin HTTP/1.1\r\n\r\nGET /profile HTTP/1.1\r\n\r\n"
Result: /admin accessed without victim's authorization
```

This is the **core exploitation mechanism** for bounty-worthy vulnerabilities.

---

## Key Indicators of Vulnerable Architecture

1. **Reverse Proxy + Backend Mismatch**
   - Nginx (CL only) + Apache (prefers TE)
   - HAProxy + custom backend

2. **HTTP/2 Frontend + HTTP/1.1 Backend**
   - Most common in modern setups
   - Downgrade via ALPN

3. **Connection Reuse Without Synchronization**
   - Keep-Alive enabled
   - No request-level synchronization checks

4. **Unusual Header Processing**
   - Proxy removes certain headers
   - Different normalization rules

5. **Caching Layer**
   - Enables cache poisoning attacks
   - Responses cached per request URL (ignoring body)

---

## Why This Matters for Bounty Hunting

✓ **High Impact:** Bypasses authentication, accesses admin panels
✓ **Hard to Find:** Requires multi-layer setup understanding  
✓ **Reproducible:** Once found, consistently exploitable
✓ **Proof Concrete:** Network traces + differential responses
✓ **Real Impact:** Can demonstrate account takeover, data access

**Typical Bounties:** $1,000 - $50,000+ (depends on scope + impact)

