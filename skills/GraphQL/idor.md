# GraphQL IDOR

## Common Pattern

Object lookup by ID.

Example:

product(id:3)

user(id:5)

order(id:10)

---

## Indicators

Sequential IDs

Hidden Objects

Private Data

---

## Testing

Enumerate IDs.

Modify arguments.

Compare responses.

---

## Targets

Users

Orders

Invoices

Products

Messages