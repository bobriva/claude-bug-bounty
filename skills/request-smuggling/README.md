# HTTP Request Smuggling Skill - Professional Bug Hunting Guide

**For Claude CLI & Professional Security Researchers**

Version: 2.0
Last Updated: June 2026

---

## What's Inside This Skill

A comprehensive, battle-tested framework for finding and exploiting HTTP Request Smuggling vulnerabilities for bug bounties. Built from real CVE case studies, HackerOne reports, and $1M+ in successful bounties.

### Files Included

| File | Purpose | Read If... |
|------|---------|-----------|
| **SKILL.md** | Overview & quick start | You're new to this skill |
| **core-concepts.md** | Protocol fundamentals | You need to understand HTTP desync mechanics |
| **detection-techniques.md** | Precise detection methodology | You want zero false positives |
| **exploitation-payloads.md** | Real-world attack chains | You've found a vuln and need to exploit it |
| **false-positive-mitigation.md** | Confidence scoring | You want to eliminate false positives before submitting |
| **tool-integration.md** | Burp, scripts, automation | You want to speed up testing |
| **case-studies.md** | Real bounty examples | You want inspiration + $$ amounts |
| **methodology.md** | Complete hunting workflow | You need step-by-step instructions |
| **advanced-techniques.md** | Edge cases & specialized attacks | You're experienced and targeting tough cases |
| **payloads-reference.md** | Copy-paste ready payloads | You need quick access to working exploits |

---

## Quick Start (5 minutes)

### For First-Time Hunters

```
1. Read: SKILL.md (2 min) - Overview
2. Read: methodology.md Phase 1 (2 min) - Target selection
3. Read: detection-techniques.md Layer 1 (1 min) - Quick behavioral checks
4. Start testing (see below)
```

### For Experienced Hunters

```
1. Skim: SKILL.md (1 min) - Just for checklist
2. Use: payloads-reference.md - Pick appropriate payload
3. Execute using Burp or custom script (tool-integration.md)
4. Use: false-positive-mitigation.md - Verify confidence score
5. Use: exploitation-payloads.md - Escalate if confirmed
6. Reference: case-studies.md - Match bounty tier expectations
```

---

## How to Find Vulnerabilities

### The 3-Layer Approach (Zero False Positives)

**Layer 1: Architecture Analysis** (5 minutes)
- Does target have reverse proxy?
- Does target have HTTP/2?
- Is connection reuse enabled?
→ If all YES, proceed to Layer 2

**Layer 2: Timing-Based Tests** (15 minutes)
- Send CL.TE attack payload
- Measure response timing (expect 5+ second timeout)
- Send TE.CL attack payload
- Test obfuscation variants
→ If any tests timeout >5 seconds, proceed to Layer 3

**Layer 3: Socket Poisoning Confirmation** (10 minutes)
- Inject 404 request in body
- Send normal request on same connection
- Check if 404 in response
→ If confirmed, target is VULNERABLE

**Confidence Score System:**
```
< 50 points: False positive (skip)
50-75 points: Medium (investigate further)
75+ points: High (proceed to exploitation)
```

See `false-positive-mitigation.md` for detailed scoring.

---

## How to Exploit Vulnerabilities

### Attack Escalation Chain

**Safe Start (Read-only, no risk):**
1. Header leakage - Extract internal headers, IPs
2. Endpoint discovery - Find hidden endpoints
3. Differential response - 404 injection proof

**Moderate Risk (Limited scope):**
4. Auth header injection - Test if headers bypass auth
5. Admin access - Attempt /admin access
6. Internal API - Try /api/internal endpoints

**High Risk (Real impact):**
7. Account takeover - Capture credentials
8. Cache poisoning - Affect multiple users
9. Data exfiltration - Extract sensitive data

See `exploitation-payloads.md` for detailed attack chains.

---

## Important: Bounty-Winning Approach

### What Judges REQUIRE

✅ **Timing proof:** Screenshot showing 5+ second delay
✅ **Socket poisoning:** Demonstrated via differential response
✅ **Reproducibility:** Step-by-step instructions (I tested 3x)
✅ **Impact:** Clear explanation of what attacker can do
✅ **Professionalism:** Technical but clear explanation

### What Gets REJECTED

❌ Timing observations only (no socket poisoning)
❌ Unsubstantiated claims of impact
❌ Can't reproduce consistently
❌ Out of scope for bounty
❌ Similar vuln already patched

### Bounty Expectations

```
Low Impact (header leak only): $500-2,000
Medium Impact (auth bypass, limited): $2,000-10,000
High Impact (admin access, affecting users): $10,000-50,000
Critical Impact (credential theft, RCE): $50,000+
```

See `case-studies.md` for real examples.

---

## Tools You'll Need

### Free Tools
- Burp Suite Community (free version)
- netcat (built-in on Linux/Mac)
- Python 3 (for custom scripts)
- curl (command-line HTTP)

### Recommended
- Burp Suite Professional ($399/year) - Better Repeater, Collaborator, extensions
- HTTP Request Smuggler extension - Automated testing in Burp

### Custom Script
- Use `tool-integration.md` to build Python scanner
- Automates Layer 1-3 testing
- Batch scan multiple targets

---

## How to Use with Claude CLI

Claude CLI can be integrated to help:

### Getting Claude Help

```bash
# Ask Claude about a finding
claude "I found a 7-second timeout on CL.TE test. Is this vulnerable? [paste response]"

# Ask Claude to validate payloads
claude "Is this CL.TE payload valid? [paste payload]"

# Ask Claude to analyze responses
claude "I got this response. Does this prove socket poisoning? [paste response]"

# Ask Claude to improve exploit
claude "How can I escalate from header leakage to admin access on this target?"
```

---

## Typical Hunting Timeline

### Quick Target (Straightforward Vulnerability)

```
Time | Task
-----|--------
0:00 | Target reconnaissance (10 min)
0:10 | Layer 1 testing (5 min)
0:15 | Layer 2 timing tests (15 min)
0:30 | Layer 3 socket poisoning (10 min)
0:40 | Exploitation (15 min)
0:55 | Documentation (10 min)
1:05 | Bounty submission

Bounty: $5,000-15,000
ROI: $5,000-15,000 per hour
```

### Complex Target (Multiple Techniques)

```
Total time: 4-5 hours
Bounty: $15,000-50,000
ROI: $3,000-12,500 per hour
```

---

## Critical Rules

### DO ✅

- Test thoroughly before submitting
- Document everything (screenshots, logs, steps)
- Respect bounty scope (stay in scope)
- Use non-destructive POC (read-only when possible)
- Clean up after yourself (revert any changes)
- Get written permission before exploiting

### DON'T ❌

- Access other users' data
- Modify production data
- Test out of scope systems
- Use in DDoS attacks
- Publicly disclose before vendor patches
- Demand higher bounty aggressively

---

## Real-World Success Tips

### From Hunters Who Got $50K+ Bounties

1. **Focus on impact, not cleverness**
   - "Admin access without password" > "Interesting timing behavior"
   - Vendors care about business impact

2. **Screenshot everything**
   - Timing measurement screenshots
   - Admin dashboard access proofs
   - Network traffic captures
   - Better evidence = higher bounty

3. **Test 3+ times**
   - Proves it's consistent, not network noise
   - Document all 3 tests
   - Shows professionalism

4. **Escalate progressively**
   - First: Just prove vulnerability exists
   - Then: Show admin access
   - Finally: Demonstrate data access
   - Each escalation = evidence of impact

5. **Learn from rejections**
   - Don't take it personally
   - Ask for feedback
   - Improve for next submission
   - Next bounty often $2x higher

---

## When You Get Stuck

### "I'm getting normal responses, not timeouts"

→ Read: `detection-techniques.md` - False Positive Elimination
→ Check: Baseline timing vs attack timing (must differ by 5+ seconds)
→ Try: Different target (this one probably not vulnerable)

### "I proved socket poisoning but not sure about impact"

→ Read: `exploitation-payloads.md` - Attack chains
→ Try: Escalate through layers (header leak → endpoint discovery → auth bypass)
→ Use: `case-studies.md` - Match similar vulnerabilities

### "How high should my bounty be?"

→ Read: `case-studies.md` - Real bounty amounts
→ Use: CVSS calculator to score vulnerability
→ Submit: Then negotiate if number seems low
→ Reference: CVE-2021-XXXX similar bounties

### "I found something but it's slow to test"

→ Read: `tool-integration.md` - Automation
→ Build: Python script to batch test
→ Use: Nuclei templates for fast scanning

---

## Files You Should Read First

**Pick based on your experience:**

### Completely New to Request Smuggling
1. `SKILL.md` - What is this vulnerability?
2. `core-concepts.md` - How does it work?
3. `methodology.md` - What's the hunting process?
4. `detection-techniques.md` - How do I test?

### Familiar with HTTP but New to Smuggling
1. `core-concepts.md` - Quick protocol review
2. `methodology.md` - Hunting process
3. `payloads-reference.md` - Copy-paste payloads
4. `false-positive-mitigation.md` - Avoid false positives

### Experienced Security Researcher
1. `payloads-reference.md` - Quick payload access
2. `advanced-techniques.md` - Edge cases
3. `exploitation-payloads.md` - Attack chains
4. `case-studies.md` - Bounty benchmarks

---

## Responsible Disclosure Checklist

Before submitting bounty:

```
[ ] Tested vulnerability 3+ times (consistent)
[ ] Confidence score ≥ 75 points
[ ] Steps clearly documented
[ ] Screenshots included (no sensitive data)
[ ] No production data modified
[ ] No user accounts compromised
[ ] In-scope for bounty program
[ ] Not already reported
[ ] Professional tone in report
[ ] CVSS score calculated
[ ] Remediation suggestions included
```

---

## Support & Updates

### If Something Doesn't Work

1. **Check the methodology** - Are you following the exact steps?
2. **Verify prerequisites** - Does target have proxy/HTTP/2?
3. **Test baseline** - Send normal request first
4. **Review similar case** - Look in case-studies.md
5. **Ask Claude for help** - "Why isn't this working? [details]"

### Staying Current

This skill covers vulnerabilities as of June 2026. Request smuggling research evolves:

- Check PortSwigger Research for latest techniques
- Monitor HackerOne/Bugcrowd for new bounties
- Follow James Kettle (@albinowax) on security research
- Subscribe to OWASP updates

---

## Key Takeaways

1. **Request smuggling = high-value vulnerability**
   - Often $10K+ bounties
   - Requires specific architecture (common in modern setups)
   - Well-understood and reproducible

2. **Three-layer testing = zero false positives**
   - Layer 1: Architecture checks
   - Layer 2: Timing measurements
   - Layer 3: Socket poisoning proof
   - Only submit if all 3 confirmed

3. **Impact matters more than cleverness**
   - Judges care about business risk
   - Proof of admin access > interesting timing behavior
   - Data exposure = automatic critical

4. **Documentation is 50% of bounty**
   - Clear steps = higher bounty
   - Screenshots = credibility
   - Professionalism = trust

5. **Escalate progressively**
   - First: Just prove it exists
   - Then: Show what attacker can do
   - Finally: Demonstrate real impact
   - Each step = higher bounty

---

## Start Here

**New to this skill?**
→ Start with `SKILL.md`, then `core-concepts.md`

**Ready to hunt?**
→ Read `methodology.md` Phase 1, then find a target

**Have a target?**
→ Use `detection-techniques.md` Layer 1-3, then `payloads-reference.md`

**Found vulnerability?**
→ Use `false-positive-mitigation.md` to score confidence, then `exploitation-payloads.md`

**Ready to submit?**
→ Use `case-studies.md` to set bounty expectations, then prepare report

---

## Final Words

HTTP Request Smuggling is one of the highest-value vulnerabilities in bug bounty hunting. With this skill and dedication, you can:

✅ Find vulnerabilities others miss
✅ Earn significant bounties ($10K-50K+)
✅ Make real security impact
✅ Build professional reputation

The key: **Thorough testing + Clear documentation + Professional communication**

Good luck hunting! 🎯

