# Web Cache Deception Methodology

## Phase 1 - Endpoint Discovery

Identify dynamic endpoints containing:

- User profile
- Orders
- Invoices
- Messages
- Notifications
- Documents
- Account settings
- Dashboard data

Focus on:

GET
HEAD
OPTIONS

Avoid:

POST
PUT
PATCH
DELETE

unless behavior indicates caching.

---

## Phase 2 - Cache Detection

Check headers:

X-Cache
CF-Cache-Status
Age
Cache-Control
Via

Indicators:

Hit
Miss
Dynamic
Refresh

Compare response times.

---

## Phase 3 - Cache Busting

Use unique query parameters:

?cb=123
?cachebuster=random

Ensure every request generates a unique cache key.

---

## Phase 4 - Discrepancy Testing

Test:

- Path Mapping
- Delimiters
- Encoded Delimiters
- Normalization
- Static Directories
- Exact Match Rules

---

## Phase 5 - Exploitation

Confirm:

Victim Request
↓
Response Cached
↓
Unauthenticated Request
↓
Cached Sensitive Data Retrieved
