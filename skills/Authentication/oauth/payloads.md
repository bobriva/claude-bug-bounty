# OAuth Payloads

## Redirect URI

https://trusted.com@evil.com

https://trusted.com.evil.com

https://trusted.com#evil.com

https://trusted.com?next=https://evil.com

---

## Duplicate Redirect URI

redirect_uri=https://trusted.com

redirect_uri=https://evil.com

---

## Localhost

https://localhost.evil.com

https://127.0.0.1.evil.com

---

## Scope Manipulation

openid profile

openid profile email

openid profile email address

---

## Response Types

response_type=code

response_type=token

response_type=id_token

response_type=id_token token

response_type=id_token code