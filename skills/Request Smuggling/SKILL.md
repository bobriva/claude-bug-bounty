---
name: request-smuggling
description: Detect and exploit HTTP Request Smuggling including CL.TE, TE.CL, HTTP/2 desync, browser-powered desync, request tunnelling and response queue poisoning.
---

# HTTP Request Smuggling

Use this skill when:

- Reverse proxy exists
- CDN exists
- Load balancer exists
- HTTP/2 exists
- Multiple backend systems exist

Objectives:

1. Detect desynchronization
2. Confirm socket poisoning
3. Explore attack surface
4. Build exploit chains

Read:

- methodology.md
- detection.md
- exploitation.md