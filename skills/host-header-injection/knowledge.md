# Host Header Knowledge Base

## Core Concepts

### HTTP Host Header
- Identifies which domain request is for
- Set by client/browser
- Controls virtual host routing
- Often trusted by application

### Virtual Hosting
- Single server hosts multiple applications
- Routing based on Host header value
- Common in shared hosting, cloud

### Reverse Proxies/Load Balancers
- Sit between client and server
- May add override headers (X-Forwarded-Host)
- Use Host for routing decisions

### CDN/Caching
- Cache responses based on Host
- Poisoned cache affects all users
- Can lead to persistent attacks

---

## Common Vulnerabilities

### Password Reset Poisoning
**Risk:** Account takeover
**Impact:** High
**Effort:** Low

### Cache Poisoning
**Risk:** XSS, open redirect, persistent attack
**Impact:** High
**Effort:** Medium

### Routing SSRF
**Risk:** Internal access, metadata access
**Impact:** Medium-High
**Effort:** Low

### Virtual Host Discovery
**Risk:** Hidden apps, admin panels
**Impact:** Medium
**Effort:** Low

### Authentication Bypass
**Risk:** Unauthorized access
**Impact:** Critical
**Effort:** Medium

---

## Attack Vectors

### Direct Host Header
```
Host: attacker.com
```

### Override Headers
```
X-Forwarded-Host: attacker.com
Forwarded: host=attacker.com
X-Original-Host: attacker.com
```

### Port Confusion
```
Host: target.com:attacker.com
```

### Whitespace/Encoding
```
Host: attacker.com%20
Host: attacker.com\x0d\x0a
```

---

## High-Value Targets

1. **Password Reset** → Account takeover ($2,000+)
2. **Admin Panels** → Full access ($3,000+)
3. **Internal Services** → Infrastructure access ($2,000+)
4. **Cache Systems** → Persistent attacks ($2,000+)
5. **Cloud Metadata** → Credential theft ($5,000+)

---

## Hunting Approach

1. Always test Host header first
2. Test override headers second
3. Fuzz for virtual hosts
4. Look for password reset
5. Check for cache behavior
6. Attempt exploitation
7. Chain vulnerabilities

