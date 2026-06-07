# Mass Assignment

## Definition

Mass Assignment occurs when request parameters are automatically bound to internal object properties.

---

## Discovery

Compare:

GET response

PATCH request

POST request

---

## Example

Response:

{
  "id": 123,
  "email": "user@example.com",
  "isAdmin": false
}

Potential hidden fields:

id

isAdmin

role

permissions

---

## Testing

Add hidden field.

Modify value.

Observe:

- Different responses
- Privilege changes
- Authorization changes

---

## High Value Targets

isAdmin

role

permissions

accountType

verified

status