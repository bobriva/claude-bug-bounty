# Race Condition Methodology

## Core Principle

Every request may contain hidden sub-states.

The goal is to interact with shared state during the race window.

---

## Predict → Probe → Prove

### 1. Predict

Ask:

- Is endpoint security sensitive?
- Does it modify shared state?
- Can two requests affect same object?

Examples:

- Coupons
- Gift Cards
- Password Reset
- MFA
- Email Verification

---

### 2. Probe

Benchmark normal behavior.

Then send requests in parallel.

Look for:

- Different responses
- Duplicate actions
- Unexpected state changes
- Email anomalies

---

### 3. Prove

Reduce requests.

Reproduce reliably.

Escalate impact.