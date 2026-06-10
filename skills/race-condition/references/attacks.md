# Attack Patterns Reference

## 1. Limit Overrun (Most Common)

### What It Is

The most common race condition. The backend performs a CHECK then an UPDATE
as two separate non-atomic operations. Race multiple requests into the gap.

```
Vulnerable pattern:
  1. SELECT: "has user used this coupon?" → NO
  2. [RACE WINDOW — this is where you attack]
  3. INSERT/UPDATE: mark coupon as used
  4. Apply discount

Safe pattern (atomic):
  1. UPDATE coupons SET used=true WHERE code=X AND used=false
     → affected_rows = 0? → reject
     → affected_rows = 1? → apply discount
```

### Targets

```
Coupon/Discount Codes:
- POST /apply-coupon
- POST /checkout/discount
- POST /cart/promo

Gift Cards:
- POST /redeem-gift-card
- POST /wallet/add-funds
- POST /gift/redeem

Referral Rewards:
- POST /referral/claim
- POST /invite/complete

Rate-Limited Actions:
- POST /vote
- POST /like
- POST /share-bonus
- POST /daily-reward

Subscription/Paywall:
- POST /upgrade-trial
- POST /activate-feature
- POST /claim-free-tier

API Credits:
- POST /api/consume-credit
- POST /query (AI API with usage limits)
```

### Step-by-Step Exploit

```
1. Identify the endpoint and capture request in Burp
2. Test single request → confirm it works (check state)
3. Reset state (delete test order, un-redeem coupon if possible)
4. Create Burp Repeater group with 20x the same request
5. Send group in parallel → check responses
6. If any multiple successes → confirm state (is discount applied multiple times?)
7. Scale up or down thread count to find reliable reproduction
```

### Impact Escalation

```
Basic: Apply $10 coupon twice = $20 discount
Escalated: Send 1000 parallel requests → near-zero price
Financial: Gift card with $50 → race 20 times → $1000 extracted
Paywall: Free trial → race upgrade → permanent premium access
```

---

## 2. Single-Endpoint Race Conditions

### Pattern A: Email Verification Token Misrouting

**Based on James Kettle's research (GitLab CVE-2022-4037 class)**

```
Workflow:
User → POST /change-email {email: "new@email.com"}
Server → generates verification token
Server → sends token to "new@email.com"
User → clicks link → account email updated

Race attack:
1. Send POST /change-email {email: "attacker1@evil.com"} ─┐ simultaneously
2. Send POST /change-email {email: "attacker2@evil.com"} ─┘
3. Server races:
   - generates token for attacker1 → may send it to attacker2's inbox
   - generates token for attacker2 → may send it to attacker1's inbox
4. If token is misrouted → click link → verify using other attacker's token
5. Chain: register with victim's email → race email change → get victim's verify token
   → verify victim's email → take over victim account
```

### Pattern B: Password Reset Token Race

```
Attack: Request 2 password resets for same account simultaneously

Normal: Token A generated → sent to email → Token B generated → A invalidated
Race:   Token A generated ─┐ both tokens valid simultaneously
        Token B generated ─┘ → use either to reset password

Exploit:
1. Capture: POST /reset-password {email: "victim@target.com"}
2. Send 10 parallel resets via Burp group
3. Intercept ALL emails (if you own the address) or use Burp Collaborator for timing
4. Attempt both/all tokens before they expire
5. If multiple tokens work → window to reset before victim notices
```

### Pattern C: Double-Action on Same Resource

```
Race: simultaneously send "verify email" + "change email" requests

POST /verify-email {token: "TOKEN_FOR_A@ATTACKER.COM"} ─┐ race
POST /change-email {email: "B@ATTACKER.COM"}            ─┘

Expected: verify confirms old email, then email changes to B
Race result: email changes to B, but token for A is used to verify B
→ Account with B@attacker.com becomes verified without receiving B's verification email
→ Useful for verifying emails the attacker doesn't control
```

### Pattern D: OTP/2FA Bypass

```
Real-world case (eCommerce ATO writeup, 2024):
- Login flow uses flowId to associate OTP with account
- Attacker intercepts flowId after entering victim's email
- Race: simultaneously:
  POST /verify-otp {flowId: VICTIM_FLOW_ID, otp: OTP_SENT_TO_ATTACKER} ─┐
  POST /change-email {flowId: VICTIM_FLOW_ID, email: ATTACKER_EMAIL}    ─┘
- If race wins: OTP sent to attacker's new email, attacker receives it
- Use OTP → login as victim

Testing:
1. Start login as victim (get flowId)
2. Trigger OTP send
3. Race: verify-otp (with wrong code first to identify endpoint) +
         change-associated-email simultaneously
4. Monitor attacker email inbox for unexpected OTP
```

---

## 3. Multi-Endpoint Race Conditions

### Pattern A: Payment + Cart Manipulation

```
Vulnerability: payment processing races against cart modification

Normal flow:
  1. Add item A ($100) to cart
  2. Checkout → payment charged for $100
  3. Order confirmed for item A

Race attack:
  1. Add expensive item A ($1000) to cart
  2. Initiate checkout (payment check begins → will charge $1000)
  3. SIMULTANEOUSLY: change cart to cheap item B ($1)
  4. Race window: payment was checked for $1000, cart updated to $1 before order confirmation
  5. Result: charged $1, received item A ($1000)

Burp setup:
  Tab 1: POST /checkout/confirm {cartId: X}
  Tab 2: POST /cart/update {items: [{id: cheap_item, qty: 1}]}
  → Send both simultaneously after checkout initiated
```

### Pattern B: Subscription Check + Feature Access

```
Attack: hit "use premium feature" while simultaneously downgrading subscription

POST /subscription/downgrade ─┐ race
POST /feature/premium/use    ─┘

If race wins: feature used despite downgrade completing first
Scale: repeat to use premium features on free tier
```

### Pattern C: Invitation + Role Escalation

```
Multi-step invitation flow:
  1. Admin invites user with role="viewer"
  2. User accepts invitation → account created with role viewer
  3. Admin separately assigns elevated role

Race: accept invitation + request role upgrade simultaneously
  POST /invite/accept {token: INVITE_TOKEN}         ─┐ race
  POST /user/update-role {role: "admin"}             ─┘

If role update races before account fully initialized:
→ Account created with admin role despite only viewer invitation
```

### Pattern D: MFA Setup + Protected Resource Access

```
MFA enrollment window race:
  1. User starts MFA setup (MFA "pending" state — not yet enforced)
  2. During "pending" state, access protected resource

POST /mfa/setup/begin ─┐ race
GET /admin/dashboard  ─┘

In pending state, some implementations don't enforce MFA requirement yet
→ Access admin dashboard while MFA check is momentarily bypassed
```

### Pattern E: Transfer + Balance Check (Fintech)

```
Wallet: $100 balance
  
Normal:
  Transfer 1: check $100 ≥ $100 → transfer → update balance to $0
  Transfer 2: check $0 ≥ $100 → DENIED

Race:
  Transfer 1: check $100 ≥ $100 ─┐ both checks pass simultaneously
  Transfer 2: check $100 ≥ $100 ─┘ → both transfers execute → $200 extracted, balance goes negative

Real-world impact: CVE-2024-58248 (nopCommerce gift card), multiple fintech bounties $5K-$20K
```

---

## 4. Partial Construction Attacks

### What It Is

Objects (users, API keys, sessions) are often created in MULTIPLE database writes.
Between write 1 and write 2, the object exists in an incomplete/vulnerable state.

```
Vulnerable registration pattern:
  Write 1: INSERT user (username, password_hash)    ← object exists with no security attributes!
  [RACE WINDOW: user exists, email_verified=NULL, mfa=NULL, role=NULL]
  Write 2: INSERT user_attributes (email, role='user', mfa_required=true)

Attack: interact with user account during WRITE 1 → WRITE 2 window
```

### Pattern A: Registration Race

```
Attack flow:
1. Send registration request
2. Simultaneously poll the login endpoint with new credentials

POST /register {username: "newuser", password: "pass123"} ─┐
POST /login {username: "newuser", password: "pass123"}    ─┘

In the window between writes:
→ Login may succeed but return account with NULL role
→ NULL role sometimes interpreted as admin by broken logic
→ No MFA enforced (not written yet)
→ Email not verified (not written yet) — but account is accessible

GitLab bug class: token to confirm account was NULL during window
→ Any token (including empty string) worked for verification
```

### Pattern B: API Key Generation Race

```
Attack flow:
1. Trigger API key generation
2. Race to retrieve key before permissions are written

POST /api-keys/generate ─┐ race (key created, permissions not set yet)
GET /api-keys/latest    ─┘ (retrieve key before permissions attached)

During creation window:
→ Key may exist with NULL permissions (no restrictions applied)
→ Key may have elevated default permissions before reduction
```

### Pattern C: Session Creation with Incomplete Auth

```
Multi-step authentication:
  Step 1: POST /login/password {correct password} → creates partial session
  Step 2: POST /login/mfa {otp_code} → completes session, grants full access

Partial session race:
  Complete step 1 → partial session token issued
  Race: use partial session token to access protected resources
        WHILE simultaneously submitting MFA step

  In some implementations:
  → Partial session has implicit access before MFA step completes
  → MFA check happens AFTER session is created (TOCTOU)
```

### Pattern D: Password Reset + Login Race

```
Reset flow:
  1. POST /reset/request {email: X} → generates reset token (token stored with timestamp)
  2. POST /reset/complete {token: T, newPassword: P} → updates password → invalidates token

Race:
  POST /reset/complete {token: T, newPassword: "hacked"} ─┐ two completions
  POST /reset/complete {token: T, newPassword: "hacked"} ─┘ simultaneously

If token invalidation is not atomic:
→ Both completions process before either invalidates the token
→ Password is set (no practical impact from double-set in this case)

More interesting variant:
  POST /reset/complete {token: T, newPassword: "hacked"} ─┐
  POST /login {email: X, password: OLD_PASSWORD}         ─┘

If reset + login race:
→ Login may succeed using old password during reset window
→ Indicates reset doesn't properly invalidate sessions
```

---

## 5. Time-Sensitive / Hidden State Races

### Pattern: Concurrent Request State Confusion

```
When the same session is used by parallel requests in a stateful app:
Thread 1: GET /checkout → server sets session["cart"] = cart_A
Thread 2: GET /checkout → server sets session["cart"] = cart_B
Thread 1: POST /pay → pays for whatever is in session["cart"] (may be cart_B!)

Attack: manipulate timing so payment uses wrong cart context
```

### Pattern: Cache Poisoning via Race

```
Read-through cache:
  1. Cache MISS → reads from DB → stores in cache
  2. Cache HIT → returns cached value

Race: two writes race to populate cache
  Thread 1: write user_role=admin → cache populates with admin
  Thread 2: write user_role=user  → cache should overwrite
  
  If Thread 2 cache write is lost:
  → user_role=admin stuck in cache until TTL
  → Privilege escalation for cache TTL duration
```

### Pattern: Distributed System State Race

```
Microservice A: handles payment → marks order as paid
Microservice B: handles fulfillment → checks if order is paid

Race window: A marks paid, B checks paid before A's update propagates
→ B sends product before payment is confirmed
→ With quick chargeback: product received, payment reversed
```

---

## 6. Attack Checklist

```
Limit Overrun:
[ ] Coupon/promo code — parallel redemption
[ ] Gift card — parallel balance deduction
[ ] API credits — parallel consumption
[ ] Vote/rate limit — parallel submission
[ ] Daily reward — parallel claim

Single Endpoint:
[ ] Email change — token misrouting (2 parallel changes)
[ ] Password reset — parallel reset tokens
[ ] OTP verification — parallel verify + email change
[ ] Invitation — parallel accepts

Multi-Endpoint:
[ ] Payment + cart update simultaneously
[ ] Subscription downgrade + feature use
[ ] MFA setup + protected resource access
[ ] Balance check + two parallel transfers

Partial Construction:
[ ] Registration + immediate login race
[ ] API key generate + retrieve before permissions
[ ] Multi-step auth: partial session access
[ ] Password reset token + concurrent login
```
