# IAM — Thử thách: Security, Scale & Production Edge Cases

> OAuth2/OIDC flow nhìn đơn giản trên diagram. Nhưng production có: brute force, token theft, tenant data leak, refresh token abuse, session management cross-device, và scale cho nhiều tenant. File này đi sâu vào từng thử thách.

---

## Mục lục

1. [Thử thách 1: Brute Force & Account Lockout](#1-thử-thách-1-brute-force--account-lockout)
2. [Thử thách 2: Token Theft — Access token bị đánh cắp](#2-thử-thách-2-token-theft--access-token-bị-đánh-cắp)
3. [Thử thách 3: Refresh Token Abuse — Rotation & Family Detection](#3-thử-thách-3-refresh-token-abuse--rotation--family-detection)
4. [Thử thách 4: Cross-Tenant Data Leak — Bài toán nguy hiểm nhất](#4-thử-thách-4-cross-tenant-data-leak--bài-toán-nguy-hiểm-nhất)
5. [Thử thách 5: Session Management — Logout, Multi-Device](#5-thử-thách-5-session-management--logout-multi-device)
6. [Thử thách 6: Token Revocation — Stateless nhưng cần revoke](#6-thử-thách-6-token-revocation--stateless-nhưng-cần-revoke)
7. [Thử thách 7: Rate Limiting & DDoS Protection](#7-thử-thách-7-rate-limiting--ddos-protection)
8. [Thử thách 8: Secret Management — Lưu credentials an toàn](#8-thử-thách-8-secret-management--lưu-credentials-an-toàn)
9. [Thử thách 9: Audit & Compliance — Ai login lúc nào](#9-thử-thách-9-audit--compliance--ai-login-lúc-nào)
10. [Thử thách 10: Scale — Nhiều tenant, nhiều user, nhiều token](#10-thử-thách-10-scale--nhiều-tenant-nhiều-user-nhiều-token)
11. [Interview Quick Reference](#11-interview-quick-reference)

---

## 1. Thử thách 1: Brute Force & Account Lockout

### Bài toán

```
Attacker thử password liên tục:
  POST /token { email: "admin@acme.com", password: "123456" }
  POST /token { email: "admin@acme.com", password: "password" }
  POST /token { email: "admin@acme.com", password: "admin123" }
  ... 10,000 attempts

Nếu không có protection:
  → Attacker thử dictionary attack → có thể đoán được password
  → Đặc biệt nguy hiểm cho admin accounts
```

### Giải pháp: Multi-Layer Protection

```
Layer 1: Rate limiting (IP-based)
  → Max 10 login attempts per IP per phút
  → Quá → 429 Too Many Requests + exponential backoff
  → Chặn attack từ 1 IP

Layer 2: Account lockout (user-based)
  → 5 failed attempts → lock account 15 phút
  → 10 failed attempts → lock account 1 giờ
  → 20 failed attempts → lock account + alert admin
  → Chặn attack dù từ nhiều IPs (distributed brute force)

Layer 3: Progressive delay
  → Attempt 1: respond ngay
  → Attempt 2: delay 1 giây
  → Attempt 3: delay 2 giây
  → Attempt N: delay min(2^(N-1), 30) giây
  → Slow down attacker mà không block user hợp lệ

Layer 4: CAPTCHA (after 3 failed attempts)
  → Hiện CAPTCHA → chặn automated tools
  → Chỉ hiện sau N failed attempts (không ảnh hưởng UX bình thường)
```

### Account Lockout — Cẩn thận DoS

```
Vấn đề: Attacker biết email admin → spam wrong password → lock admin
  → Denial of Service cho user hợp lệ

Giải pháp:
  - Lockout có thời hạn (15 phút → auto-unlock)
  - Admin có thể unlock qua channel khác (email link, support)
  - Lockout chỉ block password login, SSO vẫn hoạt động
  - Alert admin khi account bị lock → investigate
```

### Implementation

```sql
-- Mỗi login attempt:
UPDATE users
SET failed_login_count = failed_login_count + 1,
    locked_until = CASE
      WHEN failed_login_count >= 5
      THEN NOW() + INTERVAL '15 minutes'
      ELSE locked_until
    END
WHERE id = :userId;

-- Check trước khi verify password:
SELECT * FROM users
WHERE email = :email
  AND tenant_id = :tenantId
  AND (locked_until IS NULL OR locked_until < NOW());
-- locked_until < NOW() → auto-unlock khi hết thời hạn

-- Login thành công → reset counter:
UPDATE users
SET failed_login_count = 0, locked_until = NULL, last_login_at = NOW()
WHERE id = :userId;
```

---

## 2. Thử thách 2: Token Theft — Access token bị đánh cắp

### Bài toán

```
Access token (JWT) bị leak:
  - XSS attack → steal token từ JavaScript memory
  - Man-in-the-middle (nếu không HTTPS)
  - Log file chứa token
  - Browser extension malicious → đọc memory

Attacker có access token → gọi API → giả danh user
→ JWT stateless → server không biết token bị steal
→ Token valid 15 phút → attacker có 15 phút access
```

### Giải pháp: Minimize exposure + Short lifetime

```
1. Short-lived access token: 15 phút
   → Window of exposure: tối đa 15 phút
   → Attacker phải liên tục steal token mới

2. Token storage (SPA):
   → access_token: IN-MEMORY ONLY (JavaScript variable)
     ✗ KHÔNG lưu localStorage (persistent, XSS access)
     ✗ KHÔNG lưu sessionStorage (XSS access)
     ✓ Memory: mất khi refresh page → attacker khó steal persist
   → refresh_token: HttpOnly Secure cookie
     ✗ JavaScript KHÔNG đọc được (HttpOnly)
     ✓ Browser tự gửi kèm request (Secure = HTTPS only)

3. HTTPS everywhere: TLS encrypt → chống MITM

4. Content Security Policy (CSP): chống XSS
   → Restrict inline script, external script sources
   → Giảm risk XSS → giảm risk token theft

5. Token binding (advanced — future):
   → Token bound to client fingerprint
   → Stolen token + different device → reject
```

### Access Token in Memory — Refresh page mất token?

```
User refresh page (F5):
  → access_token ở memory → mất
  → refresh_token ở HttpOnly cookie → còn
  → Silent refresh: iframe POST /token với refresh_token cookie
  → IAM issue access_token mới → lưu memory → user tiếp tục

Flow:
  Page load → check memory: access_token?
    Không → silent refresh (background iframe/fetch)
    → POST /token { grant_type: refresh_token } (cookie tự gửi)
    → Nhận access_token mới → lưu memory → done
    → User không thấy login page (seamless)
```

---

## 3. Thử thách 3: Refresh Token Abuse — Rotation & Family Detection

### Bài toán

```
Refresh token sống 7 ngày. Nếu bị đánh cắp:
  → Attacker dùng refresh token → lấy access token mới → access 7 ngày
  → Nguy hiểm hơn access token theft nhiều
```

### Giải pháp: Refresh Token Rotation + Family Detection

```
Rotation:
  Mỗi lần dùng refresh_token → issue CẶP MỚI (access + refresh)
  → Refresh token cũ bị revoke
  → Mỗi refresh token CHỈ DÙNG 1 LẦN

Family Detection:
  Mỗi "chuỗi" refresh tokens thuộc 1 family (family_id)
  Login → refresh_token_1 (family: F1)
  Refresh → refresh_token_2 (family: F1) + revoke token_1
  Refresh → refresh_token_3 (family: F1) + revoke token_2

  Nếu attacker dùng token_1 (đã revoke):
    → IAM detect: token_1 thuộc family F1, đã revoke
    → REUSE DETECTED → revoke TOÀN BỘ family F1
    → User và attacker đều bị kick → user phải re-login
    → Alert: possible token theft
```

```
Timeline:
  10:00 - User login → refresh_token_1 (family F1)
  10:15 - User refresh → refresh_token_2 (F1), revoke token_1
  10:16 - Attacker steal token_1 (từ trước khi revoke)
  10:30 - User refresh → refresh_token_3 (F1), revoke token_2
  10:31 - Attacker dùng token_1 → REVOKED → reuse detected!
          → Revoke TOÀN BỘ F1 (token_2, token_3)
          → User bị kick → phải login lại (nhưng an toàn)
          → Alert security team
```

### Implementation

```sql
-- Refresh token exchange:
BEGIN;
  -- 1. Verify token
  SELECT * FROM refresh_tokens
  WHERE token_hash = SHA256(:token)
    AND expires_at > NOW();

  -- 2. Check revoked
  IF token.revoked = TRUE THEN
    -- REUSE DETECTED! Revoke entire family
    UPDATE refresh_tokens SET revoked = TRUE
    WHERE family_id = token.family_id;
    RAISE 'token_reuse_detected';
  END IF;

  -- 3. Revoke current token
  UPDATE refresh_tokens SET revoked = TRUE WHERE id = token.id;

  -- 4. Issue new token (same family)
  INSERT INTO refresh_tokens (user_id, client_id, token_hash, family_id, expires_at)
  VALUES (:userId, :clientId, SHA256(:newToken), token.family_id, NOW() + INTERVAL '7 days');
COMMIT;
```

---

## 4. Thử thách 4: Cross-Tenant Data Leak — Bài toán nguy hiểm nhất

### Bài toán

```
User của Acme Corp → bằng cách nào đó → thấy data của Beta Inc
→ Data breach → mất hợp đồng CẢ 2 tenant → lawsuit → end of business

Cách xảy ra:
  1. Bug code: quên WHERE tenant_id = ? → query trả data mọi tenant
  2. Token craft: sửa tenant_id trong JWT → access tenant khác
  3. URL manipulation: /api/tenants/beta/users (Acme user gọi Beta URL)
  4. Admin error: assign user nhầm tenant
```

### Giải pháp: Defense in Depth

```
Layer 1: JWT signature (RS256)
  → Token signed bằng private key → sửa tenant_id → signature invalid
  → Service verify signature → reject tampered token
  → Chặn token craft

Layer 2: Row Level Security (PostgreSQL)
  → SET app.current_tenant_id = token.tenant_id (từ middleware)
  → RLS policy: WHERE tenant_id = current_setting('app.current_tenant_id')
  → Dù code quên filter → RLS chặn ở DB level
  → Safety net cuối cùng

Layer 3: Application middleware
  → Extract tenant_id từ JWT → set context
  → Mọi repository/query tự thêm tenant_id filter
  → URL path /tenants/:tenantId → verify tenantId == token.tenant_id
  → Mismatch → 403 Forbidden

Layer 4: Integration test
  → Test: user Acme gọi API → chỉ thấy data Acme
  → Test: user Acme gọi API với Beta tenant_id → 403
  → Test: SQL query không có tenant_id → RLS block
  → CI/CD chạy mỗi commit

Layer 5: Audit log
  → Log mọi cross-tenant access attempt
  → Alert nếu user A cố access tenant B
  → Monthly review
```

### Tại sao RLS là safety net quan trọng nhất?

```
Code có bug → quên tenant_id filter:
  SELECT * FROM orders WHERE created_at > '2026-01-01';
  → Trả orders CỦA MỌI TENANT → data leak

Có RLS:
  SET app.current_tenant_id = 'acme';  (middleware set từ JWT)
  SELECT * FROM orders WHERE created_at > '2026-01-01';
  → PostgreSQL tự thêm: AND tenant_id = 'acme'
  → Chỉ trả orders của Acme → safe

RLS = defense at data layer → bất kể application bug
```

---

## 5. Thử thách 5: Session Management — Logout, Multi-Device

### Single Logout (SLO)

```
User login Module A + Module B (SSO).
User logout Module A → Module B vẫn đăng nhập?

Approach 1: Chỉ logout module hiện tại (simple)
  → Module A: xóa access_token + refresh_token
  → Module B: vẫn có token → vẫn đăng nhập
  → IAM session: vẫn active
  → Nhược: user tưởng đã logout hoàn toàn nhưng Module B vẫn active

Approach 2: Revoke IAM session + all tokens (recommend)
  → Module A → POST /logout → IAM
  → IAM: delete session cookie
  → IAM: revoke TẤT CẢ refresh tokens của user
  → Module B: access token expire (15 phút) → refresh fail → logged out
  → Gap: Module B vẫn active tối đa 15 phút (access token lifetime)

Approach 3: Backchannel logout (OIDC standard)
  → IAM → POST /backchannel-logout → Module A, Module B
  → Modules xóa session ngay
  → Phức tạp, cần modules implement logout endpoint
  → Cho phase sau nếu cần instant logout
```

### Multi-Device

```
User login laptop + mobile:
  → 2 refresh token families (F1 = laptop, F2 = mobile)
  → Logout laptop → revoke F1 → mobile vẫn đăng nhập
  → "Logout all devices" → revoke F1 + F2 → mọi device bị kick

Admin "force logout user" (security incident):
  → Revoke TẤT CẢ refresh tokens của user
  → Tất cả devices: access token expire → refresh fail → logged out
  → Thời gian chờ: tối đa 15 phút (access token lifetime)
  → Nếu cần instant: thêm token blacklist (Redis) — xem Thử thách 6
```

---

## 6. Thử thách 6: Token Revocation — Stateless nhưng cần revoke

### Bài toán

```
JWT access token là stateless:
  → Server verify bằng signature + expiry → KHÔNG query DB
  → Nhanh, scalable
  → Nhưng: KHÔNG revoke được trước khi expire

Case cần revoke ngay:
  - Admin deactivate user → user vẫn access 15 phút
  - Security incident → token bị leak → 15 phút exposure
  - User đổi password → old token vẫn valid 15 phút
```

### Giải pháp: Trade-off giữa stateless và security

```
Option 1: Chấp nhận 15 phút gap (recommend cho hầu hết case)
  → Access token 15 phút → expire tự nhiên
  → Revoke refresh token → không issue access token mới
  → Gap 15 phút: chấp nhận được cho 99% cases
  → Đơn giản, performant

Option 2: Token blacklist (Redis) — cho critical cases
  → Khi cần revoke: thêm token JTI vào Redis blacklist
  → TTL = token remaining lifetime (tối đa 15 phút)
  → Services check blacklist mỗi request
  → Instant revoke nhưng thêm Redis call mỗi request

  Trade-off:
    Redis blacklist size: nhỏ (chỉ revoked tokens, TTL auto-cleanup)
    Latency: +1-2ms per request (Redis lookup)
    Complexity: services cần check blacklist

Option 3: Short-lived tokens (5 phút thay 15 phút)
  → Giảm exposure window → 5 phút
  → Nhưng: refresh thường xuyên hơn → nhiều /token calls hơn
  → Trade-off: security vs performance

Recommend:
  - Default: Option 1 (15 phút gap, revoke refresh token)
  - Admin deactivate, password change: Option 1 đủ
  - Security incident (token theft confirmed): Option 2 (blacklist)
```

---

## 7. Thử thách 7: Rate Limiting & DDoS Protection

### Rate Limiting Strategy

```
Endpoint-based limits:

| Endpoint | Limit | Tại sao |
|----------|-------|---------|
| POST /token (login) | 10/phút per IP | Chống brute force |
| POST /token (refresh) | 20/phút per user | Chống refresh abuse |
| GET /authorize | 30/phút per IP | Chống redirect flood |
| GET /userinfo | 60/phút per token | Normal usage |
| POST /users (admin) | 100/phút per tenant | Bulk create protection |

Tenant-based limits (fair usage):
  - Mỗi tenant: max 1000 login/phút (dù có 5000 users)
  - Prevent 1 tenant DoS cả platform
```

### Implementation

```
Redis-based sliding window:

Key: ratelimit:{endpoint}:{identifier}
  identifier = IP (login) | user_id (refresh) | tenant_id (tenant limit)

Lua script (atomic check + increment):
  local count = redis.call('INCR', key)
  if count == 1 then redis.call('EXPIRE', key, window) end
  if count > limit then return 0 end  -- rejected
  return 1  -- allowed
```

---

## 8. Thử thách 8: Secret Management — Lưu credentials an toàn

### Những secret cần bảo vệ

```
1. User passwords → Argon2id hash (KHÔNG plain text, KHÔNG MD5/SHA1)
2. OAuth2 client secrets → encrypted at rest (AES-256)
3. JWT private key → HSM hoặc vault (KHÔNG trong code/env)
4. SSO IdP credentials (client_secret cho OIDC) → encrypted in DB
5. Database credentials → environment variable hoặc vault
6. API keys (internal service-to-service) → rotate periodically
```

### Password Hashing

```
Recommend: Argon2id (winner Password Hashing Competition)
Fallback: bcrypt (nếu Argon2 không available)

KHÔNG DÙNG: MD5, SHA1, SHA256 (không phải password hash function)

Argon2id config:
  - memory: 64MB
  - iterations: 3
  - parallelism: 4
  - salt: 16 bytes random per password
  → ~100ms per hash → chậm đủ để chống brute force
  → Nhưng 1 login = 1 hash → user không cảm nhận

Tại sao không bcrypt?
  - bcrypt: đủ tốt, proven, widely supported
  - Argon2id: tốt hơn (memory-hard → resist GPU/ASIC attack)
  - Cả 2 đều chấp nhận được cho production
```

---

## 9. Thử thách 9: Audit & Compliance — Ai login lúc nào

### Audit Events

```
Mọi action quan trọng phải log:

| Event | Data | Tại sao |
|-------|------|---------|
| LOGIN_SUCCESS | user, tenant, IP, device, method (password/SSO) | Track access |
| LOGIN_FAILED | email, tenant, IP, reason | Detect brute force |
| LOGOUT | user, tenant, method (user/admin/session_expire) | Track session |
| TOKEN_REFRESH | user, tenant, client | Track active sessions |
| PASSWORD_CHANGE | user, tenant, method (self/admin_reset) | Security audit |
| ACCOUNT_LOCKED | user, tenant, reason, lock_duration | Security alert |
| ACCOUNT_UNLOCKED | user, tenant, method (auto/admin) | Track lockout |
| USER_CREATED | user, tenant, created_by, method (manual/JIT/SCIM) | Provisioning audit |
| USER_DEACTIVATED | user, tenant, deactivated_by | Offboarding audit |
| SSO_CONFIG_CHANGED | tenant, changed_by, changes | Config audit |
| ADMIN_ACTION | tenant, admin, action, target_user | Admin audit |
```

### Tại sao Audit quan trọng cho B2B?

```
Doanh nghiệp khách hàng cần:
  1. "Ai đã login vào hệ thống lúc 3 giờ sáng?" → login_events
  2. "Nhân viên A nghỉ việc tháng trước, có login sau đó không?" → login_events
  3. "Có ai brute force account admin không?" → failed login events
  4. Compliance report hàng quý → export audit log
  5. Pháp lý: tranh chấp → audit trail là bằng chứng

Retention:
  - login_events: 2 năm (compliance)
  - admin_actions: 5 năm (legal requirement tùy ngành)
```

---

## 10. Thử thách 10: Scale — Nhiều tenant, nhiều user, nhiều token

### Numbers

```
100 tenants × 500 users/tenant = 50,000 users
Peak: 20% users login cùng lúc = 10,000 login/giờ ≈ 3 login/giây
Token refresh: 10,000 active users × refresh mỗi 15 phút = 11 refresh/giây
JWKS endpoint: services cache → ~10 requests/phút (sau cold start)

→ Không phải high-scale problem. Single PostgreSQL + Redis đủ.
```

### Bottleneck tiềm năng

```
1. Password hashing (Argon2id ~100ms/hash):
   3 login/giây × 100ms = 300ms CPU/giây → 1 core đủ
   Peak 30 login/giây → 3 cores → vẫn OK

2. Refresh token DB lookup:
   11 queries/giây → PostgreSQL dễ handle
   Nếu cần: cache active refresh tokens trong Redis

3. JWKS endpoint:
   Services cache public keys → gần 0 load
   Cache-Control: max-age=3600 → refresh mỗi giờ

→ IAM scale concern thật sự: nếu 10,000 tenants × 1,000 users
   → 10M users, 50,000 login/giây → cần horizontal scale
   → Nhưng đó là problem cho future, không phải ngày 1
```

### Scale strategy khi cần

```
Theo thứ tự, chỉ thêm khi cần:

1. Redis cache cho user lookup + refresh token
   → Giảm DB queries cho hot path

2. Read replica PostgreSQL
   → Login verify (read) → replica
   → Token issue (write) → primary

3. Multiple IAM instances (stateless)
   → Load balance
   → Session → Redis (shared)
   → JWT signing → shared private key (vault)

4. Shard by tenant_id (extreme scale)
   → Tenant A-M → DB shard 1
   → Tenant N-Z → DB shard 2
   → Khi 10,000+ tenants
```

---

## 11. Interview Quick Reference

### Elevator Pitch (challenges)

> "Thử thách lớn nhất IAM multi-tenant: cross-tenant data leak. Defense in depth: JWT signature chống token craft, RLS chống code bug quên tenant filter, middleware verify tenant context, integration test mỗi commit. RLS là safety net cuối cùng — dù application bug, data vẫn isolate ở DB level."

### "Token bị steal?"

> "Access token 15 phút, lưu in-memory only (không localStorage). Refresh token HttpOnly Secure cookie — JS không đọc được. Refresh token rotation: mỗi lần dùng → issue mới + revoke cũ. Family detection: reuse token cũ → revoke toàn family → force re-login + alert."

### "Revoke token ngay được không?"

> "Access token stateless → không revoke trước expire (15 phút). Trade-off chấp nhận được cho 99% cases. Critical case (token theft): Redis blacklist — thêm JTI, TTL = remaining lifetime, services check mỗi request (+1-2ms). Revoke refresh token → không issue access mới."

### "Brute force?"

> "4 layer: rate limit IP (10/phút), account lockout (5 fails → lock 15 phút), progressive delay (exponential), CAPTCHA sau 3 fails. Lockout có thời hạn → auto-unlock. Lockout chỉ block password, SSO vẫn hoạt động."

### "Cross-tenant data leak?"

> "5 layer: JWT signature (chống token craft), RLS PostgreSQL (chống code bug), middleware tenant verify, integration test, audit log. RLS quan trọng nhất — dù code quên WHERE tenant_id, PostgreSQL tự filter. Defense at data layer."

### Bảng quyết định tổng hợp

| Quyết định | Chọn | Tại sao | Nếu bị challenge |
|-----------|------|---------|------------------|
| Access token lifetime | 15 phút | Balance security (short) vs UX (không refresh quá thường) | "5 phút?" → Nhiều refresh call hơn, 15 phút là industry standard |
| Token storage (SPA) | Memory (access) + HttpOnly cookie (refresh) | XSS không steal được refresh token | "Page refresh?" → Silent refresh via cookie |
| Refresh rotation | Rotate + family detection | Stolen token → reuse → detect → revoke all | "Phức tạp?" → Critical security, worth complexity |
| Token revocation | Revoke refresh + accept 15m gap | Stateless fast path, revocable refresh | "Instant revoke?" → Redis blacklist cho critical cases |
| Tenant isolation | RLS + middleware + JWT + test | Defense in depth, RLS = safety net | "RLS overhead?" → Minimal, PostgreSQL optimize well |
| Password hash | Argon2id | Memory-hard, resist GPU attack | "bcrypt?" → Cũng OK, Argon2id tốt hơn |
| Brute force | Rate limit + lockout + delay + CAPTCHA | 4 layers chặn 4 loại attack khác nhau | "Lockout = DoS?" → Auto-unlock + SSO bypass |
| Scale | Single DB + Redis đủ cho 50K users | Premature optimization is evil | "10M users?" → Read replica + shard, nhưng không phải ngày 1 |
