# GraphQL Methodology

## Phase 1 - Endpoint Discovery

Common paths:

/graphql

/api/graphql

/graphql/api

/graphql/graphql

/graphql/v1

---

## Phase 2 - Universal Query

Test:

query{__typename}

Expected:

{"data":{"__typename":"query"}}

---

## Phase 3 - Schema Discovery

Attempt:

Introspection

Suggestions

Clairvoyance

---

## Phase 4 - Authorization Testing

Inspect:

Queries

Mutations

Arguments

Object IDs

---

## Phase 5 - Abuse Testing

Aliases

CSRF

Depth Abuse

Operation Abuse