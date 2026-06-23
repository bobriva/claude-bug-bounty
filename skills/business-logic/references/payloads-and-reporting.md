# Payloads, Severity Framework, and Report Template

## Input-abuse payload cheat sheet

```
Negative:    -1, -0.01, -999999
Zero:        0
Decimal where integer expected: 1.5, 2.99
Oversized:   999999999, 1e308
Empty/null:  "", null
Missing:     (omit the field entirely)
Wrong type:  "abc" where a number is expected; true where a string is expected
Array/object substitution: [1, 999] or {"amount": 0.01} in place of a scalar
Locale-confusable numbers: "1,000" vs "1.000" vs "1 000"
```

## Parameter removal levels (test all three)

```
1. Value only:    "price": ""
2. Parameter:     (omit "price" key entirely)
3. Whole object:  (omit the entire pricing/discount sub-object)
```

## Identifier-normalization probes

```
user@example.com
user+test1@example.com
user+test2@example.com
u.s.e.r@example.com
USER@EXAMPLE.COM
```

## Workflow-abuse quick probes

```
Call step N+1 directly without completing step N
Replay a completed step's request a second time
Submit steps in reverse order
Set a client-trusted status field directly: "status": "paid" / "verified" / "approved"
```

---

## Severity framework (business logic doesn't map cleanly to CVSS)

CVSS is built around confidentiality/integrity/availability of a system — it doesn't have a clean axis for "this lets someone get free money." Most programs score business logic findings on actual demonstrated business impact instead. Use this framework when writing the severity section, and be explicit that you're using business-impact terms rather than forcing a CVSS vector that doesn't fit:

| Severity | Criteria |
|---|---|
| **Critical** | Unlimited or trivially automatable monetary gain or free goods/services, no realistic precondition, demonstrated mechanism that clearly scales (e.g., a race condition or normalization gap with no per-attempt friction) |
| **High** | Meaningful bounded monetary gain per attempt (a specific dollar amount, a specific quantity), repeatable but requires some minor precondition (e.g., owning two email aliases, timing a request) |
| **Medium** | Limited one-off gain, requires a specific precondition that narrows real-world likelihood (a particular account state, a particular timing window) but is still realistically reachable by an ordinary attacker |
| **Low/Informational** | Theoretical or requires an unrealistic precondition (predicting exact registration timing, requiring insider access), or impact is self-only with negligible value |

When in doubt, state the actual dollar/point/quantity delta you demonstrated and let the triager weigh it — a precise small number ("$2.50 per attempt, repeatable with zero per-attempt friction") is more convincing and more honestly scored than an unsupported "Critical" label.

## Full report template

```markdown
**Title**: [Mechanism] in [exact workflow] allows [actor] to [quantified impact]

## Summary
2-3 plain sentences: the assumption that was violated, the workflow it's
in, and the quantified result. No preamble.

## Steps to Reproduce
1. [Setup — account state, cart contents, starting balance, etc.]
2. Send: [exact request(s) — method, path, headers, body; if a race
   condition, specify how many parallel requests and the technique used]
3. Observe: [the resulting state — order total, balance, redemption count]
4. [If applicable] Repeat to confirm the result is consistent, not a
   one-off fluke

## Proof of Concept
Before/after state, explicitly: starting balance vs. ending balance,
listed price vs. charged price, intended single redemption vs. actual
count. Screenshots or exported request/response pairs.

## Impact
State the exact numbers demonstrated (e.g., "$45 discount applied
instead of $15, a $30 loss per attempt") and, separately and clearly
labeled as an estimate, the extrapolated risk if automated at scale —
without having actually run it at scale.

## Suggested Fix
One or two sentences specific to this workflow — e.g., "Enforce
single-use coupon redemption atomically (database-level unique
constraint or row lock), not via a check-then-act read followed by a
separate write."

## Severity Assessment
[Critical/High/Medium/Low] — business-impact basis (see framework
above), not CVSS: [one line stating which criteria applied]
```

### Tone notes

- Lead with the actual number, not a description of the workflow — "A single $15 coupon code can be applied 3 times on one order, reducing a $45 cart to $0" beats three paragraphs explaining what coupons are.
- Always show the real before/after state. A business-logic report without a concrete before/after number reads as speculation, no matter how well the mechanism is explained.
- State plainly whether reproducing this required special conditions (two email aliases you control, a race condition tool, a specific account tier) — triagers will ask if you don't, and pre-answering it builds trust.
- Don't claim "this could bankrupt the company" or similar — state the per-attempt number and, if relevant, a soberly-labeled scale estimate. Triagers discount hyperbole quickly.
- Mention that you reversed/refunded the test transaction where possible — it signals good faith and is often required by program rules.
- Keep it under ~600 words, consistent with the other skills in this set.
