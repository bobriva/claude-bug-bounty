---
name: information-disclosure
description: Systematic methodology for discovering and exploiting information disclosure vulnerabilities to chain into higher severity attacks (IDOR, SSRF, RCE, auth bypass).
---

# Information Disclosure Skill

## When to Use This Skill

Use when testing:
- Web applications and APIs
- Cloud deployments (AWS, GCP, Azure)
- Git repositories exposed
- Debug endpoints accessible
- Error pages revealing information
- Source code backup files
- Environment variables exposed
- API documentation
- GraphQL introspection
- JavaScript files with secrets

---

## Why Information Disclosure Matters

**Critical mindset:** Information disclosure is NOT just a "low-impact" vulnerability!

**Disclosed information can chain into:**
- ✅ IDOR (user ID enumeration)
- ✅ SSRF (internal service discovery)
- ✅ RCE (command injection with known paths)
- ✅ Authentication Bypass (leaked secrets/keys)
- ✅ Privilege Escalation (role enumeration)
- ✅ Business Logic Abuse (workflow manipulation)

**Real bounty examples:**
- AWS metadata access → EC2 takeover ($15,000+)
- Git exposure → Source code + secrets ($8,000-$20,000)
- Debug endpoints → Database credentials ($10,000+)
- Error stack traces → File paths → RCE ($5,000-$15,000)

---

## The 3 Main Categories

### 📁 Category 1: BASIC RECON (60% of cases)

**Focus:** File discovery and source code exposure

**Covers:**
- ✅ robots.txt, sitemap.xml enumeration
- ✅ Backup file discovery (.bak, .old, .zip, .orig)
- ✅ .git folder exposure & dumping
- ✅ Source code disclosure (PHP~, ASP.bak)
- ✅ Error page analysis & stack traces

**Expected bounty:** $200-$2,000 per finding

**Files:**
- 1.1_web-file-discovery.md
- 1.2_source-code-exposure.md
- 1.3_error-analysis.md

---

### 📁 Category 2: DEBUG ENDPOINTS (30% of cases)

**Focus:** Exposed debugging & configuration endpoints

**Covers:**
- ✅ Spring Boot Actuator endpoints
- ✅ PHP/ASP debug pages
- ✅ Environment variable exposure
- ✅ Cloud metadata endpoints
- ✅ Health/status endpoints

**Expected bounty:** $500-$5,000 per finding

**Files:**
- 2.1_debug-endpoints.md
- 2.2_cloud-metadata.md
- 2.3_env-vars-exposure.md

---

### 📁 Category 3: ADVANCED EXPLOITATION (10% of cases)

**Focus:** Fuzzing for leakage & chaining to higher severity

**Covers:**
- ✅ Fuzzing to trigger error messages
- ✅ JavaScript mining for secrets
- ✅ Chaining disclosure to RCE/IDOR/SSRF
- ✅ AWS/GCP metadata exploitation
- ✅ GraphQL schema introspection

**Expected bounty:** $1,000-$10,000+ per finding

**Files:**
- 3.1_fuzzing-info-leakage.md
- 3.2_javascript-mining.md
- 3.3_chaining-to-rce.md
- 3.4_metadata-abuse.md

---

## Quick Decision Tree

```
Target discovered?
│
├─ Check basic files first (Category 1)
│  ├─ robots.txt, sitemap.xml
│  ├─ Backup files (.bak, .old, .zip)
│  ├─ .git folder
│  └─ Found info? → Analyze & chain
│
├─ Try debug endpoints (Category 2)
│  ├─ /actuator, /phpinfo, /status
│  ├─ /debug, /env, /metrics
│  └─ Cloud metadata endpoints
│
└─ Fuzz for errors (Category 3)
   ├─ Invalid input fuzzing
   ├─ Trigger stack traces
   ├─ Mine JavaScript
   └─ Chain to higher severity
```

---

## Information Categories & Impact

| Info Type | Example | Severity | Bounty |
|-----------|---------|----------|--------|
| File paths | /var/www/html/ → RCE | Medium | $1K-$5K |
| DB creds | user:pass@localhost | High | $2K-$10K |
| API keys | AWS_ACCESS_KEY | Critical | $5K-$25K+ |
| Internal IPs | 10.0.0.1 → SSRF | Medium | $1K-$3K |
| Source code | Secret keys, logic | High | $2K-$8K |
| Cloud metadata | IAM roles | Critical | $5K-$20K+ |
| Framework version | Laravel 5.2 (vuln) | Medium | $1K-$5K |

---

## 5-Phase Methodology

### Phase 1: Discovery
Enumerate information sources:
- Web files, headers, metadata
- Error messages, source code
- API responses, comments

### Phase 2: Collection
Gather systematically:
- Use tools (Burp, ffuf, Wayback)
- Check all endpoints
- Parse responses

### Phase 3: Analysis
Analyze disclosures:
- What secrets? (API keys, passwords)
- What infrastructure? (IPs, paths, services)
- What code? (Logic flaws, hardcoded values)
- What's the impact?

### Phase 4: Chaining
Convert to higher severity:
- Can I access internal systems? (SSRF)
- Can I impersonate users? (IDOR)
- Can I execute code? (RCE)
- Can I escalate privileges?

### Phase 5: Exploitation & Reporting
- Execute the attack
- Prove impact with PoC
- Document step-by-step
- Report with findings

---

## Tools You'll Need

**Essential:**
- Burp Suite (+ Intruder)
- ffuf (fast fuzzing)
- curl/wget (fetch files)
- git-dumper (dump .git)

**Nice to have:**
- Wayback Machine
- Param Miner
- SecLists
- grep/jq

---

## Real-World Case Studies

### Case 1: .git Exposure → $20,000
```
Discovery: /.git/config exposed
Impact: Retrieved all commits + secrets
Bounty: $20,000
```

### Case 2: /actuator → $8,000
```
Discovery: Spring Boot actuator endpoints
Impact: Accessed environment variables with DB creds
Bounty: $8,000
```

### Case 3: AWS Metadata → $15,000
```
Discovery: SSRF to AWS metadata endpoint
Impact: Accessed EC2 role credentials
Bounty: $15,000
```

---

## Success Indicators

**Information disclosed if:**
- ✅ Sensitive data accessible without auth
- ✅ File/folder listing available
- ✅ Error messages revealing details
- ✅ Source code or configs exposed
- ✅ API keys/credentials visible
- ✅ Internal IPs/hostnames disclosed

---

## Bounty Summary

- Basic recon: $200-$2,000
- Debug endpoints: $500-$5,000
- Advanced exploitation: $1,000-$10,000+

**Average:** $1,000-$3,000 per vulnerability

---

## Start with Category 1 for foundational techniques!
