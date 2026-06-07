# Session Locking

## Goal

Determine whether framework serializes requests.

---

## Common Behaviors

PHP Session Locking

Framework-level locking

Application-level locking

---

## Testing

Send parallel requests using:

Same Session

Different Sessions

Compare behavior.

---

## Findings

If same session is serialized:

Retry using multiple sessions.