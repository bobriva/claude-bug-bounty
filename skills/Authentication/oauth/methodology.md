# OAuth Testing Methodology

## Phase 1 - Identify OAuth

Look for:

- Login with Google
- Login with GitHub
- Login with Facebook
- Login with Microsoft

Observe:

- /authorize
- /authorization
- /oauth
- /callback
- /token

---

## Phase 2 - Discover Flow

Authorization Code Flow

response_type=code

Implicit Flow

response_type=token

OIDC

scope=openid

response_type=id_token

---

## Phase 3 - Recon

Check:

/.well-known/oauth-authorization-server

/.well-known/openid-configuration

Identify:

- Supported flows
- Endpoints
- Registration endpoints
- Scope support

---

## Phase 4 - Security Testing

Test:

- redirect_uri
- state
- scope
- response_mode
- request_uri
- client registration

---

## Phase 5 - Exploitation

Verify:

- Account takeover
- Token theft
- Authorization code theft
- Privilege escalation