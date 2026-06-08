# Host Header Testing Methodology

## 4-Phase Approach

### Phase 1: Discovery (30 minutes)

**Goal:** Identify if application trusts Host header

**Steps:**
1. Send baseline requests to application
2. Note response format
3. Send request with different Host (attacker.com)
4. Compare responses

**Success indicators:**
- Absolute URLs change
- Location headers affected
- Content differs
- Host reflected somewhere

**Tools:** curl, Burp Repeater

---

### Phase 2: Override Header Testing (20 minutes)

**Goal:** Test if override headers bypass Host validation

**Steps:**
1. Try X-Forwarded-Host
2. Try X-Host
3. Try Forwarded
4. Try X-Original-Host
5. Try X-Rewrite-URL

**Success indicators:**
- Override header reflected
- Different responses
- Application trusts override

**Tools:** curl, Burp Intruder

---

### Phase 3: Enumeration (45 minutes)

**Goal:** Discover virtual hosts and hidden applications

**Steps:**
1. Manual fuzzing with common names
2. FFUF with wordlist
3. Test localhost, 127.0.0.1
4. Test internal, admin, dev, staging
5. Document all discoveries

**Success indicators:**
- Different status codes
- Different content
- New applications found
- Unique titles/headers

**Tools:** ffuf, Burp Intruder, curl

---

### Phase 4: Exploitation (60+ minutes)

**Goal:** Exploit discovered vulnerabilities

**Steps:**
1. Test password reset poisoning
2. Test cache poisoning
3. Test routing SSRF
4. Chain vulnerabilities
5. Demonstrate impact

**Success indicators:**
- Account takeover
- Unauthorized access
- Data exfiltration
- Cache compromise

**Tools:** Burp Repeater, custom scripts

---

## Timeline

```
30 min - Phase 1 (Discovery)
20 min - Phase 2 (Override Headers)
45 min - Phase 3 (Enumeration)
60 min - Phase 4 (Exploitation)

Total: ~3 hours per target
```

---

## Decision Points

### After Phase 1
- **If Host reflected:** Continue to Phase 2
- **If not reflected:** Try Phase 3 (virtual hosts)
- **If nothing:** Try override headers (Phase 2)

### After Phase 2
- **If override works:** Test password reset (Category 2)
- **If not:** Continue to Phase 3 (enumeration)

### After Phase 3
- **If virtual hosts found:** Exploit each one
- **If internal services:** Try routing SSRF (Category 2)
- **If nothing:** Try fuzzing for errors (Category 3)

### During Phase 4
- **If one vulnerability found:** Report as-is
- **If multiple vulnerabilities:** Chain them for higher impact
- **If can chain into RCE:** Maximum bounty

---

## Key Mindset

1. **Assume trust:** Start assuming application trusts Host
2. **Test systematically:** Try every vector
3. **Document findings:** Note all discoveries
4. **Chain when possible:** Multiple vulns = higher bounty
5. **Prove impact:** Demonstrate real damage

