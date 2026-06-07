# Multi Endpoint Race Conditions

## Definition

Two different endpoints interact with same state.

---

## Examples

Payment + Cart Update

Email Change + Verification

Password Reset + Login

MFA Setup + Protected Resource

---

## Goal

Force inconsistent state transitions.

---

## Testing

Identify workflows.

Map backend state.

Send endpoints simultaneously.

Observe state conflicts.