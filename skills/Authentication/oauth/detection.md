# OAuth Detection

## OAuth Indicators

Parameters:

client_id
redirect_uri
response_type
scope
state

---

## Authorization Code Flow

response_type=code

---

## Implicit Flow

response_type=token

---

## OpenID Connect

scope=openid

response_type=id_token

---

## Discovery Endpoints

/.well-known/oauth-authorization-server

/.well-known/openid-configuration

---

## Interesting Endpoints

/authorize
/auth
/token
/callback
/userinfo
/register