---
name: business-logic
description: Identify and exploit business logic flaws — workflow step-skipping, state-machine abuse, race conditions/limit-overrun, price and parameter tampering, negative-value/overflow abuse, and coupon/discount/referral/loyalty/payment fraud patterns — for authorized bug bounty and pentest work. Covers single-packet and last-byte-sync race condition techniques (Burp "send group in parallel," Turbo Intruder) for winning limit-overrun races reliably, client-trusted price/status/quantity manipulation, parser discrepancies (email plus-addressing, currency switching) that bypass per-account limits, and a validation gate plus report template focused on demonstrating real before/after financial or access impact. Use whenever scanning yields little, or the target has checkout/payment, discounts/coupons, refunds, rewards/loyalty, gift cards, subscriptions, approval workflows, or any multi-step process — even if the user just describes a feature and asks "is there a way to abuse this?"
---

# Business Logic Testing

Business logic flaws are invisible to scanners and often the highest-business-impact findings a program will see — a single confirmed race condition on a payment flow routinely outranks a dozen reflected-XSS reports. But they're also the easiest category to over-claim in, because "I think this could be abused" is a much weaker statement than "I have a confirmed order, charged $0, that the system accepted." This skill exists to close that gap.

## Core principle

Don't ask **"is this vulnerable?"** Ask **"what assumption did the developer make, and what happens if I violate it?"** Every workflow encodes assumptions: that users follow the steps in order, that a value the client sends back is the same one the server sent out, that two requests for the same resource won't arrive at the same instant, that one email address means one human. Your job is to enumerate those assumptions and try to break each one — not to "hack" anything in the traditional sense, since most business logic exploitation uses entirely legitimate functionality, just in a sequence or combination the developer never tested.

Once you have a candidate, the same discipline from the other skills in this set applies: treat it as a **lead, not a bug**, until you've completed the action end-to-end and can show a concrete before/after state (an order total, an account balance, a redemption count) that proves the business rule was actually broken — not just that a request was accepted or a response looked unusual.

## Before you start: scope and footprint

This category more than any other risks causing real financial/business impact if tested carelessly:

- **Use your own test account(s) and the smallest amount/quantity that still proves the bug.** A $0.01 order proves price manipulation as well as a $10,000 one does.
- **Check whether the program allows triggering real transactions.** Some require sandbox/test payment methods, advance coordination, or a cap on financial-impact testing — check before running anything that moves real money, issues real refunds, or creates real reward-point balances.
- **Where possible, reverse what you can** (cancel the order, return the item, void the transaction) after capturing proof, and say so in the report.
- **Never run abuse at real scale** (hundreds of fake accounts, mass coupon redemption) to "prove" automatability — demonstrate the mechanism once, cleanly, and state the extrapolated impact in the writeup instead of actually causing it.

## The methodology

| Phase | Goal | Reference |
|---|---|---|
| 1. Understand the business | What generates revenue, what creates cost, what grants access | This file, below |
| 2. Map workflows & state machines | Every multi-step process and its intended state transitions | `references/workflow-and-state-abuse.md` |
| 3. Abuse input & parameters | Negative/zero/overflow values, parser discrepancies, hidden fields | `references/input-and-parameter-abuse.md` |
| 4. Race conditions | Limit-overrun via concurrent requests | `references/race-conditions.md` |
| 5. Financial-pattern abuse | Coupons, discounts, referrals, loyalty, gift cards, refunds, subscriptions | `references/financial-abuse-patterns.md` |
| 6. Validate | Run every candidate through the gate below | This file, "Validation gate" |
| 7. Report | Concrete before/after state, quantified impact | This file + `references/payloads-and-reporting.md` |

## Phase 1: understand the business

Before touching any request, answer these about the target:

- What action makes the business money (a purchase, a subscription, a transaction fee)?
- What action costs the business money (a refund, a payout, a reward redemption, free shipping)?
- What grants access or privilege (an approval, a verification, a role)?
- Where do humans usually slow an attacker down on purpose (CAPTCHA, manual review, a confirmation step, a cooldown)? Those are exactly the points worth attacking first — they exist because an unrestricted version of that action is valuable enough to protect.

Then map every multi-step workflow you can find: registration, checkout, payment, refund, reward redemption, promotion/coupon application, approval chains, subscription upgrade/downgrade/cancellation. For each, ask the questions in the hunting checklist below.

## The hunting checklist — ask this of every workflow

- Can I skip a step?
- Can I repeat a step?
- Can I reverse a step?
- Can I perform two steps at once (race them)?
- Can I perform steps out of order?
- Can I modify a value the client shouldn't control (price, status, role, quantity)?
- Can I remove a parameter, or the whole object, instead of changing its value?
- Can I submit a negative, zero, decimal, oversized, or otherwise "impossible" value?
- Can I make the system treat two different identifiers (emails, accounts, coupons) as the same — or the same one as different?
- Can I combine two unrelated features to produce an effect neither was designed for alone?

Always look for: **money, access, privilege, scale, and fraud potential** — that's where the impact lives.

## Validation gate — run this before calling anything a finding

Every "no" means: keep digging, downgrade to "suspected," or drop it.

1. **Did you complete the action end-to-end with a real before/after state?** An order actually placed at the wrong price, a coupon actually applied twice on a confirmed order, a reward balance that actually increased twice for one action — not "the request was accepted" or "the response looked different."
2. **Is the impact quantifiable?** State the actual numbers: the price delta, the number of times an action was duplicated, the balance gained. "Could potentially be abused for financial gain" is not a number.
3. **Did you keep the footprint minimal?** Smallest amount, your own account, reversed/refunded afterward where possible.
4. **Was this allowed under program scope?** Re-check before anything that creates real financial movement or affects production data outside your own test account.
5. **Is it a duplicate?** Negative-quantity price manipulation, "no rate limit on coupon application," and basic step-skipping are extremely commonly reported — check disclosed reports before assuming novelty; novelty isn't required for a valid report, but it changes how you frame severity expectations.
6. **Would a triager agree a business rule was actually broken, not just that an edge case behaves oddly?** A negative quantity that the UI displays strangely but that the server ultimately rejects at checkout is not a finding.

### Patterns that usually die at this gate (don't submit these alone)

- A negative/zero/overflow value accepted by one API but rejected at the final confirmation step, with no completed transaction showing actual impact
- "I sent two requests close together and got slightly different responses" with no clean, repeatable win and no proof of an exceeded limit in the actual account/order state
- A workflow step that *can* technically be called out of order, with no demonstrated unintended outcome (e.g., calling a "confirm" endpoint early that simply re-validates and rejects)
- A hidden/admin-reachable function found, with no demonstrated unauthorized data access or action — this is really a BFLA finding; see the `api-security` skill's `vulnerability-classes.md` for that validation bar
- Theoretical coupon-stacking ("the UI doesn't prevent applying two codes") with no completed order showing the stacked discount actually reflected in the final charge

### Patterns that become valid once chained

| Weak alone | Chain it with | Becomes |
|---|---|---|
| Workflow lets you call the "confirm" step early | A request crafted with the required state values pre-set (e.g., `payment_status: "paid"`) accepted without real payment | Free/unpaid order — confirmed revenue loss |
| Negative quantity accepted by the cart API | A completed checkout where the final charged total is actually reduced/zeroed | Confirmed price manipulation, not just a display glitch |
| Suspected race condition on a limit-checked resource | A clean win using the single-packet attack technique, with before/after state showing the limit exceeded | Confirmed limit-overrun (double redemption, stacked discount, duplicate reward) |
| Per-account limit on a promo/free-trial | A parser discrepancy (email plus-addressing, dot-insertion) shown to register as a "new" account while still controlled by you | Confirmed unlimited promo/trial harvesting |
| Client-supplied discounted price in a response | That manipulated value actually accepted and reflected in a placed, confirmed order | Confirmed full price-bypass, not just "the response was editable" |

Spend the extra time chaining a weak signal into an actually-completed transaction before writing it up — in this category specifically, the gap between "interesting" and "paid" is almost always whether you have a real confirmed before/after state.

## Writing it up

Same standard as the other skills in this set: lead with the concrete impact, not a description of the workflow. For business logic specifically, the impact section should read like an incident report, not a hypothesis — exact numbers, exact before/after state, and (carefully) an extrapolated scale estimate without having actually run that scale.

**Title formula:** `[Mechanism] in [exact workflow] allows [actor] to [quantified impact]`
Good: `Race condition in /api/coupons/claim allows a single coupon code to be applied 3x on one order, reducing total by $45 instead of $15`
Bad: `Business logic bug in checkout`

Full report template, payload cheat-sheet, and severity guidance: `references/payloads-and-reporting.md`.

## Reference files in this skill

- `references/workflow-and-state-abuse.md` — step skipping, state-machine abuse, client-trusted status/price values
- `references/input-and-parameter-abuse.md` — negative/zero/overflow values, parser discrepancies, hidden-field and integrity-token abuse
- `references/race-conditions.md` — limit-overrun detection, single-packet attack, last-byte sync, real disclosed examples
- `references/financial-abuse-patterns.md` — coupon/discount/referral/loyalty/gift-card/refund/subscription abuse patterns
- `references/payloads-and-reporting.md` — ready-to-use payloads, severity guidance, full report template
