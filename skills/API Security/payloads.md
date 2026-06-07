# API Security Payloads

## HTTP Methods

GET

POST

PUT

PATCH

DELETE

OPTIONS

HEAD

---

## Content Types

application/json

application/xml

application/x-www-form-urlencoded

multipart/form-data

text/plain

---

## Mass Assignment

{
  "isAdmin": true
}

{
  "role": "admin"
}

{
  "permissions": ["admin"]
}

{
  "verified": true
}

---

## Version Discovery

/v1/

/v2/

/v3/

/beta/

/internal/