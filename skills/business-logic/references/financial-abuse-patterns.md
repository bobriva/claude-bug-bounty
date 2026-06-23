# Financial Abuse Patterns

Concrete, recurring patterns across coupons, refunds, loyalty/referral systems, gift cards, and subscriptions — the targets named in the original checklist, with the actual mechanism behind each one and what a confirmed version looks like.

## Coupon and discount abuse

**Stacking via race condition** — this is the single most consistently reported and rewarded pattern in this category. A coupon/discount code is meant to be single-use, but the "is it already used" check and the "apply it" step aren't atomic. A real disclosed report against a major grocery-delivery platform described exactly this: a race condition in coupon redemption that let the same code be applied multiple times, stacking the discount with each successful parallel request. See `race-conditions.md` for the exploitation technique — this is the limit-overrun pattern applied specifically to coupon codes.

**Coupon code leakage** — internal/unreleased discount codes sometimes end up reachable: hardcoded in a JS bundle, returned in an API response for a "recommended promotions" feature regardless of eligibility, or guessable from a predictable generation pattern (sequential, timestamp-derived, or a short alphabet). Check JS files and any endpoint that surfaces promotional data, not just the explicit "redeem a code" field.

**Applying a discount to ineligible items** — test whether a discount meant for one product category, one SKU, or one tier of customer can be applied by directly crafting the apply-discount request against an ineligible cart, rather than relying on the UI to only offer it where it's valid.

**Response manipulation on validation** — if the client calls a "validate this coupon" endpoint before applying it, check whether the *validation* response (often a simple `{"valid": true}`) is what the checkout flow actually trusts, separately from the server re-validating at the point of charge.

## Refund and currency abuse

```
- Pay in currency A, request the refund in currency B — if the
  application allows choosing the refund currency independently and
  uses a different (or stale) exchange rate for the conversion, the
  refunded amount can exceed the original payment's value in the
  account's base currency.
- Partial refund logic: request a partial refund, then check whether
  the remaining "refundable" balance on the order is tracked
  correctly — repeated partial refunds sometimes aren't decremented
  from a shared pool correctly, allowing the total refunded to exceed
  the original charge.
- Cancel-after-fulfillment: can an order be cancelled (triggering a
  refund) after it has already shipped/been delivered/been consumed,
  letting the customer keep both the product/service and the money?
- Chained balance manipulation: some flows let a *subscription or
  membership fee* push the account balance negative, and separately
  grant a *credit or discount* tied to having that membership active.
  Test whether those two effects can be combined within the same
  checkout to produce a near-zero or favorable final charge that
  neither rule was individually designed to allow.
```

## Loyalty, referral, and reward abuse

```
- Duplicate referral/signup bonus via identifier normalization gap:
  see `workflow-and-state-abuse.md`'s "parser and identifier
  discrepancies" section — email plus-addressing or dot-insensitivity
  is the most common concrete mechanism. Confirm by actually
  triggering the bonus twice using two identifiers that resolve to
  one real mailbox you control, and showing both credits landing.
- Race-condition duplication of a one-time reward trigger — apply the
  single-packet technique from `race-conditions.md` to the specific
  "claim this reward" endpoint.
- Reward/point value manipulation via client-trusted fields — if the
  number of points awarded for an action is sent by the client rather
  than computed server-side from the action itself, test overriding it.
- Self-referral: can an account refer itself (a second email alias,
  a second device/browser) and claim both the referrer and referee
  bonus from a single real person?
```

**A calibration note on this category specifically:** programs frequently reject referral/promo-abuse reports that only demonstrate a small one-off gain (e.g., "I created two accounts and got two $5 bonuses") without showing the underlying mechanism is genuinely repeatable at meaningful scale. The fix isn't to actually farm the bonus hundreds of times — it's to clearly demonstrate *why* the mechanism scales (the normalization gap or race condition is general, not a one-off fluke) and state the extrapolated impact, rather than either under-proving it (one redemption, no clear repeatability argument) or over-proving it (mass real-world abuse, which itself risks violating program rules).

## Payment and price-bypass chains

```
- Negative fee/tip/surcharge field directly reducing the charged
  total — test every adjustable monetary field on a checkout/booking
  flow for negative-value acceptance, not just the headline price.
- Client-trusted price replay: capture the price the server returned
  for an item, then submit a *lower* value for that same item in the
  next request in the flow, and check whether the final charge uses
  the resubmitted value instead of re-deriving it from the catalog.
- Membership/subscription fee interacting with an automatic discount:
  see the "chained balance manipulation" pattern above — a fee that
  pushes balance negative, combined with a discount that's only
  available *because* that membership is active, can sometimes be
  sequenced to extract more value than either feature owner intended.
- Free-shipping/delivery-fee bypass via a parameter that toggles
  eligibility independent of the actual order total threshold meant
  to gate it.
```

## What makes a financial-abuse finding confirmed, not theoretical

A completed transaction or credited balance with the actual wrong number visible — not "the coupon field accepted a second code" but "the order confirmation shows $X charged when it should show $Y," not "I think referral bonuses could be farmed" but "here are two credits in my own test account's balance history from one real identity." Keep the proof to the smallest real transaction that demonstrates the number is wrong, and reverse/refund it afterward where the program allows.
