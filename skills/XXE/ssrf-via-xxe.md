# SSRF via XXE

## Goal

Force backend requests.

---

## Basic Payload

<!ENTITY xxe SYSTEM "http://internal-service">

---

## Targets

localhost

127.0.0.1

Internal APIs

Cloud Metadata

Admin Interfaces

---

## Indicators

Response disclosure

Backend errors

Timing differences

OAST interactions