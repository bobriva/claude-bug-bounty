# HTTP Request Smuggling Skill - Complete Index

Navigate this skill by topic, experience level, or task.

---

## By File (Alphabetical)

### advanced-techniques.md
- Browser-powered desync (CL.0)
- Request tunnelling
- Pause-based desync
- HTTP/2 specific attacks
- Unicode & encoding attacks
- Whitespace manipulation
- Response queue poisoning
- Exploit chaining
- Blind exploitation

**Read when:** You've mastered basics and need specialized knowledge
**Length:** ~3000 words
**Time to read:** 15-20 minutes

---

### case-studies.md
- 5 real bounty case studies with bounty amounts
- Real CVE references
- Bounty scaling factors
- What judges look for
- Top 10 highest bounties
- Lessons from successful reports
- High-bounty report template

**Read when:** You want to understand what really works + expected bounty amounts
**Length:** ~3000 words
**Time to read:** 15 minutes

---

### core-concepts.md
- HTTP/1.1 protocol fundamentals
- Content-Length vs Transfer-Encoding
- HTTP/2 differences
- CL.TE desynchronization
- TE.CL desynchronization
- TE.TE obfuscation
- H2.CL attacks
- H2.TE attacks
- CL.0 attacks
- Socket poisoning
- Why this matters for bounties

**Read when:** You don't understand HTTP desync mechanics
**Length:** ~2500 words
**Time to read:** 12-15 minutes

---

### detection-techniques.md
- 3-layer detection strategy
- Layer 1: Behavioral filters
- Layer 2: Timing-based detection (with timing thresholds)
- Layer 3: Differential response confirmation
- HTTP/2 detection
- Automated detection workflow
- False positive elimination
- Confidence scoring matrix
- Copy-paste ready payloads

**Read when:** You need to detect vulnerabilities with zero false positives
**Length:** ~3500 words
**Time to read:** 18-20 minutes

---

### exploitation-payloads.md
- Attack 1: Header leakage
- Attack 2: Cache poisoning
- Attack 3: Access control bypass
- Attack 4: Credential theft
- Attack 5: Auth mechanism bypass
- Attack 6: Admin panel access
- Real-world bounty examples
- Safe testing approach
- Payload generation checklist

**Read when:** You've confirmed vulnerability and need to demonstrate impact
**Length:** ~3000 words
**Time to read:** 15-18 minutes

---

### false-positive-mitigation.md
- Confidence scoring system (0-100 points)
- Layer 1-3 scoring breakdown
- Common false positives & elimination
- Pre-submission validation checklist
- Red flags (when NOT to submit)
- Documentation for bounty report
- Technical details formatting

**Read when:** You want to avoid submitting false positives
**Length:** ~4000 words
**Time to read:** 20-25 minutes

---

### methodology.md
- Phase 1: Target selection (15 minutes)
- Phase 2: Detection testing (30 minutes)
- Phase 3: Variant identification (15 minutes)
- Phase 4: Exploitation planning (30 minutes)
- Phase 5: Proof generation (60 minutes)
- Phase 6: Responsible disclosure (timeline)
- Phase 7: Bounty negotiation
- Complete workflow diagram
- Time estimates

**Read when:** You need step-by-step instructions from start to bounty
**Length:** ~4000 words
**Time to read:** 20-25 minutes

---

### payloads-reference.md
- Detection payloads (CL.TE, TE.CL, TE.TE variants, CL.0, H2.CL)
- Differential response payloads
- Exploitation payloads
- Testing payloads
- Tools for sending payloads
- Payload template generator
- Success indicators
- Common errors & fixes

**Read when:** You need quick access to working exploits
**Length:** ~2500 words
**Time to read:** 5-10 minutes (reference only)

---

### README.md
- Skill overview
- What's inside
- Quick start guides (different experience levels)
- 3-layer approach explanation
- Tools needed
- Timeline expectations
- Rules (do's and don'ts)
- Success tips
- Navigation by task

**Read when:** You're just starting or need orientation
**Length:** ~3000 words
**Time to read:** 10-15 minutes

---

### SKILL.md
- Main skill definition
- When to use this skill
- Impact assessment
- Quick start checklist
- File structure overview

**Read when:** Claude CLI prompts you to read the skill
**Length:** ~500 words
**Time to read:** 2-3 minutes

---

### tool-integration.md
- Tool landscape & recommendations
- Burp Suite Repeater workflow
- HTTP Request Smuggler extension
- Custom Python script (full code)
- Nuclei templates
- Batch scanning script
- Jenkins CI/CD integration
- Tool comparison matrix

**Read when:** You want to automate testing or use specific tools
**Length:** ~2500 words
**Time to read:** 12-15 minutes

---

## By Experience Level

### Complete Beginner (No security experience)
1. Read: `README.md` (orientation, 10 min)
2. Read: `SKILL.md` (overview, 2 min)
3. Read: `core-concepts.md` (fundamentals, 15 min)
4. Read: `methodology.md` Phase 1 only (target selection, 5 min)
5. Try: Pick a target from your bug bounty program
6. Read: `detection-techniques.md` (how to test, 20 min)
7. Try: Execute Layer 1 tests from your target
8. Read: `false-positive-mitigation.md` (avoid false positives, 20 min)
9. Read: `payloads-reference.md` (copy payload, 5 min)
10. Try: Execute Layer 2 timing tests

**Total time before first test:** 1.5-2 hours

---

### Experienced Developer (Knows HTTP, new to sec)
1. Skim: `core-concepts.md` (review, 5 min)
2. Read: `methodology.md` (full process, 20 min)
3. Read: `detection-techniques.md` (detection, 20 min)
4. Use: `payloads-reference.md` (execute tests)
5. Read: `exploitation-payloads.md` (if confirmed vulnerable)
6. Read: `false-positive-mitigation.md` (validate findings)
7. Submit: Use `case-studies.md` as bounty reference

**Total time before first test:** 1 hour

---

### Security Professional (Bug bounty experience)
1. Skim: `SKILL.md` (2 min)
2. Reference: `payloads-reference.md` (run tests)
3. If vulnerable: `exploitation-payloads.md` (escalate)
4. Reference: `false-positive-mitigation.md` (validate, 5 min)
5. Reference: `case-studies.md` (bounty expectations)
6. Use: `tool-integration.md` (automation if batch testing)

**Total time before first test:** 15-20 minutes

---

### Expert Bug Hunter (Multiple CVEs found)
1. skim: Everything (orientation, 10 min)
2. Deep dive: `advanced-techniques.md` (edge cases, 15 min)
3. Execute: `payloads-reference.md`
4. Reference: `exploitation-payloads.md` (attack chains)
5. Focus: Find the $50K+ bounty opportunities (real impact)

**Total time before first test:** 25 minutes

---

## By Task

### "I want to find request smuggling vulnerabilities"

**Start here:**
1. `SKILL.md` - What is it?
2. `core-concepts.md` - How does it work?
3. `methodology.md` - Step-by-step process
4. `detection-techniques.md` - How to test

**Then:**
- Use `payloads-reference.md` with your target

---

### "I've found a vulnerability, now what?"

**Start here:**
1. `false-positive-mitigation.md` - Is it real?
2. `exploitation-payloads.md` - How to demonstrate impact
3. `case-studies.md` - What bounty to expect

**Then:**
- Document findings
- Submit bounty report

---

### "I'm getting lots of timeouts but not sure if real"

**Read:**
1. `detection-techniques.md` - False Positive Elimination section
2. `false-positive-mitigation.md` - Confidence scoring (must be 75+)
3. Make sure: Layer 3 (socket poisoning) confirmed

---

### "How do I prove vulnerability to vendor?"

**Read:**
1. `exploitation-payloads.md` - Real impact demonstration
2. `case-studies.md` - Report template
3. `methodology.md` Phase 6 - Responsible disclosure

---

### "What bounty should I ask for?"

**Read:**
1. `case-studies.md` - Real bounty examples and amounts
2. `exploitation-payloads.md` - Impact levels
3. Match your finding to similar vulnerabilities

---

### "I want to automate testing"

**Read:**
1. `tool-integration.md` - Tools and scripts
2. Look for: Python script section
3. Look for: Nuclei template section

---

### "I want to understand HTTP/2 attacks"

**Read:**
1. `core-concepts.md` - HTTP/2 section
2. `advanced-techniques.md` - HTTP/2 specific attacks
3. `payloads-reference.md` - H2.CL payload

---

## By Length (Reading Time)

### 5-10 minutes
- `SKILL.md` (2-3 min)
- `payloads-reference.md` (5-10 min if just referencing)

### 10-20 minutes
- `README.md` (10-15 min)
- `case-studies.md` (15 min)
- `tool-integration.md` (12-15 min)

### 20-30 minutes
- `core-concepts.md` (12-15 min)
- `detection-techniques.md` (18-20 min)
- `methodology.md` (20-25 min)

### 30+ minutes
- `exploitation-payloads.md` (15-18 min)
- `false-positive-mitigation.md` (20-25 min)
- `advanced-techniques.md` (15-20 min)

**Total skill reading time:** 3-4 hours for complete mastery
**Minimum to get started:** 1.5 hours

---

## Search by Topic

### Burp Suite
- `tool-integration.md` - Burp Repeater workflow & HTTP Request Smuggler

### Cache Poisoning
- `exploitation-payloads.md` - Attack 2
- `case-studies.md` - Case Study 2

### CL.0 (Browser Desync)
- `core-concepts.md` - CL.0 section
- `advanced-techniques.md` - Browser-Powered Desync
- `payloads-reference.md` - CL.0 payload

### Confidence Scoring
- `false-positive-mitigation.md` - Complete scoring system (0-100 points)

### Detection (How to test)
- `detection-techniques.md` - Complete 3-layer methodology

### Differential Response
- `detection-techniques.md` - Layer 3
- `payloads-reference.md` - Differential response payloads

### False Positives (Avoiding them)
- `false-positive-mitigation.md` - Elimination techniques & common false positives
- `detection-techniques.md` - Timing thresholds & verification

### HTTP/2 (H2.CL, H2.TE)
- `core-concepts.md` - HTTP/2 Differences & H2.CL section
- `advanced-techniques.md` - HTTP/2 Specific Attacks
- `detection-techniques.md` - HTTP/2 Detection section
- `payloads-reference.md` - H2.CL payload

### Methodology (Step-by-step)
- `methodology.md` - Phases 1-7, complete workflow

### Nuclei & Automation
- `tool-integration.md` - Nuclei template

### Payloads (Copy-paste ready)
- `payloads-reference.md` - All payloads in one place

### Python Script
- `tool-integration.md` - Custom Python scanner (full code)

### Real Bounties (Examples with amounts)
- `case-studies.md` - 5 case studies + top 10

### Response Queue Poisoning
- `core-concepts.md` - Socket Poisoning section
- `exploitation-payloads.md` - Attack 4
- `advanced-techniques.md` - Response Queue Poisoning

### TE.CL (Transfer-Encoding vs Content-Length)
- `core-concepts.md` - TE.CL section
- `detection-techniques.md` - TE.CL detection
- `payloads-reference.md` - TE.CL payloads

### TE.TE (Obfuscation)
- `core-concepts.md` - TE.TE section
- `detection-techniques.md` - TE.TE detection
- `payloads-reference.md` - TE.TE variants

### Timing Attacks
- `detection-techniques.md` - Layer 2: Timing-Based Detection
- `false-positive-mitigation.md` - False Positive 1: Network Latency Spike

### Tools & Integration
- `tool-integration.md` - Complete tools section

### WAF Bypass
- `exploitation-payloads.md` - Technique 1: Header Injection
- `tool-integration.md` - Custom Python script for WAF bypass

---

## Quick Decision Tree

```
START: "I want to find HTTP Request Smuggling bugs"
  │
  ├─ YES: "Do I understand HTTP?"
  │   ├─ NO → Read: core-concepts.md
  │   └─ YES → Continue
  │
  ├─ "Do I have a target yet?"
  │   ├─ NO → Read: methodology.md Phase 1
  │   └─ YES → Continue
  │
  ├─ "Do I know how to test?"
  │   ├─ NO → Read: detection-techniques.md
  │   └─ YES → Continue
  │
  ├─ "Ready to test?"
  │   └─ YES → Use: payloads-reference.md
  │
  ├─ "Confirmed vulnerable?"
  │   ├─ NO → Check: false-positive-mitigation.md
  │   └─ YES → Continue
  │
  ├─ "Need to prove impact?"
  │   └─ YES → Read: exploitation-payloads.md
  │
  ├─ "Ready to submit?"
  │   └─ YES → Use: case-studies.md for bounty expectations
  │
  └─ SUBMIT & GET PAID! 🎯
```

---

## For Claude CLI Users

When using Claude CLI to reference this skill:

**Ask Claude for help:**
```
claude "I found a 7-second timeout on CL.TE test. Is this vulnerable?"
→ Claude will reference detection-techniques.md automatically

claude "How do I prove this vulnerability?"
→ Claude will reference exploitation-payloads.md automatically

claude "What bounty should I expect?"
→ Claude will reference case-studies.md automatically
```

**In the background:**
- Claude reads all files in this skill
- Claude provides context-aware answers
- Claude references specific sections
- Claude provides working payloads

---

## File Dependencies

Files you need to read in order for true understanding:

```
README.md
    ↓
SKILL.md
    ↓
core-concepts.md (foundation)
    ├── detection-techniques.md (how to find)
    │   └── false-positive-mitigation.md (validate)
    │       └── exploitation-payloads.md (escalate)
    │           └── case-studies.md (bounty)
    │
    └── methodology.md (complete process)
        └── All above ^ combined into workflow
```

**Standalone files** (can read anytime):
- `payloads-reference.md` (quick lookup)
- `tool-integration.md` (tool-specific)
- `advanced-techniques.md` (specialized)

---

## Total Coverage

This skill covers:
- ✅ 5 common variants (CL.TE, TE.CL, TE.TE, H2.CL, H2.TE)
- ✅ Browser-powered desync (CL.0)
- ✅ Request tunnelling
- ✅ Response queue poisoning
- ✅ Cache poisoning
- ✅ Authentication bypass
- ✅ Admin access exploitation
- ✅ Credential theft
- ✅ Header injection
- ✅ Zero false positive methodology
- ✅ 3-layer detection system
- ✅ Automated testing scripts
- ✅ Burp Suite integration
- ✅ Real bounty examples
- ✅ Complete hunting workflow
- ✅ Responsible disclosure guidelines

**NOT covered** (out of scope):
- ❌ Network layer attacks
- ❌ TLS/SSL vulnerabilities
- ❌ DNS poisoning
- ❌ BGP hijacking
- ❌ Wireless attacks

---

## Getting Most Value

### Minimum (1.5 hours, find basic vulns)
1. `README.md` quick start
2. `detection-techniques.md` Layer 1-2
3. `payloads-reference.md` execution
4. Submit finding

### Standard (3 hours, professional approach)
1. Complete `methodology.md`
2. Complete `detection-techniques.md`
3. Complete `exploitation-payloads.md`
4. Complete `false-positive-mitigation.md`
5. Submit professional report

### Expert (5 hours, maximize bounties)
1. All above +
2. `advanced-techniques.md` for edge cases
3. `case-studies.md` for bounty positioning
4. `tool-integration.md` for automation
5. Submit high-impact findings + negotiate bounties

---

## Questions?

If you can't find what you need:

1. **Use search** - Find by topic above
2. **Check decision tree** - Find by your situation
3. **Use table of contents** - Find by file
4. **Ask Claude** - "I need help with [specific task]"

Good luck hunting! 🎯

