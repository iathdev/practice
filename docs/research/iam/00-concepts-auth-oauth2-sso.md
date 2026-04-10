# IAM — Khái niệm nền tảng: Authentication, OAuth2, OIDC, SSO

> Trước khi đọc kiến trúc IAM, cần hiểu rõ từng khái niệm. File này giải thích từ gốc: Authentication vs Authorization, session vs token, OAuth2 grant types, PKCE, OIDC, SSO, SAML — và cơ chế login thực tế của Google, GitHub, Twitter/X.

---

## Mục lục

1. [Authentication vs Authorization — Phân biệt gốc](#1-authentication-vs-authorization--phân-biệt-gốc)
2. [Session-Based vs Token-Based — 2 trường phái](#2-session-based-vs-token-based--2-trường-phái)
3. [OAuth2 — Framework, không phải protocol](#3-oauth2--framework-không-phải-protocol)
4. [OAuth2 Grant Types — 4 cách lấy token](#4-oauth2-grant-types--4-cách-lấy-token)
5. [PKCE — Tại sao SPA/Mobile cần thêm bước này](#5-pkce--tại-sao-spamobile-cần-thêm-bước-này)
6. [OpenID Connect (OIDC) — Authentication layer trên OAuth2](#6-openid-connect-oidc--authentication-layer-trên-oauth2)
7. [JWT — Cấu trúc token chi tiết](#7-jwt--cấu-trúc-token-chi-tiết)
8. [SSO — Single Sign-On hoạt động thế nào](#8-sso--single-sign-on-hoạt-động-thế-nào)
9. [SAML 2.0 — Enterprise SSO protocol](#9-saml-20--enterprise-sso-protocol)
10. [State Parameter — Chống CSRF trong OAuth2](#10-state-parameter--chống-csrf-trong-oauth2)
11. [Nonce — Chống Replay Attack trong OIDC](#11-nonce--chống-replay-attack-trong-oidc)
12. [Real-World: Google, GitHub, Twitter/X login thế nào](#12-real-world-google-github-twitterx-login-thế-nào)
13. [So sánh tổng hợp các protocol](#13-so-sánh-tổng-hợp-các-protocol)
14. [Thuật ngữ Quick Reference](#14-thuật-ngữ-quick-reference)

---

## 1. Authentication vs Authorization — Phân biệt gốc

```
Authentication (AuthN): BẠN LÀ AI?
  → Verify identity: email + password, fingerprint, SSO
  → Kết quả: "Đây là John, user_id = 123"
  → Ví dụ: nhập password → hệ thống xác nhận đúng là John

Authorization (AuthZ): BẠN ĐƯỢC LÀM GÌ?
  → Check permission: role, scope, policy
  → Kết quả: "John được xem report nhưng không được xóa"
  → Ví dụ: John gọi DELETE /reports/1 → 403 Forbidden

Thứ tự luôn là: AuthN trước → AuthZ sau
  Không biết bạn là ai → không thể quyết định bạn được làm gì
```

| | Authentication | Authorization |
|---|---|---|
| **Câu hỏi** | Bạn là ai? | Bạn được làm gì? |
| **Khi nào** | Login | Mỗi request |
| **Dựa vào** | Credentials (password, token, biometric) | Roles, permissions, policies |
| **Protocol** | OIDC, SAML | OAuth2 scopes, RBAC, ABAC |
| **Fail → response** | 401 Unauthorized | 403 Forbidden |

---

## 2. Session-Based vs Token-Based — 2 trường phái

### Session-Based (truyền thống)

```
┌──────────┐                     ┌──────────────┐
│  Browser  │                     │    Server     │
│           │ 1. POST /login      │              │
│           │    {email, password} │              │
│           │────────────────────▶│              │
│           │                     │ 2. Verify    │
│           │                     │ 3. Tạo session│
│           │                     │    lưu vào    │
│           │                     │    memory/DB  │
│           │ 4. Set-Cookie:      │              │
│           │    session_id=abc   │              │
│           │◀────────────────────│              │
│           │                     │              │
│           │ 5. GET /api/data    │              │
│           │    Cookie: abc      │              │
│           │────────────────────▶│              │
│           │                     │ 6. Lookup     │
│           │                     │    session abc│
│           │                     │    → user John│
│           │ 7. Response data    │              │
│           │◀────────────────────│              │
└──────────┘                     └──────────────┘

Server giữ state (session store):
  session_abc → { user_id: 123, tenant: acme, created_at: ... }
```

### Session Lookup — Server nhận session_id rồi làm gì?

```
Browser gửi: GET /api/data, Cookie: session_id=abc

Server làm gì (MỖI request):

  ┌──────────────────────────────────────────────────────────────────┐
  │ Bước 1: Parse cookie                                             │
  │   Request headers → tìm Cookie: session_id=abc                  │
  │   → Lấy ra session_id = "abc"                                   │
  │   → Nếu không có cookie → 401 (chưa login)                      │
  └───────────────────────────┬──────────────────────────────────────┘
                              │
  ┌───────────────────────────▼──────────────────────────────────────┐
  │ Bước 2: Lookup session từ session store                          │
  │                                                                   │
  │   Session store là nơi server LƯU mapping:                       │
  │     session_id → session data                                    │
  │                                                                   │
  │   Ví dụ với Redis (phổ biến nhất):                               │
  │     GET session:abc                                              │
  │     → '{"user_id":"123","tenant":"acme","expires_at":1712430000}'│
  │                                                                   │
  │   Ví dụ với Database:                                            │
  │     SELECT * FROM sessions WHERE id = 'abc'                      │
  │                                                                   │
  │   Ví dụ với In-Memory (HashMap trong process):                   │
  │     sessions.get("abc")                                          │
  │                                                                   │
  │   → Không tìm thấy → 401 (session hết hạn hoặc invalid)         │
  └───────────────────────────┬──────────────────────────────────────┘
                              │
  ┌───────────────────────────▼──────────────────────────────────────┐
  │ Bước 3: Validate session                                         │
  │   - Session tồn tại? → Không → 401                               │
  │   - expires_at > now? → Không → 401 (expired, xóa session)      │
  │   - User còn active? → Không → 401 (bị ban/deactivate)          │
  └───────────────────────────┬──────────────────────────────────────┘
                              │
  ┌───────────────────────────▼──────────────────────────────────────┐
  │ Bước 4: Set request context                                      │
  │   Từ session data, server biết:                                  │
  │     user_id = 123 → "đây là John"                               │
  │     tenant_id = acme → "thuộc doanh nghiệp Acme"                │
  │                                                                   │
  │   Gắn vào request context để handler/controller dùng:            │
  │     request.user = { id: 123, name: "John", tenant: "acme" }    │
  └───────────────────────────┬──────────────────────────────────────┘
                              │
  ┌───────────────────────────▼──────────────────────────────────────┐
  │ Bước 5: Tiếp tục xử lý request                                  │
  │   Controller nhận request.user → biết đây là John               │
  │   → Query data WHERE user_id = 123                               │
  │   → Trả response                                                │
  └──────────────────────────────────────────────────────────────────┘
```

**Bản chất**: session_id là **chìa khóa** tra cứu vào **bảng mapping** (session store) mà server giữ. Browser chỉ giữ chìa khóa, server giữ data. Mỗi request, server dùng chìa khóa để tìm ra "ai đang gọi".

### Session Store — Lưu ở đâu?

Session store là nơi server lưu mapping `session_id → user data`. Có nhiều cách:

| Session Store | Cơ chế | Latency | Scale |
|---|---|---|---|
| **In-Memory** | HashMap trong process | ~0.001ms | Không (1 instance) |
| **Database** | SELECT từ sessions table | ~1-5ms | Shared nhưng chậm |
| **Redis** (recommend) | GET key-value, TTL tự expire | ~0.1-0.5ms | Shared + nhanh |
| **Signed Cookie** | KHÔNG lookup — data NẰM TRONG cookie, verify bằng signature | ~0.01ms | Stateless |

```
Tại sao Redis phổ biến nhất?
  In-Memory:  Nhanh nhưng restart = mất, không shared giữa instances
  Database:   Shared nhưng MỖI request = 1 DB query
  Redis:      Nhanh (in-memory) + shared (multi-instance) + TTL tự xóa expired
              → Framework hỗ trợ sẵn (Spring Session, Express, Django)

Signed Cookie khác hoàn toàn:
  Không có session store. Session data encrypt + sign rồi nhét vào cookie.
  Server nhận cookie → verify signature → decrypt → lấy data.
  Giống JWT nhưng ở dạng cookie. Nhược: 4KB limit, khó revoke.
```

### So sánh: Session Lookup vs JWT Verify

```
Session-Based (mỗi request):
  Cookie: session_id=abc
  → Server QUERY session store (Redis/DB): "abc là ai?"
  → Session store trả: { user_id: 123 }
  → Server biết: đây là John
  ⚡ Cần round-trip đến session store

Token-Based JWT (mỗi request):
  Authorization: Bearer eyJhbG...
  → Server VERIFY signature bằng public key (local, không query gì)
  → Decode payload: { sub: "123", tenant: "acme" }
  → Server biết: đây là John
  ⚡ Không cần round-trip → nhanh hơn, stateless

Bản chất khác nhau:
  Session: "hỏi session store xem ID này là ai" (lookup)
  JWT: "token tự nói nó là ai, tôi chỉ verify chữ ký" (self-contained)
```

### Token-Based (hiện đại — OAuth2/JWT)

```
┌──────────┐                     ┌──────────────┐
│  Client   │                     │    Server     │
│  (SPA)    │ 1. POST /token      │              │
│           │    {email, password} │              │
│           │────────────────────▶│              │
│           │                     │ 2. Verify    │
│           │                     │ 3. Sign JWT  │
│           │                     │    (KHÔNG lưu)│
│           │ 4. { access_token:  │              │
│           │      "eyJhbG..." }  │              │
│           │◀────────────────────│              │
│           │                     │              │
│           │ 5. GET /api/data    │              │
│           │    Authorization:   │              │
│           │    Bearer eyJhbG... │              │
│           │────────────────────▶│              │
│           │                     │ 6. Verify    │
│           │                     │    signature │
│           │                     │    (public key)│
│           │                     │    → user John│
│           │ 7. Response data    │              │
│           │◀────────────────────│              │
└──────────┘                     └──────────────┘

Server KHÔNG giữ state. Token tự chứa thông tin user.
Verify bằng cryptographic signature, không query DB.
```

### So sánh

| | Session-Based | Token-Based (JWT) |
|---|---|---|
| **State** | Server giữ (stateful) | Client giữ (stateless server) |
| **Scale** | Cần shared session store (Redis) | Bất kỳ server nào verify được (public key) |
| **Revoke** | Xóa session → instant revoke | Không revoke được, chờ expire |
| **Cross-domain** | Khó (cookie same-origin) | Dễ (token gửi trong header) |
| **Mobile** | Khó (cookie management) | Dễ (token trong header) |
| **Microservices** | Mỗi service query session store | Mỗi service verify token local |
| **Khi nào** | Monolith, server-rendered | SPA, mobile, microservices |

---

## 3. OAuth2 — Framework, không phải protocol

### OAuth2 là gì?

```
OAuth2 KHÔNG phải authentication protocol.
OAuth2 là AUTHORIZATION framework.

Bài toán gốc:
  User muốn app X đọc Google Calendar của mình.
  NHƯNG không muốn đưa Google password cho app X.

  OAuth2 giải quyết:
    1. App X redirect user sang Google
    2. Google hỏi: "App X muốn đọc Calendar. Cho phép?"
    3. User đồng ý → Google cho app X 1 access_token
    4. App X dùng token → đọc Calendar → KHÔNG biết password

OAuth2 = "Cho phép app này truy cập tài nguyên của tôi"
  KHÔNG phải: "Xác nhận tôi là ai"
  (Xác nhận identity = OIDC, xây TRÊN OAuth2)
```

### 4 vai trò trong OAuth2

```
┌────────────────┐         ┌──────────────────┐
│ Resource Owner  │         │ Authorization     │
│ (User)          │         │ Server            │
│                 │         │ (Google, GAM IAM) │
│ Người sở hữu    │         │ Issue token       │
│ tài nguyên      │         │                   │
└────────┬────────┘         └────────┬──────────┘
         │                           │
         │ "Cho phép"                │ "Đây là token"
         │                           │
┌────────▼────────┐         ┌────────▼──────────┐
│ Client           │         │ Resource Server   │
│ (App X, SPA)     │         │ (Google Calendar, │
│                  │         │  GAM API)          │
│ Muốn truy cập   │         │ Giữ tài nguyên    │
│ tài nguyên       │         │ Verify token      │
└─────────────────┘         └───────────────────┘
```

---

## 4. OAuth2 Grant Types — 4 cách lấy token

### 4.1 Authorization Code (chuẩn nhất, an toàn nhất)

```
Dùng cho: Server-side app (có backend giữ secret)
Ví dụ: Web app truyền thống (PHP, Rails, Spring MVC)

Flow:
  1. Client redirect user → Authorization Server /authorize
  2. User login + consent
  3. Authorization Server SINH authorization code (random string)
     → Lưu mapping: code → { client_id, redirect_uri, scope, expire }
     → Redirect về client kèm code trong URL
  4. Client (backend) gửi CODE + client_secret → /token
  5. Authorization Server trả access_token

Authorization Code là gì?
  → Random string do AUTH SERVER sinh ra (KHÔNG phải client tạo)
  → "Biên nhận tạm" — chứng minh user đã login + consent xong
  → KHÔNG phải token — không dùng gọi API được
  → Dùng ĐÚNG 1 LẦN → exchange xong Auth Server xóa
  → Expire cực nhanh: 30-60 giây
  → Ví dụ: ?code=SplxlOBeZQQYbYS6WxSbIA

Tại sao 2 bước (code → token)?
  → Code đi qua browser (front-channel, URL) → có thể bị thấy
  → Token exchange qua backend (back-channel, POST body) → an toàn
  → Code chỉ dùng 1 lần, expire nhanh (60 giây)
  → Dù code bị steal → không có client_secret → không exchange được
  → Token KHÔNG BAO GIỜ lộ qua URL

┌──────────┐       ┌───────────┐       ┌──────────────┐
│ Browser   │       │ Backend   │       │ Auth Server  │
│           │       │ (giữ      │       │              │
│           │       │  secret)  │       │              │
│ 1. /authorize     │           │       │              │
│──────────────────────────────────────▶│              │
│           │       │           │       │ 2. Login page│
│◀──────────────────────────────────────│              │
│ 3. User login     │           │       │              │
│──────────────────────────────────────▶│              │
│ 4. Redirect + code│           │       │              │
│◀──────────────────────────────────────│              │
│ 5. Send code      │           │       │              │
│──────────────────▶│           │       │              │
│           │       │ 6. code + │       │              │
│           │       │ secret    │       │              │
│           │       │──────────────────▶│              │
│           │       │ 7. token  │       │              │
│           │       │◀──────────────────│              │
│ 8. Set session    │           │       │              │
│◀──────────────────│           │       │              │
└──────────┘       └───────────┘       └──────────────┘
```

### 4.2 Authorization Code + PKCE (cho SPA/Mobile)

```
Dùng cho: SPA (React, Vue), Mobile app — KHÔNG có backend giữ secret
Ví dụ: GAM modules (SPA), mobile app

Vấn đề: SPA chạy trong browser → source code public → KHÔNG giữ được client_secret
  → Authorization Code thường cần secret ở bước exchange
  → SPA không có secret → dùng gì?

PKCE (Proof Key for Code Exchange) thay thế secret:
  → Client tự tạo 1 cặp (code_verifier, code_challenge) mỗi lần login
  → Dùng code_challenge chứng minh "tôi là người khởi tạo request"
  → Chi tiết cơ chế xem mục 5

Flow:
  1. Client tạo code_verifier (random) + code_challenge = SHA256(verifier)
  2. Client redirect user → /authorize kèm code_challenge
  3. User login + consent
  4. Auth Server redirect về client với CODE
  5. Client gửi CODE + code_verifier → /token (KHÔNG cần client_secret)
  6. Auth Server: SHA256(code_verifier) == code_challenge đã lưu? → Match → issue token

┌──────────┐                              ┌──────────────┐
│ SPA/Mobile│                              │ Auth Server  │
│           │                              │              │
│ 1. Tạo:  │                              │              │
│  verifier │                              │              │
│  = random │                              │              │
│  challenge│                              │              │
│  = SHA256 │                              │              │
│  (verifier)                              │              │
│           │                              │              │
│ 2. GET /authorize                        │              │
│    ?code_challenge=xxx                   │              │
│    &code_challenge_method=S256           │              │
│    &client_id=spa_app                    │              │
│    &redirect_uri=.../callback            │              │
│    &scope=openid profile                 │              │
│    &state=csrf_random                    │              │
│──────────────────────────────────────────▶              │
│           │                              │ 3. Login page│
│           │                              │    User login│
│           │                              │    + consent │
│           │                              │              │
│           │                              │ 4. Lưu:     │
│           │                              │  code → challenge
│           │                              │              │
│ 5. Redirect + code + state              │              │
│◀─────────────────────────────────────────│              │
│           │                              │              │
│ 6. Verify state                          │              │
│           │                              │              │
│ 7. POST /token                           │              │
│    { code, code_verifier,                │              │
│      client_id, redirect_uri }           │              │
│──────────────────────────────────────────▶              │
│           │                              │              │
│           │                              │ 8. Verify:   │
│           │                              │  SHA256(verifier)
│           │                              │  == challenge?
│           │                              │  → Match!    │
│           │                              │              │
│ 9. { access_token, id_token }            │              │
│◀─────────────────────────────────────────│              │
└──────────┘                              └──────────────┘

So sánh với Authorization Code thường:
  ┌────────────────────────────┬──────────────────────────────────┐
  │ Authorization Code         │ Authorization Code + PKCE        │
  ├────────────────────────────┼──────────────────────────────────┤
  │ Exchange: code + secret    │ Exchange: code + code_verifier   │
  │ Secret cố định (hardcode)  │ Verifier tạo MỚI mỗi lần login │
  │ Secret lộ = compromised    │ Verifier lộ = chỉ 1 session     │
  │ Cần backend giữ secret     │ KHÔNG cần backend               │
  │ Server-side app            │ SPA / Mobile / Public client    │
  └────────────────────────────┴──────────────────────────────────┘

Tại sao PKCE an toàn hơn Implicit (deprecated)?
  Implicit: token trả thẳng qua URL → lộ trong browser history, log
  PKCE: token trả qua POST response → không lộ trong URL
  PKCE: có code_verifier → chứng minh client identity
  Implicit: không chứng minh được gì
```

### 4.3 Client Credentials (service-to-service)

```
Dùng cho: Backend service gọi backend service (KHÔNG có user context)
Ví dụ: Cron job, background worker, internal API

Flow (đơn giản nhất):
  POST /token
  {
    "grant_type": "client_credentials",
    "client_id": "service_a",
    "client_secret": "secret_xxx",
    "scope": "service_b:read"
  }
  → Response: { access_token: "..." }

Không có user → token chỉ đại diện cho service
Token payload: { sub: "service_a", scope: "service_b:read" }
  → Không có user_id, không có tenant_id
```

### 4.4 Refresh Token (gia hạn token)

```
Không phải grant type riêng — mà là cơ chế mở rộng:

Access token expire (15 phút):
  POST /token
  {
    "grant_type": "refresh_token",
    "refresh_token": "refresh_xxx"
  }
  → Response: { access_token: "new_access", refresh_token: "new_refresh" }

Tại sao cần refresh token?
  → Access token ngắn (15 phút) → bảo mật
  → Nhưng user không muốn login mỗi 15 phút
  → Refresh token dài (7 ngày) → lấy access token mới
  → Refresh token lưu server-side → revoke được
```

### Deprecated Grant Types (KHÔNG DÙNG)

```
Password Grant (Resource Owner Password):
  Client gửi email + password TRỰC TIẾP cho auth server
  ✗ Client biết password → trust issue
  ✗ Không hỗ trợ MFA
  ✗ OAuth2.1 đã loại bỏ

Implicit Grant:
  Token trả thẳng qua URL fragment (#access_token=...)
  ✗ Token lộ trong browser history, referrer header
  ✗ Không có refresh token
  ✗ OAuth2.1 đã loại bỏ → thay bằng Authorization Code + PKCE
```

---

## 5. PKCE — Tại sao SPA/Mobile cần thêm bước này

### Bài toán

```
Authorization Code flow cho SPA:
  1. SPA redirect user → /authorize
  2. User login → redirect về SPA với CODE
  3. SPA gửi CODE → /token
  4. Auth Server: "code đúng, nhưng BẠN LÀ AI?"
     → SPA không có client_secret → ai cũng có thể gửi code

Attack: Authorization Code Interception
  1. Attacker intercept redirect (malware, custom URL scheme mobile)
  2. Attacker lấy CODE
  3. Attacker gửi CODE → /token → nhận access_token
  → Attacker có token → giả danh user
```

### PKCE Flow (Proof Key for Code Exchange)

```
Trước khi redirect:
  1. Client tạo code_verifier = random string (43-128 chars)
     Ví dụ: "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk"

  2. Client tính code_challenge = BASE64URL(SHA256(code_verifier))
     Ví dụ: "E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM"

  3. Client GIỮ code_verifier (memory), GỬI code_challenge

Request /authorize:
  GET /authorize
    ?response_type=code
    &client_id=module_a
    &redirect_uri=https://acme.gam.com/callback
    &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
    &code_challenge_method=S256
    &scope=openid profile
    &state=random_csrf

Auth Server lưu: { code → code_challenge }

Exchange token:
  POST /token
  {
    "grant_type": "authorization_code",
    "code": "AUTH_CODE",
    "code_verifier": "dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk"
  }

Auth Server verify:
  SHA256("dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk")
  == stored code_challenge?
  → Match → issue token
  → Not match → reject
```

### Tại sao PKCE an toàn?

```
Attacker intercept CODE:
  → Gửi CODE → /token
  → Auth Server: "code_verifier đâu?"
  → Attacker KHÔNG CÓ code_verifier (chỉ client gốc có, giữ trong memory)
  → Reject

Attacker biết code_challenge (public):
  → code_challenge = SHA256(code_verifier)
  → SHA256 one-way → không tính ngược code_verifier
  → Vô dụng

Tóm lại:
  code_challenge: public, gửi lúc /authorize (ai thấy cũng được)
  code_verifier: secret, gửi lúc /token (chỉ client gốc có)
  → Chứng minh: "Tôi là người đã gửi request /authorize ban đầu"
```

---

## 6. OpenID Connect (OIDC) — Authentication layer trên OAuth2

### OAuth2 thiếu gì?

```
OAuth2 chỉ cho access_token → "được truy cập resource"
Nhưng KHÔNG nói: "user là ai"

Ví dụ:
  App nhận access_token từ Google
  → Gọi GET /calendar → OK, đọc được calendar
  → Nhưng: user là ai? Email gì? Tên gì?
  → OAuth2 KHÔNG định nghĩa cách lấy thông tin này

OIDC thêm:
  1. id_token — JWT chứa identity (sub, email, name)
  2. /userinfo endpoint — API lấy thông tin user
  3. Discovery — /.well-known/openid-configuration
  4. Standard claims — email, name, picture, ...
```

### id_token vs access_token

```
id_token (OIDC):
  → Cho CLIENT biết: "User là ai"
  → Dùng 1 lần khi login → extract identity
  → KHÔNG gửi cho API (không phải bearer token)
  → Chứa: sub (user ID), email, name, iat, exp, nonce

access_token (OAuth2):
  → Cho API biết: "Request này được phép"
  → Gửi kèm MỌI API call (Authorization: Bearer)
  → Có thể là JWT hoặc opaque string
  → Chứa: sub, scope, aud, exp

Ví dụ login "Login with Google":
  Google trả cả id_token + access_token:
  → id_token: { sub: "google_123", email: "john@gmail.com", name: "John" }
    → App biết: đây là John, email john@gmail.com
  → access_token: dùng để gọi Google APIs (Calendar, Drive, ...)
```

### Discovery Endpoint

```
GET https://accounts.google.com/.well-known/openid-configuration

{
  "issuer": "https://accounts.google.com",
  "authorization_endpoint": "https://accounts.google.com/o/oauth2/v2/auth",
  "token_endpoint": "https://oauth2.googleapis.com/token",
  "userinfo_endpoint": "https://openidconnect.googleapis.com/v1/userinfo",
  "jwks_uri": "https://www.googleapis.com/oauth2/v3/certs",
  "scopes_supported": ["openid", "email", "profile"],
  "response_types_supported": ["code", "token", "id_token"],
  "subject_types_supported": ["public"],
  "id_token_signing_alg_values_supported": ["RS256"]
}

Tại sao cần?
  → Client chỉ cần biết issuer URL → auto-discover tất cả endpoints
  → Không hardcode URL → IdP đổi endpoint → client tự update
  → Standard: mọi OIDC provider đều có endpoint này
```

---

## 7. JWT — Cấu trúc token chi tiết

### 3 phần: Header.Payload.Signature

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6ImtleS0yMDI2LTAxIn0
.
eyJpc3MiOiJodHRwczovL2lhbS5nYW0uY29tIiwic3ViIjoiMTIzIiwiZXhwIjoxNzEyNDAxODAwfQ
.
signature_bytes_here

Phần 1: Header (Base64URL encoded JSON)
  {
    "alg": "RS256",        // thuật toán ký
    "typ": "JWT",
    "kid": "key-2026-01"   // key ID → tìm public key trong JWKS
  }

Phần 2: Payload (Base64URL encoded JSON)
  {
    "iss": "https://iam.gam.com",   // issuer: ai ký token này
    "sub": "user_123",               // subject: user ID
    "aud": ["module-a"],             // audience: token dùng cho service nào
    "exp": 1712401800,               // expires: hết hạn khi nào (Unix timestamp)
    "iat": 1712400000,               // issued at: ký lúc nào
    "jti": "unique-token-id",        // JWT ID: unique ID cho revocation
    "tenant_id": "acme",             // custom claim: tenant
    "email": "john@acme.com",
    "name": "John Doe",
    "scope": "openid profile email"
  }

Phần 3: Signature
  RS256: sign(header + "." + payload, private_key)
  → Verify: verify(header + "." + payload, signature, public_key)
  → Nếu payload bị sửa → signature không match → reject
```

### Signing Algorithms

| Algorithm | Type | Key | Khi nào |
|-----------|------|-----|---------|
| **RS256** | Asymmetric (RSA) | Private sign, public verify | **Microservices (recommend)** — services chỉ cần public key |
| RS384/RS512 | Asymmetric (RSA) | Stronger RSA | High-security requirement |
| ES256 | Asymmetric (ECDSA) | Shorter key, same security | Mobile (nhỏ hơn RSA) |
| HS256 | Symmetric (HMAC) | Shared secret | Monolith (1 service sign + verify) |
| **none** | **Không ký** | — | **KHÔNG BAO GIỜ dùng. Vulnerability nổi tiếng** |

```
Tại sao RS256 cho microservices:
  HS256: sign + verify dùng CÙNG secret
    → Mỗi service có secret → service nào cũng forge được token
    → Secret leak = toàn bộ hệ thống compromised

  RS256: sign bằng private key (CHỈ IAM), verify bằng public key
    → Service chỉ có public key → KHÔNG forge được
    → Public key leak = vô hại (public mà)
    → Private key chỉ ở IAM → 1 nơi bảo vệ
```

### JWKS — Public Key Distribution

```
GET https://iam.gam.com/.well-known/jwks.json

{
  "keys": [
    {
      "kid": "key-2026-01",     // key ID — match JWT header.kid
      "kty": "RSA",
      "alg": "RS256",
      "use": "sig",
      "n": "0vx7agoebGc...",   // RSA modulus (public)
      "e": "AQAB"              // RSA exponent (public)
    }
  ]
}

Service verify token:
  1. Parse JWT header → kid = "key-2026-01"
  2. Fetch JWKS (cache 1 giờ) → tìm key có kid match
  3. Verify signature bằng public key
  4. Check exp, iss, aud
  → Toàn bộ verify KHÔNG cần gọi IAM → stateless, nhanh
```

---

## 8. SSO — Single Sign-On hoạt động thế nào

### Khái niệm

```
SSO: Login 1 lần → dùng nhiều ứng dụng

Không có SSO:
  Module A: login (nhập password)
  Module B: login (nhập password LẠI)
  Module C: login (nhập password LẠI LẦN NỮA)
  → User: "Tại sao phải login 3 lần?"

Có SSO:
  Module A: login (nhập password)
  Module B: đã đăng nhập (tự động)
  Module C: đã đăng nhập (tự động)
  → User: "Login 1 lần, dùng mọi nơi"
```

### SSO hoạt động nhờ shared session

```
              Module A           IAM            Module B
                │                 │                │
User mở A       │                 │                │
────────────────▶                 │                │
                │ Redirect /authorize              │
                │────────────────▶│                │
                │                 │ Login page     │
                │                 │◀── User login  │
                │                 │                │
                │                 │ Set session    │
                │                 │ cookie         │
                │ Code            │ (iam.gam.com)  │
                │◀────────────────│                │
                │ Exchange → token│                │
                │                 │                │
        ┈┈┈┈┈┈┈┈ Sau đó ┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈┈
                │                 │                │
User mở B       │                 │     ◀──────────│
                │                 │                │
                │                 │ Redirect       │
                │                 │ /authorize     │
                │                 │◀───────────────│
                │                 │                │
                │                 │ Session cookie │
                │                 │ đã có!         │
                │                 │ → SKIP login   │
                │                 │ → Issue code   │
                │                 │                │
                │                 │ Code ──────────▶
                │                 │ Exchange → token│
                │                 │                │
                         Module B đã login
                         KHÔNG nhập password
```

### Cốt lõi: IAM session cookie

```
Khi user login lần đầu:
  Set-Cookie: iam_session=encrypted_id;
    Domain=iam.gam.com;
    HttpOnly; Secure; SameSite=Lax

Khi Module B redirect đến IAM:
  → Browser tự gửi iam_session cookie (cùng domain iam.gam.com)
  → IAM: session hợp lệ → user đã authenticated
  → KHÔNG hiện login page → issue code ngay
  → Redirect về Module B → Module B có token

SSO = shared session trên IAM domain
  Module A, B, C đều redirect về IAM
  IAM nhận cùng session cookie
  → Login 1 lần ở IAM = login cho tất cả modules
```

---

## 9. SAML 2.0 — Enterprise SSO protocol

### SAML là gì?

```
SAML (Security Assertion Markup Language):
  → XML-based SSO protocol
  → Ra đời trước OAuth2/OIDC (2005)
  → Phổ biến trong enterprise (ADFS, Azure AD, Okta)
  → Dùng cho: browser-based SSO giữa enterprise IdP và SaaS app

Thuật ngữ:
  Identity Provider (IdP): Nơi authen user (Azure AD, ADFS)
  Service Provider (SP): App cần user login (GAM IAM)
  Assertion: XML document chứa user identity (giống id_token của OIDC)
  Binding: Cách gửi SAML messages (HTTP POST, HTTP Redirect)
```

### SAML Flow (SP-Initiated)

```
1. User truy cập GAM (SP)
2. GAM chưa login → tạo SAML AuthnRequest (XML, signed)
3. Redirect user → Enterprise IdP (Azure AD)
   (gửi AuthnRequest qua HTTP Redirect hoặc POST)
4. IdP hiện login page → user login
5. IdP tạo SAML Response (XML assertion, signed)
6. Redirect user → GAM /sso/saml/callback (POST form auto-submit)
7. GAM verify XML signature → extract user info → create session
```

### SAML Assertion (XML)

```xml
<saml:Assertion>
  <saml:Issuer>https://adfs.acme.com</saml:Issuer>
  <saml:Subject>
    <saml:NameID>john@acme.com</saml:NameID>
  </saml:Subject>
  <saml:Conditions NotBefore="..." NotOnOrAfter="...">
    <saml:AudienceRestriction>
      <saml:Audience>https://iam.gam.com</saml:Audience>
    </saml:AudienceRestriction>
  </saml:Conditions>
  <saml:AttributeStatement>
    <saml:Attribute Name="email">
      <saml:AttributeValue>john@acme.com</saml:AttributeValue>
    </saml:Attribute>
    <saml:Attribute Name="displayName">
      <saml:AttributeValue>John Doe</saml:AttributeValue>
    </saml:Attribute>
  </saml:AttributeStatement>
</saml:Assertion>
```

### SAML vs OIDC

| | SAML 2.0 | OIDC |
|---|---|---|
| **Format** | XML (verbose, complex) | JSON/JWT (compact, simple) |
| **Signature** | XML Signature (complex, security pitfalls) | JWS (standard, simple) |
| **Transport** | Browser redirect only | Browser + API (back-channel) |
| **Mobile** | Rất khó | Native support |
| **Discovery** | Manual metadata exchange | Auto-discovery (/.well-known/) |
| **Refresh** | Không có (re-authenticate) | Refresh token |
| **Tuổi** | 2005 | 2014 |
| **Dùng khi** | Enterprise legacy, ADFS | Default choice, modern apps |

---

## 10. State Parameter — Chống CSRF trong OAuth2

### Bài toán

```
CSRF (Cross-Site Request Forgery) trong OAuth2:

1. Attacker login tài khoản ATTACKER trên Google
2. Attacker lấy authorization code (cho tài khoản attacker)
3. Attacker gửi link cho victim:
   https://app.com/callback?code=ATTACKER_CODE
4. Victim click → app exchange ATTACKER_CODE → nhận ATTACKER token
5. App login victim vào tài khoản ATTACKER
6. Victim upload file nhạy cảm → file ở trong tài khoản attacker!

→ Victim bị trick vào tài khoản của attacker mà không biết
```

### State parameter chặn thế nào

```
Bước 1: Client tạo state = random string (unguessable)
  → Lưu vào session/localStorage

Bước 2: Gửi state trong /authorize request:
  GET /authorize?...&state=abc123random

Bước 3: Auth Server redirect về với CÙNG state:
  /callback?code=AUTH_CODE&state=abc123random

Bước 4: Client verify:
  state trong response == state đã lưu?
  → Match → OK, request này do chính client khởi tạo
  → Not match → REJECT (CSRF attack!)

Attacker gửi /callback?code=ATTACKER_CODE&state=???:
  → Attacker KHÔNG biết state của victim (random, lưu trong session)
  → state mismatch → client reject → attack fail
```

---

## 11. Nonce — Chống Replay Attack trong OIDC

### Bài toán

```
Replay attack:
  1. Attacker intercept id_token hợp lệ
  2. Attacker gửi lại id_token cho app
  3. App: "id_token hợp lệ" → login attacker vào tài khoản victim

Khác state (chống CSRF) vì:
  State: chống giả request từ bên ngoài
  Nonce: chống dùng lại token cũ
```

### Nonce flow

```
Bước 1: Client tạo nonce = random string
  → Lưu vào session

Bước 2: Gửi trong /authorize:
  GET /authorize?...&nonce=xyz789random

Bước 3: Auth Server đặt nonce VÀO id_token:
  id_token payload: { ..., "nonce": "xyz789random" }

Bước 4: Client verify:
  nonce trong id_token == nonce đã lưu?
  → Match → token này được issue CHO request này
  → Not match → REJECT (replay hoặc wrong token)

Attacker replay id_token cũ:
  → nonce trong token cũ ≠ nonce client mới tạo
  → Reject → replay fail
```

---

## 12. Real-World: Google, GitHub, Twitter/X login thế nào

### "Login with Google" (OIDC)

```
Google dùng: OAuth2 + OIDC (Authorization Code + PKCE)

1. App redirect:
   https://accounts.google.com/o/oauth2/v2/auth
     ?client_id=YOUR_CLIENT_ID
     &redirect_uri=https://yourapp.com/callback
     &response_type=code
     &scope=openid email profile
     &state=random_csrf
     &nonce=random_replay
     &code_challenge=xxx
     &code_challenge_method=S256

2. Google login page → user chọn account

3. Consent screen: "App X muốn biết email và tên. Cho phép?"

4. Redirect: yourapp.com/callback?code=AUTH_CODE&state=random_csrf

5. Backend exchange:
   POST https://oauth2.googleapis.com/token
   { code, code_verifier, client_id, client_secret, redirect_uri }

6. Response: { access_token, id_token, refresh_token }
   id_token: { sub: "google_123", email: "john@gmail.com",
               name: "John Doe", picture: "...", nonce: "random_replay" }

7. App verify id_token:
   → Signature (Google public key từ JWKS)
   → iss == "https://accounts.google.com"
   → aud == YOUR_CLIENT_ID
   → exp chưa qua
   → nonce match

8. App: tạo/login user John → set session/token
```

### "Login with GitHub" (OAuth2 — không phải OIDC)

```
GitHub dùng: OAuth2 (Authorization Code), KHÔNG phải OIDC
→ Không có id_token, không có discovery endpoint

1. Redirect:
   https://github.com/login/oauth/authorize
     ?client_id=YOUR_CLIENT_ID
     &redirect_uri=https://yourapp.com/callback
     &scope=user:email
     &state=random_csrf

2. GitHub login → consent

3. Redirect: yourapp.com/callback?code=AUTH_CODE&state=random_csrf

4. Exchange (GitHub dùng POST, trả về dạng khác):
   POST https://github.com/login/oauth/access_token
   Accept: application/json
   { client_id, client_secret, code, redirect_uri }

   Response: { access_token, token_type: "bearer", scope: "user:email" }
   → KHÔNG có id_token (không phải OIDC)

5. Lấy user info bằng access_token (thêm 1 API call):
   GET https://api.github.com/user
   Authorization: Bearer ACCESS_TOKEN

   Response: { id: 123, login: "john", email: "john@github.com", name: "John" }

6. App: tạo/login user John

Khác Google:
  - Không có id_token → phải gọi /user API riêng
  - Không có JWKS, discovery → hardcode endpoints
  - Không có nonce (không phải OIDC)
```

### "Login with Twitter/X" (OAuth 2.0 + PKCE)

```
Twitter/X dùng: OAuth 2.0 với PKCE (Authorization Code + PKCE)
(Trước đây dùng OAuth 1.0a — phức tạp hơn nhiều)

1. Redirect:
   https://twitter.com/i/oauth2/authorize
     ?response_type=code
     &client_id=YOUR_CLIENT_ID
     &redirect_uri=https://yourapp.com/callback
     &scope=tweet.read users.read
     &state=random_csrf
     &code_challenge=xxx
     &code_challenge_method=S256

2. Twitter login → consent

3. Redirect: yourapp.com/callback?code=AUTH_CODE&state=random_csrf

4. Exchange (PKCE — public client, không cần secret):
   POST https://api.twitter.com/2/oauth2/token
   {
     code: AUTH_CODE,
     grant_type: "authorization_code",
     client_id: YOUR_CLIENT_ID,
     redirect_uri: "https://yourapp.com/callback",
     code_verifier: "original_verifier"
   }

   Response: { access_token, token_type: "bearer" }

5. Lấy user info:
   GET https://api.twitter.com/2/users/me
   Authorization: Bearer ACCESS_TOKEN

   Response: { id: "123", name: "John", username: "john_doe" }

Twitter PKCE flow:
  → Không cần client_secret (public client)
  → code_verifier chứng minh client identity
  → An toàn cho SPA/mobile
```

### Tổng hợp: 3 providers, 3 cách

| | Google | GitHub | Twitter/X |
|---|---|---|---|
| **Protocol** | OIDC (OAuth2 + identity) | OAuth2 only | OAuth2 + PKCE |
| **id_token** | Có | Không | Không |
| **PKCE** | Có | Không (dùng secret) | Có |
| **Lấy user info** | id_token (hoặc /userinfo) | GET /user (API call) | GET /users/me (API call) |
| **Discovery** | /.well-known/ | Không | Không |
| **Refresh token** | Có | Không (token dài hạn) | Có |

---

## 13. So sánh tổng hợp các protocol

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│  OAuth 2.0          → Authorization (access_token)               │
│    │                                                              │
│    ├── + OIDC        → + Authentication (id_token, /userinfo)    │
│    │                                                              │
│    ├── + PKCE        → + SPA/Mobile security (code_verifier)     │
│    │                                                              │
│    └── + state/nonce → + CSRF/Replay protection                  │
│                                                                   │
│  SAML 2.0           → Enterprise SSO (XML assertion)             │
│    (độc lập, không dựa trên OAuth2)                              │
│                                                                   │
│  JWT                → Token format (Header.Payload.Signature)    │
│    (dùng bởi OIDC, có thể dùng bởi OAuth2)                      │
│                                                                   │
│  SSO                → Concept (login 1 lần, dùng nhiều app)      │
│    (implement bằng OIDC, SAML, hoặc custom session)             │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
```

---

## 14. Thuật ngữ Quick Reference

| Thuật ngữ | Giải thích ngắn |
|-----------|----------------|
| **OAuth2** | Authorization framework — lấy access_token để truy cập resource |
| **OIDC** | Authentication layer trên OAuth2 — thêm id_token để biết user là ai |
| **JWT** | JSON Web Token — format token: Header.Payload.Signature (Base64URL) |
| **JWKS** | JSON Web Key Set — endpoint publish public keys cho token verification |
| **PKCE** | Proof Key for Code Exchange — thay client_secret cho SPA/Mobile |
| **Authorization Code** | Grant type: redirect → code → exchange → token (an toàn nhất) |
| **Client Credentials** | Grant type: service-to-service, không có user context |
| **Refresh Token** | Token dài hạn, dùng để lấy access_token mới khi expire |
| **Access Token** | Token ngắn hạn (15m), gửi kèm API requests |
| **id_token** | JWT chứa user identity (OIDC), dùng 1 lần khi login |
| **SSO** | Single Sign-On — login 1 lần dùng nhiều app |
| **SAML** | XML-based SSO protocol — enterprise legacy |
| **IdP** | Identity Provider — nơi quản lý identity (Google, Azure AD) |
| **SP** | Service Provider — app cần user login (GAM) |
| **state** | Random string chống CSRF trong OAuth2 redirect flow |
| **nonce** | Random string chống replay attack trong OIDC id_token |
| **code_verifier** | PKCE: random secret giữ ở client, gửi khi exchange token |
| **code_challenge** | PKCE: SHA256(code_verifier), gửi khi request authorize |
| **RS256** | RSA SHA-256 — asymmetric signing (private sign, public verify) |
| **HS256** | HMAC SHA-256 — symmetric signing (shared secret) |
| **kid** | Key ID — trong JWT header, dùng để tìm public key trong JWKS |
| **iss** | Issuer — JWT claim: ai ký token |
| **sub** | Subject — JWT claim: token đại diện cho ai |
| **aud** | Audience — JWT claim: token dùng cho service nào |
| **exp** | Expiration — JWT claim: token hết hạn lúc nào |
| **jti** | JWT ID — unique ID cho token (dùng cho revocation) |
| **Consent** | Màn hình hỏi user: "App X muốn truy cập data. Cho phép?" |
| **Scope** | Phạm vi quyền: openid, email, profile, calendar.read, ... |
| **Bearer Token** | Cách gửi token: `Authorization: Bearer {token}` |
| **Opaque Token** | Token không decode được (random string), server lookup |
| **RLS** | Row Level Security — PostgreSQL tự filter rows theo context |
| **MFA** | Multi-Factor Authentication — thêm yếu tố xác thực (OTP, biometric) |
| **JIT Provisioning** | Just-In-Time — auto-create user khi SSO login lần đầu |
| **SCIM** | System for Cross-domain Identity Management — API sync users giữa IdP và SP |
