# IAM — SSO, Federation & External Identity Provider

> Doanh nghiệp khách hàng đã có hệ thống identity riêng (Azure AD, Google Workspace, Okta). Họ muốn: nhân viên login GAM bằng tài khoản công ty, không cần tạo password mới. File này giải quyết: Enterprise SSO, SAML/OIDC federation, và provisioning.

---

## Mục lục

1. [Tại sao cần Enterprise SSO?](#1-tại-sao-cần-enterprise-sso)
2. [Federation Model — IAM là Service Provider](#2-federation-model--iam-là-service-provider)
3. [SAML 2.0 vs OIDC Federation — Khi nào dùng gì?](#3-saml-20-vs-oidc-federation--khi-nào-dùng-gì)
4. [OIDC Federation Flow — Chi tiết](#4-oidc-federation-flow--chi-tiết)
5. [SAML Federation Flow — Chi tiết](#5-saml-federation-flow--chi-tiết)
6. [User Provisioning — Tạo user tự động khi SSO](#6-user-provisioning--tạo-user-tự-động-khi-sso)
7. [Tenant SSO Configuration](#7-tenant-sso-configuration)
8. [Interview Quick Reference](#8-interview-quick-reference)

---

## 1. Tại sao cần Enterprise SSO?

### Bài toán

```
Doanh nghiệp "Acme Corp" đã có Azure AD:
  - 500 nhân viên, tài khoản john@acme.com quản lý bởi IT
  - Single Sign-On: 1 tài khoản cho mọi tool nội bộ
  - Security policy: MFA bắt buộc, password rotate 90 ngày

Acme mua GAM:
  "Chúng tôi muốn nhân viên dùng tài khoản Azure AD để login GAM.
   KHÔNG muốn tạo password riêng cho GAM.
   KHÔNG muốn quản lý thêm 1 hệ thống user nữa."

Nếu GAM không hỗ trợ:
  → 500 nhân viên phải tạo password mới cho GAM
  → IT quản lý thêm 1 nơi: nhân viên nghỉ việc → quên revoke GAM account
  → Mất deal B2B: "Không hỗ trợ SSO? Không mua."
```

### SSO là yêu cầu B2B gần như bắt buộc

```
Enterprise deal (>50 users):
  Checklist mua phần mềm:
    ✓ SAML/OIDC SSO
    ✓ SCIM provisioning
    ✓ Audit logs
    ✓ Data residency

  Không có SSO → eliminiate từ vòng đánh giá
  → Mất revenue B2B đáng kể

Phân loại tenant:
  Small tenant (< 20 users): username/password do GAM quản lý → đủ
  Medium tenant (20-100): optional SSO, nice to have
  Enterprise tenant (100+): SSO REQUIRED, thường là deal breaker
```

---

## 2. Federation Model — IAM là Service Provider

### Thuật ngữ

```
Identity Provider (IdP): Nơi quản lý identity gốc
  → Azure AD, Google Workspace, Okta, OneLogin
  → Doanh nghiệp sở hữu và quản lý

Service Provider (SP): Nơi cần authenticate user
  → GAM IAM
  → Tin tưởng IdP: "IdP nói user này hợp lệ → tôi tin"

Federation: Trust relationship giữa IdP và SP
  → IdP authenticate user → gửi assertion/token cho SP
  → SP nhận → tạo session → user đã login
```

### Kiến trúc Federation

```
┌──────────────┐         ┌──────────────────┐        ┌────────────────┐
│  Acme Corp   │         │    GAM IAM        │        │  GAM Modules   │
│  Azure AD    │         │  (Service Provider)│       │                │
│  (IdP)       │         │                    │        │  Module A      │
│              │◀════════│  Tenant "acme"     │        │  Module B      │
│  Authen user │  SAML/  │  SSO config:       │        │  Module C      │
│  → assertion │  OIDC   │    idp_type: azure │        │                │
│              │════════▶│    client_id: xxx  │═══════▶│  (nhận JWT     │
│              │         │    ...              │ OAuth2 │   từ IAM)     │
└──────────────┘         └──────────────────┘        └────────────────┘

Flow:
  1. User login GAM → IAM detect tenant có SSO config
  2. IAM redirect user sang Azure AD (IdP)
  3. Azure AD authenticate → redirect về IAM với assertion
  4. IAM verify assertion → map user → issue GAM JWT
  5. User dùng GAM JWT để access modules (như flow thường)
```

### IAM làm trung gian (broker)

```
Tại sao IAM làm trung gian, Module A không nói chuyện trực tiếp với Azure AD?

Direct (Module A ↔ Azure AD):
  ✗ Mỗi module tự integrate SAML/OIDC → duplicate code
  ✗ Thêm module mới → configure lại Azure AD
  ✗ Tenant dùng Okta thay Azure → sửa TẤT CẢ modules
  ✗ Không consistent: module A dùng OIDC, module B dùng SAML?

IAM broker (Module A ↔ IAM ↔ Azure AD):
  ✓ Module A chỉ biết OAuth2 (nói chuyện với IAM)
  ✓ IAM xử lý SAML/OIDC với Azure AD
  ✓ Đổi IdP → chỉ đổi config IAM, modules không đổi
  ✓ Thêm module → chỉ register OAuth2 client trong IAM
  ✓ IAM normalize: Azure AD (SAML) hay Google (OIDC) → đều ra GAM JWT

Modules luôn nói OAuth2 với IAM. IAM nói SAML/OIDC với IdP.
→ Modules không cần biết IdP là ai. Clean separation.
```

---

## 3. SAML 2.0 vs OIDC Federation — Khi nào dùng gì?

### So sánh

| | SAML 2.0 | OIDC (OpenID Connect) |
|---|---|---|
| **Format** | XML assertion | JSON (JWT) |
| **Transport** | Browser redirect (POST binding) | Browser redirect + API |
| **Phổ biến ở** | Enterprise legacy (ADFS, on-prem) | Modern cloud (Azure AD, Google, Okta) |
| **Phức tạp** | Cao (XML signature, certificates) | Thấp hơn (JSON, standard OAuth2) |
| **Mobile friendly** | Không (XML, browser redirect) | Có (API-based, token) |
| **Khi nào dùng** | Doanh nghiệp yêu cầu SAML (legacy) | Default choice, modern IdP |

### GAM hỗ trợ cả 2

```
Tại sao cả 2?
  - Enterprise lớn (bank, gov): thường chỉ support SAML
  - Company dùng Google Workspace: prefer OIDC
  - Company dùng Azure AD: support cả SAML lẫn OIDC
  - Nếu chỉ hỗ trợ 1 → mất deal với nhóm còn lại

Thứ tự ưu tiên:
  1. OIDC (default, đơn giản hơn, recommend cho tenant mới)
  2. SAML (khi tenant yêu cầu, legacy system)
```

---

## 4. OIDC Federation Flow — Chi tiết

```
Tenant "Acme" config SSO với Google Workspace (OIDC):

┌──────────┐    ┌──────────┐    ┌──────────────┐    ┌──────────────┐
│  User     │    │  Module A │    │  GAM IAM      │    │ Google       │
│  Browser  │    │  (SPA)    │    │  (SP + Broker)│    │ Workspace    │
│           │    │           │    │               │    │ (IdP)        │
└─────┬─────┘    └─────┬─────┘    └───────┬───────┘    └──────┬───────┘
      │                │                   │                    │
      │ 1. Truy cập    │                   │                    │
      │ acme.gam.com   │                   │                    │
      │───────────────▶│                   │                    │
      │                │ 2. No token →     │                    │
      │                │ redirect to IAM   │                    │
      │◀───────────────│ /authorize        │                    │
      │                │                   │                    │
      │ 3. GET /authorize?tenant=acme      │                    │
      │────────────────────────────────────▶│                    │
      │                │                   │                    │
      │                │        4. Lookup: tenant "acme"        │
      │                │           → SSO enabled                │
      │                │           → IdP = Google OIDC          │
      │                │           → client_id = xxx            │
      │                │                   │                    │
      │ 5. Redirect to Google /authorize   │                    │
      │◀───────────────────────────────────│                    │
      │                │                   │                    │
      │ 6. Google login page               │                    │
      │─────────────────────────────────────────────────────────▶
      │                │                   │                    │
      │ 7. User login Google account       │                    │
      │─────────────────────────────────────────────────────────▶
      │                │                   │                    │
      │ 8. Redirect: IAM /sso/callback     │                    │
      │    ?code=GOOGLE_AUTH_CODE          │                    │
      │────────────────────────────────────▶│                    │
      │                │                   │                    │
      │                │        9. IAM exchange code with Google│
      │                │                   │────────────────────▶
      │                │                   │ { id_token, access }│
      │                │                   │◀────────────────────
      │                │                   │                    │
      │                │       10. Verify id_token:             │
      │                │           - email: john@acme.com       │
      │                │           - email_verified: true       │
      │                │                   │                    │
      │                │       11. Find/create user in GAM:     │
      │                │           - tenant: acme               │
      │                │           - email: john@acme.com       │
      │                │           - JIT provision nếu chưa có  │
      │                │                   │                    │
      │                │       12. Issue GAM authorization code │
      │ 13. Redirect: Module A /callback   │                    │
      │    ?code=GAM_AUTH_CODE             │                    │
      │◀───────────────────────────────────│                    │
      │                │                   │                    │
      │ 14. Exchange   │                   │                    │
      │────────────────▶ POST /token       │                    │
      │                │───────────────────▶                    │
      │                │ { gam_access_token,│                    │
      │                │   gam_id_token }  │                    │
      │                │◀──────────────────│                    │
      │                │                   │                    │
      │ 15. User logged in with GAM JWT    │                    │
      │◀───────────────│                   │                    │
```

### Key point: Double redirect

```
User → Module A → IAM → Google → IAM → Module A

IAM nhận assertion từ Google → KHÔNG gửi Google token cho Module A
→ IAM issue GAM JWT (access_token, id_token riêng của GAM)
→ Module A chỉ biết GAM JWT, không biết user login qua Google hay password

Tại sao?
  - Module A consistent: luôn nhận GAM JWT, bất kể auth method
  - Token format, claims, lifetime do GAM control
  - Đổi IdP → modules không ảnh hưởng
```

---

## 5. SAML Federation Flow — Chi tiết

```
Tenant "BigBank" config SSO với ADFS (SAML):

SP-Initiated flow (user bắt đầu từ GAM):

1. User truy cập bigbank.gam.com
2. Module redirect → IAM /authorize
3. IAM: tenant "bigbank" → SSO = SAML → IdP = ADFS
4. IAM generate SAML AuthnRequest (XML, signed)
5. Redirect user → ADFS login page (POST binding)
6. User login AD credentials
7. ADFS → SAML Response (XML assertion, signed)
8. Redirect → IAM /sso/saml/callback (POST)
9. IAM verify:
   a. XML signature valid? (ADFS certificate)
   b. Assertion chưa expire?
   c. Audience = GAM IAM?
   d. Extract: NameID (email), attributes
10. Find/create user → issue GAM JWT → redirect Module
```

### SAML phức tạp hơn OIDC ở đâu?

```
SAML:
  - XML parsing + signature verification (complex, security risk: XML attacks)
  - Certificate management: IdP cert rotate → IAM update
  - POST binding: form auto-submit → CORS issues
  - No refresh: SAML assertion dùng 1 lần → IAM tự quản lý session

OIDC:
  - JSON + JWT → standard parsing
  - JWKS endpoint → key rotation tự động
  - API-based → mobile friendly
  - Refresh token built-in

→ Ưu tiên OIDC. Hỗ trợ SAML cho enterprise legacy.
```

---

## 6. User Provisioning — Tạo user tự động khi SSO

### Bài toán

```
Acme Corp enable SSO. Nhân viên mới john@acme.com:
  - Azure AD: account đã có
  - GAM: chưa có user

John login GAM lần đầu:
  → IAM nhận assertion: email=john@acme.com, name=John Doe
  → GAM chưa có user → tạo hay reject?
```

### Just-In-Time (JIT) Provisioning

```
User login qua SSO + chưa có account GAM:
  → IAM auto-create user trong GAM

Flow:
  1. OIDC/SAML assertion: { email: "john@acme.com", name: "John Doe" }
  2. IAM lookup: user "john@acme.com" trong tenant "acme" → NOT FOUND
  3. JIT provisioning enabled cho tenant?
     Có → INSERT user: { tenant: acme, email: john@acme.com, name: John Doe,
                          status: ACTIVE, auth_method: SSO }
     Không → reject: "Tài khoản chưa được tạo. Liên hệ admin."
  4. Issue GAM JWT cho user mới

Ưu điểm:
  ✓ Admin không cần tạo trước 500 accounts
  ✓ Nhân viên mới → login → tự có account
  ✓ Giảm workload IT

Nhược điểm:
  ✗ Ai cũng login được nếu có Azure AD account
  ✗ Cần domain restrict: chỉ @acme.com được JIT, không phải @gmail.com
```

### SCIM Provisioning (future — enterprise feature)

```
SCIM (System for Cross-domain Identity Management):
  → IdP push user changes sang GAM qua API
  → Tạo user, update, deactivate — tự động

Flow:
  Azure AD                    GAM IAM
      │                           │
      │ POST /scim/Users          │
      │ { email, name, ... }      │
      │──────────────────────────▶│ → Create user
      │                           │
      │ PUT /scim/Users/{id}      │
      │ { name: "John Smith" }    │
      │──────────────────────────▶│ → Update user
      │                           │
      │ DELETE /scim/Users/{id}   │
      │──────────────────────────▶│ → Deactivate user
      │                           │

Khi nào cần SCIM?
  - Enterprise >500 users: JIT không đủ (cần deactivate khi nghỉ việc)
  - Compliance: user offboard → revoke GAM access ngay
  - Khi nào: Phase 2/3 — sau khi JIT ổn định

JIT vs SCIM:
  JIT:  User login lần đầu → create. Không delete khi offboard.
  SCIM: IdP push create/update/delete → full lifecycle management.
```

---

## 7. Tenant SSO Configuration

### Data Model

```sql
CREATE TABLE tenant_sso_configs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenants(id) UNIQUE,
    protocol        TEXT NOT NULL,              -- 'oidc' | 'saml'
    enabled         BOOLEAN DEFAULT FALSE,

    -- OIDC config
    oidc_issuer         TEXT,                   -- https://accounts.google.com
    oidc_client_id      TEXT,
    oidc_client_secret_encrypted TEXT,          -- encrypted, KHÔNG plain text
    oidc_scopes         TEXT[] DEFAULT '{openid,email,profile}',

    -- SAML config
    saml_idp_entity_id  TEXT,
    saml_sso_url        TEXT,                   -- IdP login URL
    saml_certificate    TEXT,                   -- IdP public cert (PEM)
    saml_sp_entity_id   TEXT,                   -- GAM SP entity ID

    -- Provisioning
    jit_provisioning    BOOLEAN DEFAULT TRUE,

    -- Enforce: bắt buộc SSO hay cho phép password login?
    enforce_sso         BOOLEAN DEFAULT FALSE,
    -- TRUE: user PHẢI login qua SSO, không được dùng password
    -- FALSE: user chọn SSO hoặc password

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Enforce SSO

```
enforce_sso = false (default):
  Login page hiện:
    ┌──────────────────────────┐
    │  Login to Acme Corp      │
    │                          │
    │  [Login with Azure AD]   │  ← SSO button
    │                          │
    │  ─── OR ───              │
    │                          │
    │  Email: [            ]   │  ← Password login
    │  Password: [         ]   │
    │  [Login]                 │
    └──────────────────────────┘

enforce_sso = true (enterprise requirement):
  Login page hiện:
    ┌──────────────────────────┐
    │  Login to BigBank        │
    │                          │
    │  [Login with Corporate   │
    │   Account]               │
    │                          │
    │  Không hiện password form│
    └──────────────────────────┘

  → Doanh nghiệp muốn: mọi nhân viên PHẢI login qua Azure AD
  → Không cho phép tạo password riêng trong GAM
  → IT kiểm soát access hoàn toàn qua Azure AD
  → Nhân viên nghỉ việc → disable Azure AD → GAM access revoked
```

---

## 8. Interview Quick Reference

### "Enterprise SSO hoạt động thế nào?"

> "IAM là broker. Module A chỉ nói OAuth2 với IAM. IAM nói OIDC/SAML với Enterprise IdP (Azure AD, Google). User login → redirect IAM → redirect IdP → IdP authenticate → redirect IAM → IAM verify assertion → map user → issue GAM JWT → redirect Module A. Module A nhận GAM JWT — không biết user login qua IdP hay password."

### "SAML hay OIDC?"

> "OIDC là default (JSON, JWT, mobile-friendly, auto key rotation). SAML cho enterprise legacy (ADFS, on-prem IdP). GAM hỗ trợ cả 2 vì: không hỗ trợ SAML = mất deal enterprise. Per-tenant config: tenant A dùng OIDC (Google), tenant B dùng SAML (ADFS)."

### "User provisioning?"

> "Phase 1: JIT (Just-In-Time) — user login SSO lần đầu → auto-create GAM account. Đơn giản, cover 80% case. Phase 2: SCIM — IdP push create/update/delete qua API. Cần cho enterprise lớn: nhân viên nghỉ → auto-deactivate GAM account."

### "Enforce SSO?"

> "Per-tenant config. Enterprise muốn: mọi nhân viên PHẢI login qua Azure AD, không cho password riêng. enforce_sso=true → login page chỉ hiện SSO button. IT revoke Azure AD → GAM access revoked."

### Bảng quyết định

| Quyết định | Chọn | Tại sao |
|-----------|------|---------|
| IAM role | Broker (SP) giữa modules và IdP | Modules consistent, đổi IdP không ảnh hưởng modules |
| Protocol support | OIDC (default) + SAML (legacy) | Mất deal nếu thiếu 1 trong 2 |
| Provisioning | JIT (phase 1) → SCIM (phase 2) | JIT đơn giản, SCIM cho enterprise lifecycle |
| SSO enforcement | Per-tenant config | Small tenant: optional. Enterprise: required |
| Token output | Luôn GAM JWT (bất kể IdP) | Modules không biết/cần biết IdP |
