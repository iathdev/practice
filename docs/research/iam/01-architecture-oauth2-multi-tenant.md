# IAM — Kiến trúc, OAuth2 & Multi-Tenant Authentication

> Thiết kế hệ thống Identity & Access Management cho dự án GAM — B2B platform cung cấp giải pháp cho doanh nghiệp. Mỗi doanh nghiệp (tenant) có users riêng, login tập trung qua IAM, dùng OAuth2/OIDC làm core protocol.

---

## Mục lục

1. [Bối cảnh & Yêu cầu](#1-bối-cảnh--yêu-cầu)
2. [Kiến trúc tổng thể — IAM là Centralized Auth](#2-kiến-trúc-tổng-thể--iam-là-centralized-auth)
3. [OAuth2 & OpenID Connect — Core Protocol](#3-oauth2--openid-connect--core-protocol)
4. [Multi-Tenant Model — Tenant Isolation](#4-multi-tenant-model--tenant-isolation)
5. [Token Design — JWT Structure](#5-token-design--jwt-structure)
6. [Login Flow — End-to-End](#6-login-flow--end-to-end)
7. [User Management — GAM quản lý user](#7-user-management--gam-quản-lý-user)
8. [Data Model](#8-data-model)
9. [Interview Quick Reference](#9-interview-quick-reference)

---

## 1. Bối cảnh & Yêu cầu

### Dự án GAM

- **Mô hình**: B2B SaaS — GAM cung cấp giải pháp cho nhiều doanh nghiệp
- **IAM role**: Centralized authentication — mọi user login qua IAM, không qua từng service riêng
- **User management**: GAM quản lý user (không phải doanh nghiệp tự quản lý identity provider)

### Yêu cầu

| Tiêu chí | Yêu cầu |
|----------|---------|
| Multi-tenant | Mỗi doanh nghiệp là 1 tenant, user thuộc tenant, data isolate |
| Centralized login | 1 nơi login duy nhất cho toàn bộ GAM ecosystem |
| Protocol | OAuth2 + OpenID Connect (industry standard) |
| SSO capable | Có thể tích hợp SSO với hệ thống doanh nghiệp (future) |
| Token-based | Stateless authentication cho microservices |
| Scalable | Nhiều tenant, mỗi tenant nhiều user |

### Tại sao cần IAM riêng?

```
Không có IAM (mỗi service tự authen):
  Service A: login form riêng, user table riêng
  Service B: login form riêng, user table riêng
  → User phải nhớ nhiều tài khoản
  → Mỗi service implement authen → duplicate code, inconsistent security
  → Thêm service mới → lại build login từ đầu
  → Doanh nghiệp dùng 5 module → 5 lần đăng nhập → UX tệ

Có IAM (centralized):
  IAM: 1 nơi login, 1 user identity
  Service A, B, C: nhận token từ IAM → verify → trust
  → User đăng nhập 1 lần → dùng tất cả services (SSO)
  → Security tập trung: MFA, password policy, lockout — 1 nơi
  → Thêm service mới → chỉ cần integrate OAuth2 client → xong
  → Doanh nghiệp: 1 lần login → dùng mọi module
```

---

## 2. Kiến trúc tổng thể — IAM là Centralized Auth

```
┌──────────────────────────────────────────────────────────────────────┐
│                        GAM ECOSYSTEM                                  │
│                                                                       │
│  ┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐        │
│  │  Module A  │  │  Module B  │  │  Module C  │  │  Admin    │        │
│  │  (SPA)     │  │  (SPA)     │  │  (Mobile)  │  │  Portal   │        │
│  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘        │
│        │               │               │               │              │
│        │          OAuth2 / OIDC         │               │              │
│        └───────────────┼───────────────┘               │              │
│                        │                                │              │
│                        ▼                                ▼              │
│                ┌──────────────────────────────────────────┐           │
│                │              IAM SERVICE                  │           │
│                │                                          │           │
│                │  ┌────────────┐  ┌─────────────────┐    │           │
│                │  │  Auth      │  │  User            │    │           │
│                │  │  Server    │  │  Management      │    │           │
│                │  │  (OAuth2)  │  │  (CRUD users)    │    │           │
│                │  └────────────┘  └─────────────────┘    │           │
│                │                                          │           │
│                │  ┌────────────┐  ┌─────────────────┐    │           │
│                │  │  Token     │  │  Tenant          │    │           │
│                │  │  Service   │  │  Management      │    │           │
│                │  │  (JWT)     │  │  (multi-tenant)  │    │           │
│                │  └────────────┘  └─────────────────┘    │           │
│                │                                          │           │
│                └──────────────────┬───────────────────────┘           │
│                                   │                                   │
│                          ┌────────▼────────┐                         │
│                          │   PostgreSQL     │                         │
│                          │   (users,        │                         │
│                          │    tenants,      │                         │
│                          │    tokens)       │                         │
│                          └─────────────────┘                         │
└──────────────────────────────────────────────────────────────────────┘

External (SSO — future):
  ┌──────────────┐
  │ Enterprise   │
  │ IdP (Azure   │ ──── SAML/OIDC ────▶ IAM Service
  │ AD, Google)  │
  └──────────────┘
```

### Nguyên tắc thiết kế

| Nguyên tắc | Giải thích |
|-----------|-----------|
| **IAM là gateway duy nhất cho identity** | Không service nào tự authenticate. Mọi authen đi qua IAM |
| **Token-based, stateless** | IAM issue JWT. Services verify token locally (public key). Không query IAM mỗi request |
| **Tenant isolation** | User thuộc tenant. Token chứa tenant_id. Service filter data theo tenant |
| **Protocol standard** | OAuth2 + OIDC. Không tự chế protocol. Client libraries có sẵn mọi ngôn ngữ |
| **IAM không biết business logic** | IAM chỉ biết: ai là ai (identity), thuộc tenant nào. Không biết Module A làm gì |

---

## 3. OAuth2 & OpenID Connect — Core Protocol

### OAuth2 vs OIDC — Phân biệt

```
OAuth2:
  → Authorization framework (ai được làm gì)
  → Sinh access_token → dùng để truy cập resource
  → KHÔNG định nghĩa cách truyền user info

OpenID Connect (OIDC):
  → Authentication layer TRÊN OAuth2 (ai là ai)
  → Thêm id_token (JWT chứa user identity)
  → Thêm /userinfo endpoint
  → Thêm discovery endpoint (/.well-known/openid-configuration)

GAM dùng: OAuth2 + OIDC
  → OAuth2 cho authorization (access_token)
  → OIDC cho authentication (id_token + user info)
```

### Grant Types — Khi nào dùng gì?

| Grant Type | Khi nào | Flow |
|-----------|---------|------|
| **Authorization Code + PKCE** | SPA, Mobile app (GAM modules) | User login → redirect → code → exchange token |
| **Client Credentials** | Service-to-service (backend) | Service gửi client_id + secret → nhận token |
| **Refresh Token** | Gia hạn access token | Gửi refresh_token → nhận access_token mới |
| ~~Password Grant~~ | ~~KHÔNG DÙNG~~ | ~~Deprecated, insecure~~ |
| ~~Implicit~~ | ~~KHÔNG DÙNG~~ | ~~Deprecated, token leak risk~~ |

### Authorization Code + PKCE (cho SPA/Mobile — flow chính)

```
Tại sao PKCE?
  SPA chạy trong browser → không giữ được client_secret
  PKCE (Proof Key for Code Exchange) thay thế secret:
    → Client tạo random code_verifier
    → Hash → code_challenge = SHA256(code_verifier)
    → Gửi code_challenge khi request auth code
    → Gửi code_verifier khi exchange token
    → Server verify: SHA256(code_verifier) == code_challenge
    → Attacker intercept auth code nhưng không có code_verifier → vô dụng
```

```
┌──────────┐                    ┌──────────────┐                ┌──────────┐
│  Client   │                    │  IAM Auth     │                │  IAM     │
│  (SPA)    │                    │  Server       │                │  Token   │
│           │                    │  (/authorize) │                │  (/token)│
└─────┬─────┘                    └──────┬────────┘                └────┬─────┘
      │                                 │                              │
      │ 1. Generate code_verifier       │                              │
      │    + code_challenge             │                              │
      │                                 │                              │
      │ 2. GET /authorize               │                              │
      │    ?response_type=code          │                              │
      │    &client_id=module_a          │                              │
      │    &redirect_uri=...            │                              │
      │    &code_challenge=xxx          │                              │
      │    &code_challenge_method=S256  │                              │
      │    &scope=openid profile        │                              │
      │    &state=random                │                              │
      │─────────────────────────────────▶                              │
      │                                 │                              │
      │         3. Login page           │                              │
      │◀─────────────────────────────────                              │
      │                                 │                              │
      │ 4. User nhập email/password     │                              │
      │─────────────────────────────────▶                              │
      │                                 │                              │
      │ 5. Redirect: /callback          │                              │
      │    ?code=AUTH_CODE&state=random  │                              │
      │◀─────────────────────────────────                              │
      │                                 │                              │
      │ 6. POST /token                  │                              │
      │    { grant_type: authorization_code,                           │
      │      code: AUTH_CODE,                                          │
      │      code_verifier: xxx }       │                              │
      │─────────────────────────────────────────────────────────────────▶
      │                                 │                              │
      │ 7. Response:                    │                              │
      │    { access_token, id_token,    │                              │
      │      refresh_token, expires_in }│                              │
      │◀────────────────────────────────────────────────────────────────
```

### Client Credentials (service-to-service)

```
Backend service cần gọi service khác:
  POST /token
  {
    "grant_type": "client_credentials",
    "client_id": "service_a",
    "client_secret": "secret_xxx",
    "scope": "service_b:read"
  }

  → access_token (KHÔNG có user context, chỉ service identity)
  → Dùng cho: cron job, background worker, internal API calls
```

---

## 4. Multi-Tenant Model — Tenant Isolation

### Tenant là gì trong GAM?

```
Tenant = 1 doanh nghiệp khách hàng

Ví dụ:
  Tenant "Acme Corp"   → 50 users, dùng Module A + B
  Tenant "Beta Inc"     → 200 users, dùng Module A + C
  Tenant "Gamma Ltd"    → 10 users, dùng Module B

Mỗi tenant:
  - Có users riêng (email unique TRONG tenant, không across tenants)
  - Data isolate (Acme không thấy data Beta)
  - Có thể có config riêng (branding, SSO, password policy)
```

### Tenant Identification — Làm sao biết user thuộc tenant nào?

```
Vấn đề: User truy cập login page → thuộc tenant nào?

3 cách identify tenant:

1. Subdomain-based (recommend cho B2B SaaS):
   acme.gam.com      → tenant = Acme Corp
   beta.gam.com      → tenant = Beta Inc
   → Dễ hiểu, tự nhiên cho B2B
   → Login page render branding của tenant
   → Wildcard DNS + SSL cert (*.gam.com)

2. Email domain-based:
   user@acme.com     → tự map sang tenant Acme Corp
   → Đơn giản cho user (chỉ nhập email)
   → Cần mapping table: email_domain → tenant_id
   → Vấn đề: 1 domain chỉ thuộc 1 tenant? Gmail/Yahoo users?

3. Tenant selector (login page hỏi):
   Login page → "Chọn tổ chức" → dropdown tenant
   → Đơn giản implement
   → UX kém hơn (thêm 1 bước)

Recommend: Subdomain (primary) + Email domain (fallback)
  → acme.gam.com: biết ngay tenant = Acme
  → gam.com/login: nhập email → lookup email domain → resolve tenant
```

### Tenant Isolation trong Database

```
Shared database, shared schema, tenant_id column (recommend cho GAM):

  ┌─────────────────────────────────────────┐
  │  PostgreSQL                              │
  │                                          │
  │  users table:                            │
  │    id | tenant_id | email | name | ...  │
  │    ───┼───────────┼───────┼──────┼───── │
  │    1  | acme      | a@..  | John | ...  │
  │    2  | beta      | b@..  | Jane | ...  │
  │                                          │
  │  Mọi query: WHERE tenant_id = :tenantId │
  └─────────────────────────────────────────┘

Tại sao shared database, không phải database-per-tenant?

| Strategy | Ưu | Nhược | Khi nào |
|----------|-----|-------|---------|
| Shared DB + tenant_id column | Đơn giản, ít ops overhead, cross-tenant query dễ | Tenant quên filter = data leak | < 1000 tenants (recommend) |
| Schema-per-tenant | Isolate tốt hơn, per-tenant backup | N schemas = N migrations | 10-100 tenants, compliance |
| Database-per-tenant | Isolate hoàn toàn | Ops nightmare, connection pool × N | Enterprise, strict compliance |

GAM: shared DB + tenant_id. Enforce bằng:
  - Row Level Security (RLS) PostgreSQL
  - Application middleware set tenant context từ JWT
  - Code review: mọi query PHẢI có tenant_id filter
```

### Row Level Security (RLS) — Safety net

```sql
-- Bật RLS
ALTER TABLE users ENABLE ROW LEVEL SECURITY;

-- Policy: user chỉ thấy data trong tenant của mình
CREATE POLICY tenant_isolation ON users
    USING (tenant_id = current_setting('app.current_tenant_id'));

-- Mỗi request, middleware set tenant context:
SET app.current_tenant_id = 'acme';
-- Mọi query tự động filter theo tenant → dù quên WHERE cũng safe
```

---

## 5. Token Design — JWT Structure

### Access Token (JWT)

```json
{
  "header": {
    "alg": "RS256",
    "typ": "JWT",
    "kid": "key-2026-01"
  },
  "payload": {
    "iss": "https://iam.gam.com",
    "sub": "user_uuid_123",
    "aud": ["module-a", "module-b"],
    "exp": 1712401800,
    "iat": 1712400000,
    "jti": "unique-token-id",

    "tenant_id": "acme",
    "email": "john@acme.com",
    "name": "John Doe"
  }
}
```

### Tại sao RS256 (asymmetric), không phải HS256 (symmetric)?

```
HS256 (HMAC — shared secret):
  IAM sign token bằng secret key
  Services verify token bằng CÙNG secret key
  → Mỗi service cần biết secret → secret leak risk
  → Service nào cũng có thể FORGE token (có secret mà)
  → Rotate key → update TẤT CẢ services cùng lúc

RS256 (RSA — public/private key):
  IAM sign token bằng private key (CHỈ IAM có)
  Services verify token bằng public key
  → Public key: ai cũng biết → không cần bảo mật
  → Service KHÔNG THỂ forge token (không có private key)
  → Rotate key: đổi private key, publish public key mới
  → Services tải public key từ JWKS endpoint

IAM publish public keys:
  GET https://iam.gam.com/.well-known/jwks.json
  → Services cache + auto-refresh
```

### Token Lifecycle

```
                  Access Token              Refresh Token
Lifetime:         15 phút                   7 ngày (hoặc theo tenant config)
Lưu ở đâu:       Memory (SPA)              HttpOnly Secure cookie
Gửi kèm request: Authorization: Bearer     Tự động gửi (cookie)
Revoke:           Không (stateless, chờ expire)  Có (server-side revoke)
Khi expire:       Dùng refresh token lấy mới    User phải login lại
```

### Token Refresh Flow

```
Access token expire (15 phút):
  1. Client gửi request → API trả 401
  2. Client gửi refresh token → POST /token { grant_type: refresh_token }
  3. IAM verify refresh token:
     - Token hợp lệ? (signature, expire)
     - Token bị revoke chưa? (check DB/Redis)
     - User còn active? Tenant còn active?
  4. OK → issue access_token mới + refresh_token mới (rotation)
  5. Fail → user phải login lại

Refresh Token Rotation:
  Mỗi lần dùng refresh_token → issue refresh_token MỚI + revoke cũ
  → Nếu attacker đánh cắp refresh_token cũ → dùng → conflict detected
  → Revoke TOÀN BỘ refresh tokens của user → force re-login
  → Giảm window of exposure
```

### JWKS Endpoint & Key Rotation

```
GET https://iam.gam.com/.well-known/jwks.json

{
  "keys": [
    {
      "kid": "key-2026-01",
      "kty": "RSA",
      "alg": "RS256",
      "use": "sig",
      "n": "...",
      "e": "AQAB"
    },
    {
      "kid": "key-2025-12",
      "kty": "RSA",
      "alg": "RS256",
      "use": "sig",
      "n": "...",
      "e": "AQAB"
    }
  ]
}

Key Rotation:
  1. Generate key mới (key-2026-01)
  2. Publish CẢ 2 keys trong JWKS (cũ + mới)
  3. IAM sign token bằng key mới
  4. Services verify: tìm key theo kid trong JWT header → match
  5. Sau overlap period (24h) → remove key cũ khỏi JWKS
  → Zero downtime rotation
```

---

## 6. Login Flow — End-to-End

```
Phase 1: TENANT RESOLUTION
═══════════════════════════════════════
1. User truy cập acme.gam.com (Module A)
2. Module A check: có access_token hợp lệ? → Không
3. Redirect đến IAM: GET /authorize
   ?client_id=module_a
   &redirect_uri=https://acme.gam.com/callback
   &response_type=code
   &scope=openid profile email
   &code_challenge=xxx
   &code_challenge_method=S256
   &state=random_csrf_token
4. IAM: parse subdomain "acme" → tenant_id = acme
   → Load tenant config (branding, SSO setting)

Phase 2: AUTHENTICATION
═══════════════════════════════════════
5. IAM hiện login page (tenant branding: logo Acme Corp)
6. User nhập email + password
7. IAM verify:
   a. Tenant "acme" tồn tại + active?
   b. User email tồn tại TRONG tenant "acme"?
   c. Password hash match (bcrypt/argon2)?
   d. Account locked? MFA required?
8. Auth thành công → generate authorization code
   → Lưu code + code_challenge vào Redis (TTL 60 giây)

Phase 3: TOKEN EXCHANGE
═══════════════════════════════════════
9. Redirect: acme.gam.com/callback?code=AUTH_CODE&state=random
10. Module A (backend or SPA): POST /token
    { grant_type: "authorization_code",
      code: AUTH_CODE,
      code_verifier: original_verifier,
      redirect_uri: "https://acme.gam.com/callback" }
11. IAM verify:
    a. Code hợp lệ + chưa dùng?
    b. SHA256(code_verifier) == stored code_challenge?
    c. redirect_uri match?
12. OK → issue tokens:
    - access_token (JWT, 15 phút)
    - id_token (JWT, user identity)
    - refresh_token (opaque, 7 ngày)
13. Delete authorization code (1 lần dùng)

Phase 4: AUTHENTICATED SESSION
═══════════════════════════════════════
14. Module A lưu access_token (memory), refresh_token (httpOnly cookie)
15. Mọi API call: Authorization: Bearer {access_token}
16. API Gateway / Service verify token:
    a. Signature valid? (check public key từ JWKS)
    b. Token chưa expire?
    c. Audience (aud) đúng service?
    d. Extract tenant_id, user_id → set context

Phase 5: CROSS-MODULE SSO
═══════════════════════════════════════
17. User navigate sang Module B (acme.gam.com/module-b)
18. Module B check: có access_token? → Không
19. Redirect đến IAM /authorize
20. IAM: user đã có session (cookie từ Phase 2)
    → KHÔNG hiện login page
    → Issue authorization code ngay
21. Module B exchange code → nhận tokens → user đã login
    → Zero-click SSO
```

### SSO Session

```
IAM maintain session bằng cookie:

  Set-Cookie: iam_session=encrypted_session_id;
    Domain=iam.gam.com;
    HttpOnly; Secure; SameSite=Lax;
    Max-Age=28800  (8 giờ)

Khi user đã login ở Module A → navigate sang Module B:
  → Module B redirect đến IAM
  → IAM nhận iam_session cookie → user đã authenticated
  → Skip login → issue code ngay → redirect về Module B
  → User không nhập password lần 2 → SSO

Session ≠ Token:
  - Session: trên IAM, dùng cho login flow (user đã authen chưa?)
  - Token: trên client, dùng cho API access (request này có quyền không?)
  - Revoke session → user phải login lại ở module tiếp theo
  - Revoke token → request hiện tại bị reject
```

---

## 7. User Management — GAM quản lý user

### User Lifecycle

```
                   GAM Admin tạo tenant
                         │
                         ▼
                  ┌──────────────┐
                  │ Tenant ACTIVE │
                  └──────┬───────┘
                         │
              Tenant Admin invite/tạo user
                         │
                         ▼
                  ┌──────────────┐
                  │  User INVITED │ ── email invitation gửi đi
                  └──────┬───────┘
                         │ user set password
                         ▼
                  ┌──────────────┐
                  │  User ACTIVE  │ ── login được
                  └──────┬───────┘
                         │
              ┌──────────┼──────────┐
              ▼                     ▼
       ┌──────────┐          ┌───────────┐
       │ SUSPENDED │          │ DEACTIVATED│
       │ (tạm khóa)│          │ (vô hiệu)  │
       └──────────┘          └───────────┘
```

### User thuộc Tenant

```
1 User thuộc đúng 1 Tenant (đơn giản, recommend ban đầu):
  user "john@acme.com" → tenant "Acme Corp"
  → john không thể login vào tenant "Beta Inc"

Tại sao không cho 1 user thuộc nhiều tenant?
  - Phức tạp: user chọn tenant khi login? auto-detect?
  - Token chứa tenant_id → 1 token cho 1 tenant → switch tenant = re-auth
  - Multi-tenant user = consultant use case → phase 2 nếu cần

Email unique trong tenant:
  UNIQUE (tenant_id, email)
  → "john@gmail.com" ở Acme ≠ "john@gmail.com" ở Beta
  → 2 account khác nhau
```

### Password Policy (per-tenant configurable)

```
Mỗi tenant có thể customize password policy:

{
  "tenant_id": "acme",
  "password_policy": {
    "min_length": 8,
    "require_uppercase": true,
    "require_number": true,
    "require_special": false,
    "max_age_days": 90,
    "history_count": 5
  }
}

Tại sao per-tenant?
  - Doanh nghiệp lớn: password policy nghiêm ngặt (compliance)
  - Startup nhỏ: policy đơn giản (UX first)
  - GAM cung cấp default + cho phép customize
```

---

## 8. Data Model

```sql
-- Tenant: mỗi doanh nghiệp khách hàng
CREATE TABLE tenants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug            TEXT NOT NULL UNIQUE,      -- subdomain: "acme"
    name            TEXT NOT NULL,             -- "Acme Corporation"
    status          TEXT NOT NULL DEFAULT 'ACTIVE',
        -- ACTIVE | SUSPENDED | DEACTIVATED
    settings        JSONB DEFAULT '{}',        -- password policy, branding, etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Tenant email domain mapping (cho email-based tenant resolution)
CREATE TABLE tenant_domains (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    domain          TEXT NOT NULL UNIQUE,      -- "acme.com"
    verified        BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- User: thuộc 1 tenant
CREATE TABLE users (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    email           TEXT NOT NULL,
    password_hash   TEXT,                      -- NULL nếu SSO-only user
    name            TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'INVITED',
        -- INVITED | ACTIVE | SUSPENDED | DEACTIVATED
    email_verified  BOOLEAN DEFAULT FALSE,
    mfa_enabled     BOOLEAN DEFAULT FALSE,
    last_login_at   TIMESTAMPTZ,
    password_changed_at TIMESTAMPTZ,
    failed_login_count INT DEFAULT 0,
    locked_until    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE (tenant_id, email)                  -- email unique per tenant
);

-- OAuth2 clients: mỗi GAM module là 1 client
CREATE TABLE oauth_clients (
    id              TEXT PRIMARY KEY,           -- "module_a", "module_b"
    secret_hash     TEXT,                      -- NULL cho public clients (SPA)
    client_type     TEXT NOT NULL,             -- 'public' (SPA) | 'confidential' (backend)
    redirect_uris   TEXT[] NOT NULL,           -- allowed redirect URIs
    allowed_scopes  TEXT[] NOT NULL,
    tenant_id       UUID REFERENCES tenants(id), -- NULL = platform-wide client
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Refresh tokens: server-side storage cho revocation
CREATE TABLE refresh_tokens (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    client_id       TEXT NOT NULL REFERENCES oauth_clients(id),
    token_hash      TEXT NOT NULL UNIQUE,       -- SHA256 hash, KHÔNG lưu raw token
    family_id       UUID NOT NULL,              -- cho rotation detection
    expires_at      TIMESTAMPTZ NOT NULL,
    revoked         BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_refresh_tokens_user ON refresh_tokens (user_id, revoked);
CREATE INDEX idx_refresh_tokens_family ON refresh_tokens (family_id);

-- Login audit: mọi lần login attempt
CREATE TABLE login_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    user_id         UUID,                      -- NULL nếu user không tồn tại
    email           TEXT NOT NULL,
    event_type      TEXT NOT NULL,
        -- LOGIN_SUCCESS | LOGIN_FAILED | LOGOUT | TOKEN_REFRESH | ACCOUNT_LOCKED
    ip              INET,
    user_agent      TEXT,
    failure_reason  TEXT,                       -- 'invalid_password', 'account_locked', etc.
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_login_events_user ON login_events (user_id, created_at DESC);
```

### Tại sao lưu refresh_token ở server (DB/Redis)?

```
Access token (JWT): stateless → verify bằng signature → KHÔNG cần DB
  → Nhanh, scalable
  → Nhưng KHÔNG revoke được (chờ expire 15 phút)

Refresh token: stateful → lưu ở server → CÓ THỂ revoke
  → Chậm hơn (query DB), nhưng chỉ dùng mỗi 15 phút
  → Revoke khi: user logout, admin deactivate, security incident
  → Lưu hash (không lưu raw token) → DB leak không compromise

Trade-off: access token stateless (fast, no revoke) +
           refresh token stateful (slower, revocable)
  → Kết hợp tốt nhất: nhanh cho 99% requests, revokable khi cần
```

---

## 9. Interview Quick Reference

### Elevator Pitch (30 giây)

> "IAM cho GAM dùng OAuth2 + OIDC làm core. Multi-tenant: mỗi doanh nghiệp là 1 tenant, user thuộc tenant, identify bằng subdomain. Authorization Code + PKCE cho SPA/mobile. JWT access token (RS256, 15 phút) verify bằng public key — stateless, services không cần gọi IAM mỗi request. Refresh token rotation lưu server-side cho revocation. SSO cross-module bằng IAM session cookie."

### "Tại sao OAuth2 + OIDC?"

> "Industry standard — client libraries mọi ngôn ngữ, không cần tự chế protocol. OAuth2 cho authorization (access_token), OIDC thêm authentication layer (id_token, userinfo). SPA dùng Authorization Code + PKCE (không cần client secret). Service-to-service dùng Client Credentials."

### "Multi-tenant isolation thế nào?"

> "Shared DB + tenant_id column. Enforce bằng: Row Level Security (PostgreSQL), application middleware set tenant context từ JWT, code review. Tenant identify bằng subdomain (acme.gam.com) hoặc email domain. Token chứa tenant_id → service filter data theo tenant."

### "SSO cross-module?"

> "User login Module A → IAM tạo session cookie. Navigate sang Module B → redirect đến IAM → session cookie còn → skip login → issue code → redirect về Module B với token. Zero-click."

### "Token revoke thế nào?"

> "Access token stateless (JWT) → không revoke, chờ expire 15 phút. Refresh token lưu server-side → revoke được. Rotation: mỗi lần dùng refresh token → issue mới + revoke cũ. Attacker dùng token cũ → conflict → revoke family → force re-login."

### Bảng quyết định

| Quyết định | Chọn | Tại sao |
|-----------|------|---------|
| Protocol | OAuth2 + OIDC | Standard, library support, SSO ready |
| Grant type (SPA) | Authorization Code + PKCE | SPA không giữ secret, PKCE thay thế |
| Token format | JWT (RS256) | Stateless verify, asymmetric key |
| Token lifetime | Access 15m, Refresh 7d | Short-lived access + revocable refresh |
| Multi-tenant DB | Shared DB + tenant_id + RLS | Simple ops, đủ cho < 1000 tenants |
| Tenant identify | Subdomain + email domain | Natural UX cho B2B |
| Refresh token storage | Server-side (DB) | Revocable, rotation detection |
| Password hash | Argon2id (hoặc bcrypt) | Industry standard, resist GPU attack |
