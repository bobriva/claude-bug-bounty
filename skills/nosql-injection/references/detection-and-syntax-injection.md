# Detection & Syntax Injection

## Backend fingerprinting

Before picking payloads, narrow down which NoSQL engine you're dealing with — exploitation syntax doesn't transfer between them:

```
MongoDB indicators:
  - 24-character hex strings in URLs/responses (ObjectId format)
  - Node.js/Express stack fingerprints (X-Powered-By, error stack traces)
  - JSON request bodies on login/search endpoints
  - Error messages mentioning "BSON", "Cast to ObjectId failed", or
    Mongoose/Mongoid validation error formats

CouchDB indicators:
  - Port 5984 reachable
  - Error responses shaped like {"error":"not_found","reason":"missing"}
  - "/_all_dbs", "/_utils" paths present

DynamoDB / Firebase indicators:
  - AWS-style API Gateway responses, X-Amzn-RequestId headers
  - Firebase Realtime Database URLs (*.firebaseio.com) or Firestore
    REST endpoints
  - These differ structurally from MongoDB — see the closing section
    of `blind-extraction-and-where-rce.md` for what's actually
    testable on them.
```

MongoDB is overwhelmingly the most common target for this skill in practice — most of this skill's depth assumes MongoDB unless stated otherwise.

## Syntax fuzzing

Submit special characters individually into any parameter that reaches a query, and watch for error messages, status changes, or content differences:

```
'  "  `  {  }  ;
```

A general-purpose fuzz string that triggers a reaction across multiple NoSQL syntaxes at once, useful as a fast first probe:
```
'"`{ ;$Foo} $Foo \xYZ
```

If a single quote breaks the query (causes an error or behavior change) but an escaped quote (`\'`) doesn't, that's a strong signal the input lands inside a string-based query context without sanitization.

## Boolean condition testing

Once syntax injection is confirmed, test whether you can influence query logic with a true/false pair, comparing the two responses for differences in content, length, or record count:

```
False condition: ' && 0 && 'x
True condition:  ' && 1 && 'x
```

An "always true" tautology, useful for confirming injection broadly (e.g., on a category/search filter) by making the query return everything regardless of the intended filter:
```
'||'1'=='1
```

## Null-byte truncation

Some MongoDB string-matching contexts stop processing at a null byte, silently dropping any conditions that appear after it in the constructed query:

```
%00
```

If a query is built as `this.category == '<input>' && this.released == 1`, injecting a null byte immediately after closing the intended value can cause everything from the null byte onward — including the `released == 1` restriction — to be ignored, exposing unreleased/hidden records through an otherwise-normal-looking filter.

## What this phase confirms vs. what it doesn't

Syntax and boolean testing confirm *that* injection is possible and roughly *where* the input lands in the query — they don't yet demonstrate impact on their own. "The response changed when I sent `' && 1 && 'x`" is the signal to move into `operator-injection-and-bypass.md` and `blind-extraction-and-where-rce.md`, not a finding by itself; see the validation gate in `SKILL.md`.
