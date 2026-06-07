---
name: web-cache-deception
description: Detect and exploit Web Cache Deception vulnerabilities caused by discrepancies between cache servers and origin servers.
---

# Web Cache Deception Skill

Use this skill when:

- Testing CDN behavior
- Analyzing cached responses
- Reviewing static resource rules
- Looking for sensitive GET endpoints
- Testing cache/origin parsing differences

## Objectives

1. Identify sensitive dynamic endpoints
2. Detect cacheable responses
3. Find parsing discrepancies
4. Verify cache storage of sensitive content
5. Produce reproducible PoC

## Testing Priority

1. Static extension rules
2. Delimiter discrepancies
3. Encoded delimiter discrepancies
4. Static directory rules
5. Path normalization discrepancies
6. Exact filename cache rules

## Required Validation

Never report:

- Cached public content
- Browser cache artifacts
- One-time responses
- Unverified cache behavior

Always verify:

- Sensitive data exposure
- Cross-user impact
- Cache persistence
- Reproducibility
- Attacker retrieval capability

Refer to:
- methodology.md
- detection.md
- exploitation.md
