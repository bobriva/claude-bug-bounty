# Connection State Attacks

## Concept

Frontend trusts first request.

Subsequent requests inherit routing.

---

## Targets

Load Balancers

Reverse Proxies

CDNs

---

## Indicators

Different Host accepted on reused connection.

Unexpected routing.

Internal host access.

---

## Impact

Authentication bypass.

Host header bypass.

Internal application access.