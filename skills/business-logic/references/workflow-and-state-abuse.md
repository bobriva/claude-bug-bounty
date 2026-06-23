# Workflow & State-Machine Abuse

## Step skipping, reordering, and replay

Multi-step workflows (registration → verification, checkout → payment → confirmation, application → review → approval) almost always assume the client will call each step in order, exactly once, having actually completed the one before. Test all three assumptions independently:

```
Skip:    call step 3's endpoint directly, with no step 1 or 2 ever completed
Reorder: call step 2 before step 1
Replay:  call step 2 again after already completing the full flow once
```

The most valuable variant of "skip": calling the **final/confirmation** endpoint directly while supplying whatever state values it expects from the earlier steps (a payment-status flag, a verification-complete flag, an approval ID) — see the "client-trusted state values" section below, since this is usually how the skip becomes more than a 400 error.

**Forced browsing with a Referer check bypass:** if a step is gated by checking `Referer` rather than actual server-side state, set `Referer` to the URL of the page that's supposed to precede it and retry.

## Client-trusted state values

The recurring root cause across almost every high-value business logic bug: the server trusts a value the client echoes back, instead of recomputing or re-verifying it server-side at the point that matters.

```
- price / unit_price / total           (client built the cart total)
- status: "paid" | "verified" | "approved"
- quantity (especially negative — see input-and-parameter-abuse.md)
- role / tier / is_premium
- discount_applied / discount_amount
```

**Test pattern:** capture the normal request for a state-changing action, identify which of these fields the client controls, then submit the same request with that field changed to the value you want, and check whether the server actually re-derives it or just stores what you sent. The PortSwigger "excessive trust in client-side controls" lab category is built entirely around this pattern — e.g., editing the price field in an add-to-cart or checkout request and observing whether the final charge reflects the edited value or the real catalog price.

## State machine abuse

Map the application's intended state diagram for any resource that has one (orders, approvals, subscriptions, tickets, applications):

```
Example intended flow:
Pending -> Approved -> Completed
                     -> Cancelled
```

Then attempt every transition that *isn't* drawn on that diagram:

```
Pending -> Completed       (skip approval entirely)
Completed -> Pending        (revert a finished state — sometimes re-triggers
                              a payout, notification, or fulfillment action)
Approved -> Pending         (undo, then see if the resource still has the
                              privileges/access that "Approved" granted)
Cancelled -> Completed      (resurrect a cancelled order/refund after the
                              refund has already been issued)
```

Ask directly: can two states exist for the same resource simultaneously (e.g., an order that's both "refunded" and "shipped" because two different services updated it independently without checking the other's current state)? That kind of split-brain state is often where the highest-impact findings live, because it usually means the business got money/access taken away in one place while still delivering value in another.

**Confirming, not just observing:** you need the resource to actually land in the illegal state (check it via a GET/detail endpoint afterward) and for that state to grant something it shouldn't — access, a payout trigger, a completed fulfillment without payment. An API call "succeeding" with a 200 isn't enough if the underlying resource didn't actually change.

## Parser and identifier discrepancies

Different parts of the same system frequently disagree about whether two identifiers are "the same":

```
- Email plus-addressing / dot-insensitivity:
  user@example.com vs user+promo@example.com vs u.s.e.r@example.com
  (Gmail treats all of these as one mailbox; many backends store them
  as distinct accounts unless explicitly normalized)

- Unicode lookalikes / case folding:
  ADMIN vs Admin vs аdmin (Cyrillic а) — uniqueness checks and
  authorization checks sometimes normalize case differently from
  each other

- Phone number formatting: +1-555-0100 vs 5550100 vs (555) 0100 —
  test whether the uniqueness check and the actual delivery/SMS
  routing treat these as matching or not, since a mismatch here
  can let the same physical line register multiple "unique" accounts

- Currency/locale-dependent number parsing: "1,000" as one thousand
  in some locales and one in others (decimal vs thousands separator)
  — test whether switching the account/request locale changes how
  a numeric field already on the books gets interpreted downstream
```

**Why this matters for business logic specifically:** per-account limits (one free trial, one signup bonus, one referral credit) are usually enforced by checking "has this email/phone been used before" — not by checking "has this *person*." A normalization gap directly converts a one-time benefit into a repeatable one. See `financial-abuse-patterns.md` for how this chains into referral/promo farming.

**Confirming, not just observing:** register (or simulate registering) two account identifiers that a real provider would treat as identical, and show the system issuing a second instance of a benefit that's supposed to be one-per-person — not just "these two strings look different to the validator."

## Encryption oracles and feature misuse

Occasionally a completely unrelated feature can be repurposed as a primitive for something else entirely — for example, an application that reflects user input into an encrypted cookie/token can sometimes be used as an oracle: submit chosen plaintext, observe the resulting ciphertext, and use that to forge a different encrypted value (such as another user's session token or an admin notification cookie) that the application will accept elsewhere. This is the purest form of "use the application exactly as intended, in a way nobody anticipated" — when auditing any feature that takes user input and returns it back in an encrypted or signed form, ask whether that input is fully attacker-controlled and whether the output format matches something else the application trusts elsewhere.

## What makes a workflow/state finding confirmed, not theoretical

The resource needs to actually end up in the wrong state, with a real, checkable artifact (an order record, an account balance, a granted permission) — and that wrong state needs to grant or skip something with real business consequence. "I could call the endpoints out of order" without showing the resulting state and its effect is a lead, not a finding.
