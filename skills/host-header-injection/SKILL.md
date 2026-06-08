---
name: host-header
description: Comprehensive HTTP Host Header vulnerability exploitation covering password reset poisoning, cache poisoning, routing SSRF, virtual host discovery, and authentication bypass.
---

# HTTP Host Header Attacks

## When to Use This Skill

Use this skill when testing:
- Web applications with reverse proxies
- Multiple domains sharing infrastructure
- CDN configurations
- Password reset functionality
- Absolute URLs in responses/redirects
- Load balancers or WAF systems
- Microservices architectures
- Cloud-hosted applications

---

## What is the Host Header?

HTTP Host header identifies which domain the request is for:

```
GET / HTTP/1.1
Host: target.com
User-Agent: Mozilla/5.0
```

**Why it matters:**
- Tells server which application to serve
- Used for virtual host routing
- Influences generated URLs
- May be logged or cached
- Often trusted by application

---

## The Vulnerability

**Problem:** Many applications trust the Host header without validation

```
Server receives:
GET / HTTP/1.1
Host: attacker.com  ← ATTACKER CONTROLS THIS

Server trusts it and:
- Generates links using attacker.com
- Routes to attacker-controlled vhost
- Sends password reset links to attacker.com
- Caches response with attacker.com key
```

**Result:** Can chain into password reset poisoning, cache poisoning, SSRF, etc.

---

## The 3 Main Categories

### 📁 **Category 1: BASIC ATTACKS** (50% of applications vulnerable)

**Focus:** Basic host header manipulation and discovery

**Covers:**
- ✅ Host header reflection in responses
- ✅ Absolute URL generation with attacker host
- ✅ Override header testing (X-Forwarded-Host, X-Host, etc)
- ✅ Virtual host discovery (admin, dev, staging)
- ✅ Hidden application enumeration

**Expected bounty:** $200-$2,000 per finding

**Files:**
- 1.1_host-header-reflection.md
- 1.2_host-override-headers.md
- 1.3_virtual-host-discovery.md

---

### 📁 **Category 2: ADVANCED ATTACKS** (30% of applications vulnerable)

**Focus:** Exploitation with infrastructure consideration

**Covers:**
- ✅ Password reset link poisoning
- ✅ Cache poisoning (CDN, reverse proxy)
- ✅ Routing-based SSRF
- ✅ Absolute URL hijacking
- ✅ Multi-domain exploitation chains

**Expected bounty:** $500-$5,000 per finding

**Files:**
- 2.1_password-reset-poisoning.md
- 2.2_cache-poisoning.md
- 2.3_routing-ssrf.md

---

### 📁 **Category 3: EXPLOITATION** (20% of applications vulnerable)

**Focus:** Complex exploitation and chaining

**Covers:**
- ✅ Connection state attacks
- ✅ Authentication bypass chains
- ✅ Multi-host exploitation
- ✅ Infrastructure manipulation
- ✅ Persistent access via poisoning

**Expected bounty:** $1,000-$20,000+ per finding

**Files:**
- 3.1_connection-state-attacks.md
- 3.2_authentication-bypass.md
- 3.3_multi-host-exploitation.md

---

## Quick Decision Tree

```
Found Host header usage?
│
├─ Try CATEGORY 1 (Basic Attacks) first
│  ├─ Test host reflection
│  ├─ Try override headers
│  ├─ Enumerate virtual hosts
│  └─ Found something? → Continue to exploitation
│
├─ Try CATEGORY 2 (Advanced Attacks) if password reset exists
│  ├─ Password reset poisoning
│  ├─ Cache poisoning
│  ├─ Routing SSRF
│  └─ Found vulnerability? → Report bounty!
│
└─ Try CATEGORY 3 (Exploitation) if infrastructure complex
   ├─ Connection state attacks
   ├─ Authentication bypass
   └─ Multi-host chaining
```

---

## Real-World Examples

### Example 1: Password Reset Poisoning → $3,000
```
1. Application generates password reset links using Host header
2. Attacker sends request with Host: attacker.com
3. Reset link in email: https://attacker.com/reset?token=xyz
4. Victim clicks link (attacker's domain)
5. Attacker captures token and resets password
Impact: Account takeover
Bounty: $3,000
```

### Example 2: Cache Poisoning → $5,000
```
1. CDN caches responses based on Host header
2. Attacker sends request with Host: attacker.com
3. Malicious content cached with attacker domain key
4. Innocent user gets cached malicious content
5. SSRF/XSS possible
Impact: Persistent attack
Bounty: $5,000+
```

### Example 3: Virtual Host Discovery → $1,500
```
1. Attacker fuzzes Host header values
2. Discovers admin.internal, staging.api, dev.backend
3. Each with different applications
4. Finds SQL injection in staging environment
Impact: Hidden application compromise
Bounty: $1,500+
```

### Example 4: Routing SSRF → $2,500
```
1. Host header controls backend routing
2. Attacker sends Host: localhost or 127.0.0.1
3. Server routes internally to admin panel
4. Bypasses authentication via internal routing
Impact: Admin access
Bounty: $2,500+
```

---

## Common Override Headers

Applications often check these headers too:

```
X-Forwarded-Host     ← Most common
X-Host               ← Some frameworks
Forwarded            ← RFC 7239
X-Original-Host      ← Legacy systems
X-Rewrite-URL        ← IIS servers
X-Forwarded-Server   ← Some proxies
X-HTTP-Host          ← Rare but exists
```

**Test strategy:** Try replacing Host with each of these

---

## Success Indicators

**Host header vulnerable if:**
- ✅ Host reflected in absolute URLs
- ✅ Host influences redirects/Location headers
- ✅ Override headers accepted
- ✅ Password reset links use attacker's domain
- ✅ Different virtual hosts accessible
- ✅ Response cached with attacker's host
- ✅ Internal routing possible

---

## Tools Needed

- **Burp Suite** (Repeater, Intruder)
- **curl** (testing Host headers)
- **ffuf** (virtual host discovery)
- **Python** (automation)

---

## 4-Phase Methodology

### Phase 1: Basic Testing
```
1. Send requests with different Host values
2. Observe if reflected in responses
3. Check for absolute URLs using Host header
4. Test override headers
```

### Phase 2: Discovery
```
1. Enumerate virtual hosts
2. Find hidden applications
3. Map internal services
4. Identify infrastructure
```

### Phase 3: Exploitation
```
1. Test password reset poisoning
2. Test cache poisoning
3. Test routing-based SSRF
4. Chain vulnerabilities
```

### Phase 4: Impact Verification
```
1. Prove account takeover possible
2. Demonstrate persistent cache poisoning
3. Show internal resource access
4. Document full exploitation chain
```

---

## Bounty Summary

- Basic vulnerabilities: $200-$2,000
- Password reset poisoning: $1,000-$5,000
- Cache poisoning: $2,000-$10,000
- SSRF via routing: $1,500-$5,000
- Authentication bypass: $2,000-$10,000+
- Complete chains: $5,000-$20,000+

**Average:** $1,500-$4,000 per vulnerability

---

## Critical Files

**START HERE:**
1. Read SKILL.md (this file)
2. Read methodology.md (4-phase approach)
3. Choose category based on findings

**For each category:**
1. Read category files
2. Use payloads.md for test cases
3. Follow checklist.md systematically
4. Document all findings

---

## Next Steps

1. **New to Host Header?** → Start with Category 1
2. **Found vulnerable app?** → Go to Category 2
3. **Complex infrastructure?** → Try Category 3
4. **Need payloads?** → Check payloads.md
5. **Testing systematically?** → Use checklist.md

---

**Start with Category 1 for foundational understanding, then escalate!** 🎯
