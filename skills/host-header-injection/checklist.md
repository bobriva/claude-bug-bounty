# Host Header Testing Checklist

## Phase 1: Discovery

### Reflection Testing
- [ ] Test Host: attacker.com
- [ ] Compare response with baseline
- [ ] Check absolute URLs
- [ ] Check Location headers
- [ ] Check error messages
- [ ] Check email content

### Documentation
- [ ] Document reflected locations
- [ ] Note response differences
- [ ] Screenshot key findings

---

## Phase 2: Override Headers

### Header Testing
- [ ] Test X-Forwarded-Host
- [ ] Test X-Host
- [ ] Test Forwarded
- [ ] Test X-Original-Host
- [ ] Test X-Rewrite-URL
- [ ] Test X-Forwarded-Server

### Combinations
- [ ] Test header combinations
- [ ] Test multiple headers together
- [ ] Test with different values

### Documentation
- [ ] Document working headers
- [ ] Note bypass techniques

---

## Phase 3: Enumeration

### Virtual Host Fuzzing
- [ ] Try admin.target.com
- [ ] Try dev.target.com
- [ ] Try staging.target.com
- [ ] Try internal.target.com
- [ ] Try api.target.com
- [ ] Try Host: admin (no domain)
- [ ] Try Host: internal
- [ ] Try localhost
- [ ] Try 127.0.0.1
- [ ] Try private IP ranges

### Subdomain Enumeration
- [ ] Use FFUF for fuzzing
- [ ] Use wordlist of common names
- [ ] Filter by status code changes
- [ ] Filter by content length changes
- [ ] Filter by unique titles

### Documentation
- [ ] List all discovered vhosts
- [ ] Note their purposes
- [ ] Check for vulnerabilities

---

## Phase 4: Exploitation

### Password Reset
- [ ] Identify reset functionality
- [ ] Test with Host: attacker.com
- [ ] Test with X-Forwarded-Host
- [ ] Capture reset emails
- [ ] Verify attacker domain in links
- [ ] Test token validity
- [ ] Attempt account takeover

### Cache Poisoning
- [ ] Identify cache (CDN/proxy)
- [ ] Test with Host: attacker.com
- [ ] Verify cache keys use Host
- [ ] Inject malicious content
- [ ] Verify poisoning
- [ ] Test with second user

### Routing SSRF
- [ ] Test Host: localhost
- [ ] Test Host: 127.0.0.1
- [ ] Test Host: internal
- [ ] Test Host: admin
- [ ] Check responses
- [ ] Attempt exploitation

### Authentication Bypass
- [ ] Map auth requirements
- [ ] Test with admin Host
- [ ] Test with internal Host
- [ ] Attempt unauthorized actions
- [ ] Document bypass chain

### Connection State
- [ ] Establish keep-alive connection
- [ ] Send valid request first
- [ ] Reuse connection
- [ ] Send attacker Host
- [ ] Check if accepted

---

## Reporting

- [ ] Document all findings
- [ ] Create PoC screenshots
- [ ] Write exploitation steps
- [ ] Explain impact
- [ ] Suggest remediation
- [ ] Estimate severity

