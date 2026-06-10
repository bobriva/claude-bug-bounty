---
name: race-condition
description: >
  Expert race condition testing for bug bounty. Trigger this skill whenever a target
  has ANY of these signals: coupon/discount/promo codes, gift cards, wallet/credits,
  password reset flows, email verification, MFA/2FA, invitation systems, subscription
  paywalls, referral rewards, vote/like limits, API rate limits, one-time tokens,
  checkout/payment flows, limited-quantity resources, or any multi-step workflow
  that touches shared state. Also trigger for microservices and distributed
  architectures — race windows are wider and often easier to exploit there.
  The mindset: stop thinking in endpoints, start thinking in STATE TRANSITIONS.
  Every gap between a security check and the action it protects is a potential
  race window.
---

# Race Condition Testing

## Quick Orientation

Read this file first. Follow the decision tree to load the right reference.

**Reference files:**
- `references/detection.md` — Identifying race windows, benchmarking, session locking
- `references/techniques.md` — Single-packet attack, last-byte sync, connection warming, Turbo Intruder
- `references/attacks.md` — All attack patterns: limit overrun, single/multi-endpoint, partial construction, TOCTOU
- `references/payloads.md` — Turbo Intruder scripts, Burp group configs, curl parallel commands

---

## The Core Mental Model

Every TOCTOU (Time-of-Check Time-of-Use) pattern is a race condition candidate:

```
NORMAL FLOW:
  [CHECK: "is coupon used?"] → [DECISION: allow/deny] → [USE: apply discount] → [UPDATE: mark used]

RACE WINDOW:
  Thread 1: CHECK → allowed ──────────────────────────────────── USE → UPDATE
  Thread 2:          CHECK → allowed (DB not updated yet!) → USE → UPDATE
                              ↑ THIS IS THE RACE WINDOW ↑
```

The goal: land **at least two requests inside the race window** before the state update completes.

---

## Phase 1: Reconnaissance — Find Race Windows

Before sending a single attack request, map the state machine.

### Ask These Questions for Every Feature:

```
1. Does this action modify shared state? (DB, cache, file, session)
2. Is there a check BEFORE the action? (coupon validation, limit check, auth check)
3. Is the check and the action in the same atomic DB transaction?
   → If NOT → race window exists
4. Are there multiple steps in the workflow?
   → Each step boundary is a potential race window
5. Is this a distributed/microservice architecture?
   → Cross-service calls = wider race windows
```

### High-Value Race Window Locations:

```
Financial:
- Coupon / discount code redemption
- Gift card balance check → deduction
- Wallet balance check → transfer/withdraw
- Refund processing → account credit
- Referral bonus → credit application
- Payment processing → order creation

Authentication:
- OTP/2FA verification → session creation
- Password reset token → validation → invalidation
- Email verification token → account activation
- Magic link → session issuance
- Account recovery → identity confirmation

Account Management:
- Email change → verification email send
- Email verify → old email token + new email token
- Invitation accept → membership creation
- Role/permission assignment → access grant
- Subscription check → feature access

Resources:
- Limited-seat registration (events, beta signups)
- One-vote-per-user systems
- Rate-limited API calls
- Daily/weekly usage limits
- Upload quota checks
```

---

## Phase 2: Decision Tree

```
Race window identified
│
├── Single limit being bypassed? (coupon, gift card, API limit)
│   └── Load: references/attacks.md → Limit Overrun section
│       Tool: Burp Repeater group (send in parallel)
│
├── Single endpoint, same action racing against itself?
│   (email verify + email change, reset token + use)
│   └── Load: references/attacks.md → Single Endpoint section
│       Tool: Turbo Intruder single-packet
│
├── Two DIFFERENT endpoints sharing state?
│   (payment + cart update, email change + verify)
│   └── Load: references/attacks.md → Multi-Endpoint section
│       Tool: Turbo Intruder gated multi-endpoint
│
├── Object created in multiple steps?
│   (registration flow, API key generation)
│   └── Load: references/attacks.md → Partial Construction section
│
├── Session locking causing serialization? (PHP sessions)
│   └── Load: references/detection.md → Session Locking section
│       Fix: use multiple session tokens
│
└── Timing inconsistency observed but not exploitable yet?
    └── Load: references/techniques.md → Connection Warming + Timing
```

---

## Phase 3: Choose Your Weapon

| Scenario | Best Tool | Why |
|---|---|---|
| Same endpoint, many parallel requests | Burp Repeater → Send group in parallel | Fastest setup |
| Need precise sync (HTTP/1.1 target) | Turbo Intruder + last-byte sync | Handles network jitter |
| Need precise sync (HTTP/2 target) | Turbo Intruder + single-packet attack | Most reliable |
| Multi-endpoint with gate | Turbo Intruder + openGate() | Synchronizes different requests |
| Quick CLI test | `curl --parallel` or `parallel` command | No Burp needed |

---

## Phase 4: Exploit → Confirm → Escalate

```
1. BENCHMARK: establish normal behavior (response time, status codes, DB state)
2. RACE: send parallel requests using chosen technique
3. OBSERVE: look for ANY difference in response (duplicate success, different status,
   unexpected state, email anomaly, negative balance)
4. CONFIRM: reproduce reliably at least 3 times
5. ESCALATE: find the highest-impact version of the bug
6. REDUCE: find minimum requests needed to trigger (2 is ideal for report clarity)
```

---

## Severity & Bounty Guide

| Finding | Severity | Bounty Range |
|---|---|---|
| Financial abuse (double spend, negative balance) | Critical | $3K–$20K |
| Account takeover via email/OTP race | Critical | $5K–$25K |
| MFA/2FA bypass | Critical | $3K–$15K |
| Password reset token race → ATO | Critical | $3K–$15K |
| Unlimited coupon/gift card reuse | High | $1K–$8K |
| Subscription paywall bypass | High | $1K–$5K |
| Rate limit bypass (sensitive action) | High | $1K–$5K |
| Invite/role escalation | High | $1K–$5K |
| Partial construction exploit | High | $2K–$10K |
| Resource limit overrun (non-financial) | Medium | $300–$2K |

**Key reporting note:** Always prove impact beyond just "I sent parallel requests."
Show the concrete outcome: negative balance screenshot, duplicate redemption DB state,
access to protected resource, account takeover chain.

---

## Common Mistakes to Avoid

1. **Stopping after one failure** — race windows can be milliseconds wide;
   try different timings, more threads, or different techniques before giving up
2. **Ignoring session locking** — PHP/Rails sessions serialize → use fresh sessions per thread
3. **Network jitter killing the attack** — use single-packet attack or last-byte sync
4. **Not warming the connection** — cold connections add latency variance; always warm first
5. **Reporting without confirmed impact** — "I sent parallel requests" is not a bug;
   always show a concrete outcome
6. **Forgetting multi-endpoint** — most hunters test single endpoint; multi-endpoint is underexplored
7. **Missing partial construction** — registration flows and API key generation are rich targets
