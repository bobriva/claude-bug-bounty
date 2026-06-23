# Input & Parameter Abuse

Business logic doesn't need injection payloads — it needs values that are *technically valid input* but fall outside the range a developer ever imagined a legitimate user typing. Test systematically rather than randomly.

## Value-range abuse

```
Negative:   quantity=-1, amount=-50.00, tip=-1000, days=-30
Zero:       quantity=0, amount=0, weight=0
Decimal where an integer is expected: quantity=1.5, seats=2.99
Oversized:  quantity=999999999, amount=1e308
Empty/null: amount="", amount=null
Wrong type: quantity="abc", amount=true
```

**Why negative values are disproportionately effective:** a lot of pricing logic is just `total = unit_price * quantity` (or `total - discount`, `total + tip`) computed once and trusted afterward. A negative quantity or negative tip/fee can drive the total below zero, and if the only server-side check is "is the final total non-negative," adding enough *positive*-quantity items to cancel the negative one out can land exactly on a near-zero or negative-then-clamped total — see the PortSwigger "high-level logic flaw" pattern: a negative-quantity item combined with positive-quantity items specifically to make the combined total small enough to afford with insufficient store credit. The same root cause shows up in real payment integrations as a negative tip/fee field that single-handedly reduces the charged total by an attacker-chosen amount.

## Parameter removal (three distinct levels — test all three, not just one)

```
1. Remove the value only:   "price": ""
2. Remove the parameter:    (omit "price" entirely from the JSON body)
3. Remove the whole object: (omit the pricing/discount object that
                              normally wraps several related fields)
```

Different code paths sometimes handle each level differently — a missing *value* might get rejected by validation, while a missing *parameter* falls through to a default (sometimes a permissive one, like "no restriction" rather than "no access").

## Hidden-field and integrity-token abuse

```
- Diff the full response body against what the UI actually renders —
  hidden fields (is_eligible_for_discount, internal_cost, max_quantity)
  are often present in JSON even though no UI control exposes them.
- If a price/total/discount field is accompanied by a hash, checksum,
  or signature meant to prevent tampering, check whether it's actually
  verified server-side, and whether you can recompute it yourself if
  the algorithm/secret is discoverable (client-side JS, a leaked key,
  or a weak/guessable scheme).
- Try resubmitting a previous response's values verbatim in a new
  request — sometimes a value that was merely *informational* in one
  context becomes load-bearing if replayed into a different endpoint.
```

## Type and encoding confusion

```
- Array where a scalar is expected: "quantity": [1, 999]
- Object where a scalar is expected: "price": {"amount": 0.01}
- String containing a numeric-looking value with locale tricks:
  "1,000" / "1.000" / "1 000" (thousands vs. decimal separator
  parsed differently depending on locale context)
- Scientific notation: "amount": "1e-10"
```

## Where to look first

Prioritize fields that influence **money, quantity, or state**, in roughly this order of historical payout value::
```
price / unit_price / total / subtotal
quantity / count / seats / units
discount_amount / discount_percent
tip / fee / surcharge / delivery_charge
status / state / approved / verified
currency (see financial-abuse-patterns.md for currency-switching abuse)
```

## What makes an input-abuse finding confirmed, not theoretical

The unusual value needs to survive all the way to a real, completed action with a visibly wrong result — an order actually placed at the wrong total, a balance that actually went negative, a quantity that actually shipped as negative or zero with a refund still issued. A value being *accepted* at one step but clamped, recalculated, or rejected before the action completes is not yet a finding — keep tracing it through the rest of the flow before concluding it's exploitable.
