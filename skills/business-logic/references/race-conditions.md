# Race Conditions in Business Logic

Race conditions are consistently under-tested because automated scanners can't find them — there's no signature to match, no payload to fuzz for. That makes this one of the highest-value categories left to manual testers, and it's the single most reliable technique for turning a "single-use" business rule (one coupon, one free trial, one withdrawal) into a repeatable one.

## Limit-overrun races (the common case)

**Pattern:** any place the application checks "has this already been used / is this within the limit" and then, in a separate step, *applies* the action — if those two steps aren't atomic, firing many requests at the same instant can get all of them to pass the check before any of them have recorded their own effect.

**High-value targets:**
```
Coupon/discount code redemption        (apply the same single-use code N times)
Gift card / store-credit redemption    (redeem the same code N times)
Referral/signup bonus claims           (claim once per "unique" trigger, N times)
Withdrawal / balance deduction         (withdraw more than the balance allows)
Voting / rating / review submission    (submit more than once where "once" is enforced)
Invite/seat-limited registration       (register more users than the invite allows)
Rate limits and lockout counters       (see the `authentication` skill's
                                         enumeration-and-bruteforce.md — same
                                         underlying bug class, applied to login)
```

## Detecting a candidate

```
1. Identify a single HTTP request that triggers a check-then-act sequence
   tied to a limited/single-use resource.
2. Send 2 copies of that exact request using Burp's "Send group in
   parallel" (or via Repeater tabs grouped together).
3. If both succeed when only one should have, you have a race condition —
   confirm cleanly with the single-packet technique below before treating
   it as solid.
```

## Exploiting reliably: the single-packet attack

Sending requests "at the same time" over normal HTTP/1.1 connections is unreliable because of network jitter — the requests don't actually arrive simultaneously at the server. The single-packet attack technique solves this by withholding the final bytes of several HTTP/2 requests until all of them are queued, then releasing all the final frames together in one TCP packet, so the server receives ~20-30 requests effectively simultaneously.

```python
# Turbo Intruder script skeleton (race-single-packet-attack.py pattern)
def queueRequests(target, wordlists):
    engine = RequestEngine(
        endpoint=target.endpoint,
        engine=Engine.BURP2,        # required for single-packet attack
        concurrentConnections=1,
    )
    for i in range(20):
        engine.queue(target.req, gate='race1')
    engine.openGate('race1')        # releases all queued requests at once
    engine.complete(timeout=10)

def handleResponse(req, interesting):
    table.add(req)
```

Requirements and gotchas:
```
- Target must support HTTP/2 — the single-packet attack does not work
  over HTTP/1.1.
- In Burp Repeater: group the duplicated requests into a single tab
  group and use "Send group in parallel," which uses the same
  single-packet/last-byte-sync mechanism under the hood.
- If timings still look inconsistent even with the single-packet
  technique, the backend itself may have variable processing delay —
  try "connection warming" (send a few harmless requests first to
  stabilize timing) before the real attack burst.
- Negative response timestamps in Turbo Intruder's output (the
  response arriving before Turbo Intruder's log thinks the request
  was fully sent) are a strong positive signal that real overlap
  occurred — not a bug in the tool.
- On HTTP/1.1-only targets, fall back to last-byte synchronization
  (queue requests with all but the final byte sent, then release the
  final byte of each almost simultaneously) — slower to set up but
  still effective.
```

## Beyond limit-overrun: sub-state races

A single request often triggers an entire internal multi-step sequence server-side — entering and then exiting several hidden intermediate states before the response is sent. If you can find two or more HTTP requests that interact with the *same underlying data* during that window, you can sometimes expose time-sensitive logic flaws that have nothing to do with a simple counter limit. Documented examples of this broader class include flawed multi-factor login flows where the first half of login (submitting valid credentials) briefly establishes enough server-side state that, raced against a second request, lets an attacker proceed past the second factor entirely — not because any single check is missing, but because two requests touching the same in-progress login state interleave in a way neither was designed to handle. When hunting in this category, think about *any* multi-request sequence touching the same record (not just obvious "redeem"/"claim" actions) as a candidate, and consider what server-side sub-state might exist briefly between requests.

## Confirming, not just observing

A race condition finding needs a **clean, repeatable win with a real before/after state**: re-run the attack a few times and confirm it succeeds consistently, then show the actual resulting state — a balance that exceeds what should have been possible, a coupon applied N times on one confirmed order, N reward credits issued for what should have been a single trigger. "I sent parallel requests and got slightly different response codes" without that final-state proof is not yet a finding; keep digging until you can point at the actual account/order/balance record showing the limit was exceeded.

## Tools

| Tool | Use |
|---|---|
| Burp Suite "Send group in parallel" | Quickest way to test a race candidate with no scripting |
| Turbo Intruder | Full control over the single-packet/last-byte-sync engine, scriptable, needed for anything beyond a simple 2-request race |
| `h2spacex` | HTTP/2 single-packet attack as a standalone Python/Scapy library, for cases outside Burp |
