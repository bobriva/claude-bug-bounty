# NoSQL Injection Methodology

## Phase 1 - Recon

Identify:

- MongoDB
- CouchDB
- Firebase
- DynamoDB

Indicators:

- JSON APIs
- BSON responses
- Mongo error messages

---

## Phase 2 - Syntax Testing

Test:

'
"
`
{
}

Observe:

- Error messages
- Response differences
- Status changes

---

## Phase 3 - Boolean Testing

False condition

True condition

Compare:

- Content
- Response size
- Result counts

---

## Phase 4 - Operator Injection

Test operators:

$ne

$gt

$lt

$in

$regex

$where

---

## Phase 5 - Data Extraction

Enumerate:

- Users
- Fields
- Passwords
- Secrets