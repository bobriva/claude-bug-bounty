# State Machine Abuse

Applications often assume users move through states in a fixed sequence.

Example:

Pending
→ Approved
→ Completed

Test:

Pending → Completed

Pending → Archived

Approved → Pending

Completed → Approved

---

## Questions

Can states be skipped?

Can states be repeated?

Can states be reverted?

Can invalid states exist?

Can two states exist simultaneously?