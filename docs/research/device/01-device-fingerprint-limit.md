# Device Fingerprint & Device Limit — Định danh thiết bị & Giới hạn 3 devices

> Bài toán: Thí sinh chỉ được sử dụng tối đa 3 thiết bị cho việc thi và học trên hệ thống PREP. Cần định danh thiết bị qua browser mà không cần cài app.

---

## Mục lục

1. [Bài toán & Yêu cầu](#1-bài-toán--yêu-cầu)
2. [Device Fingerprint là gì?](#2-device-fingerprint-là-gì)
3. [Các tín hiệu để fingerprint](#3-các-tín-hiệu-để-fingerprint)
4. [Kiến trúc tổng thể](#4-kiến-trúc-tổng-thể)
5. [Data Model](#5-data-model)
6. [Flow chi tiết: Từ mở browser đến allow/deny](#6-flow-chi-tiết-từ-mở-browser-đến-allowdeny)
7. [Fuzzy Matching — Tại sao không so exact hash?](#7-fuzzy-matching--tại-sao-không-so-exact-hash)
8. [Device-Bound Token — Layer chính cho định danh](#8-device-bound-token--layer-chính-cho-định-danh)
9. [Thử thách & Edge Cases](#9-thử-thách--edge-cases)
10. [Device Management UI](#10-device-management-ui)
11. [Interview Quick Reference](#11-interview-quick-reference)

---

## 1. Bài toán & Yêu cầu

### Đề bài

```
Hệ thống PREP (B2B English learning & exam):
  - Mỗi user được dùng TỐI ĐA 3 thiết bị (laptop, điện thoại, tablet...)
  - Áp dụng cho cả HỌC và THI
  - User mở browser trên thiết bị mới (thứ 4) → CHẶN, yêu cầu xóa thiết bị cũ
  - Platform: Web-based (browser), KHÔNG cài app

Tại sao giới hạn?
  - Chống chia sẻ tài khoản (1 account cho nhiều người dùng)
  - Đối tác B2B trả phí per-user → chia sẻ = mất doanh thu
  - Bảo vệ nội dung bài thi (đề thi không bị phát tán)
  - Compliance: đối tác yêu cầu đảm bảo đúng người dùng đúng tài khoản
```

### Yêu cầu kỹ thuật

| Tiêu chí | Yêu cầu |
|----------|---------|
| Định danh thiết bị | Nhận biết "đây là thiết bị nào" qua browser |
| Giới hạn | Tối đa 3 devices active per user |
| Stability | Cùng thiết bị → cùng fingerprint (dù mở incognito, xóa cookie) |
| False positive thấp | Không nhầm thiết bị cũ thành thiết bị mới (gây chặn oan) |
| User tự quản lý | User xem danh sách thiết bị, xóa thiết bị cũ |
| Performance | Check device < 100ms, không ảnh hưởng UX |

---

## 2. Device Fingerprint là gì?

### Khái niệm

```
Device Fingerprint = "dấu vân tay" của thiết bị
  → Thu thập nhiều thuộc tính từ browser + phần cứng
  → Kết hợp lại → tạo thành 1 ID gần như unique cho thiết bị đó

Giống vân tay người:
  - Mỗi ngón tay riêng lẻ không unique
  - Nhưng KẾT HỢP 10 ngón → gần như unique

Mỗi tín hiệu riêng lẻ (screen size, GPU...) không unique
  → Nhưng kết hợp 20-30 tín hiệu → unique ~95-99% thiết bị
```

### Tại sao không dùng cookie?

```
Cookie / localStorage:
  ✗ User xóa cookie → mất định danh → thiết bị cũ thành "mới"
  ✗ Incognito mode → không có cookie cũ
  ✗ User chuyển browser (Chrome → Firefox) → cookie khác
  → KHÔNG ĐỦ TIN CẬY làm layer duy nhất

Fingerprint:
  ✓ Không phụ thuộc cookie (dựa vào phần cứng + browser engine)
  ✓ Incognito vẫn cùng fingerprint (canvas, WebGL, CPU, GPU không đổi)
  ✓ Khó giả mạo hơn cookie
  ✗ Không chính xác 100% (browser update có thể thay đổi nhẹ)
  → Dùng KẾT HỢP: fingerprint + device-bound token
```

---

## 3. Các tín hiệu để fingerprint

### Nhóm 1: Phần cứng (stability CAO — ít thay đổi)

| Tín hiệu | Cách lấy | Tại sao stable |
|-----------|---------|----------------|
| **Canvas Fingerprint** | Vẽ text + shapes trên `<canvas>`, hash pixel data | GPU + driver + OS font rendering khác nhau mỗi máy |
| **WebGL Fingerprint** | Query GPU renderer, vendor, extensions | GPU cụ thể cho mỗi máy |
| **AudioContext** | Xử lý audio qua `OfflineAudioContext`, hash output | Audio stack khác nhau mỗi OS + hardware |
| **Hardware Concurrency** | `navigator.hardwareConcurrency` | Số CPU cores — cố định theo máy |
| **Device Memory** | `navigator.deviceMemory` | RAM — cố định theo máy |
| **Max Touch Points** | `navigator.maxTouchPoints` | 0 = desktop, >0 = touch device |
| **Screen Resolution** | `screen.width × screen.height`, `devicePixelRatio` | Cố định theo màn hình (trừ khi gắn monitor ngoài) |

### Nhóm 2: Browser + OS (stability TRUNG BÌNH)

| Tín hiệu | Cách lấy | Lưu ý |
|-----------|---------|-------|
| **User-Agent** | `navigator.userAgent` | Thay đổi khi update browser (Chrome 124 → 125) |
| **Platform** | `navigator.platform` | "Win32", "MacIntel", "Linux x86_64" — stable |
| **Language** | `navigator.language`, `navigator.languages` | User có thể đổi |
| **Timezone** | `Intl.DateTimeFormat().resolvedOptions().timeZone` | Đổi khi đi nước khác |
| **Installed Fonts** | Đo bounding box của text với từng font | Cài thêm font → thay đổi |

### Nhóm 3: Server-side passive (KHÔNG cần JavaScript)

| Tín hiệu | Cách lấy | Lưu ý |
|-----------|---------|-------|
| **TLS Fingerprint (JA3/JA4)** | Hash từ TLS ClientHello parameters | Khác nhau giữa browser + version |
| **HTTP Header Order** | Thứ tự Accept, Accept-Language... | Mỗi browser gửi thứ tự khác |
| **TCP/IP Stack** | TTL, TCP window size | Có thể phân biệt OS |

### Tổng hợp: Signal → Fingerprint

```
Thu thập 20-30 signals
         │
         ▼
Serialize thành JSON:
  {
    "canvas": "a8f3c2...",
    "webgl_renderer": "ANGLE (NVIDIA GeForce GTX 1060)",
    "audio": "b7e4d1...",
    "cpu_cores": 8,
    "memory": 16,
    "screen": "1920x1080",
    "platform": "Win32",
    "timezone": "Asia/Ho_Chi_Minh",
    ...
  }
         │
         ▼
Hash (MurmurHash3 hoặc SHA-256):
  → device_fingerprint = "fp_3a7f2b9e1c..."

Cùng máy, cùng browser → cùng hash (đa số trường hợp)
Khác máy → khác hash (GPU, CPU, screen khác)
```

---

## 4. Kiến trúc tổng thể

### Defense in Depth — 3 layers định danh

```
Không dựa vào 1 layer duy nhất. Kết hợp 3 layers:

Layer 1 (primary):    Device-Bound Token
  → Server cấp UUID, lưu trong HttpOnly cookie
  → Ổn định nhất, chính xác nhất
  → Nhược: user xóa cookie → mất

Layer 2 (secondary):  Browser Fingerprint
  → Thu thập signals, HASH SERVER-SIDE (client không tự hash)
  → Nhận diện lại thiết bị khi cookie bị xóa
  → Nhược: browser update có thể thay đổi nhẹ

Layer 3 (passive):    TLS/HTTP Fingerprint (server-side)
  → Server tự thu thập từ connection, không cần JS
  → Bổ sung thêm confidence, không dùng độc lập
```

### Tại sao hash server-side?

```
Client-side hash (JS tự hash rồi gửi):
  ✗ User intercept → gửi hash giả → bypass device limit
  ✗ Client là untrusted territory

Server-side hash (JS gửi raw signals, server hash):
  ✓ User không biết server hash bằng thuật toán gì
  ✓ Server validate signals (phát hiện giá trị bất thường)
  ✓ Server kết hợp thêm TLS/HTTP passive signals
  → TAMPER-RESISTANT hơn nhiều
```

### Luồng tổng quan

```
┌──────────────┐     ┌──────────────────┐     ┌───────────────────┐
│   Browser     │     │   API Server      │     │   Device Service   │
│               │     │                   │     │                    │
│ 1. Thu thập   │     │                   │     │                    │
│    signals    │────▶│ 2. Nhận signals   │────▶│ 3. Hash signals    │
│    (JS SDK)   │     │    + TLS/HTTP     │     │    (server-side)   │
│               │     │    passive signals│     │                    │
│               │     │                   │     │ 4. Lookup device   │
│               │     │                   │     │    registry        │
│               │     │                   │     │                    │
│               │     │                   │     │ 5. Known device?   │
│               │◀────│◀──────────────────│◀────│    → Allow         │
│               │     │                   │     │    New + count < 3?│
│  6. Allow     │     │                   │     │    → Register      │
│  hoặc Deny   │     │                   │     │    New + count >= 3?│
│               │     │                   │     │    → DENY           │
└──────────────┘     └──────────────────┘     └───────────────────┘
```

---

## 5. Data Model

### user_devices table

```sql
CREATE TABLE user_devices (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL REFERENCES users(id),
    device_id       TEXT NOT NULL,          -- fingerprint hash (server-computed)
    device_token    UUID NOT NULL UNIQUE,   -- device-bound token (stored in cookie)
    device_label    TEXT,                   -- "Chrome on Windows — Home Desktop"
    raw_signals     JSONB,                  -- raw fingerprint signals (cho fuzzy matching)

    is_active       BOOLEAN DEFAULT TRUE,
    first_seen_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_seen_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    -- Metadata
    user_agent      TEXT,
    ip_address      INET,
    platform        TEXT,                   -- "Win32", "MacIntel"
    screen_info     TEXT,                   -- "1920x1080"

    CONSTRAINT uq_user_device UNIQUE (user_id, device_id)
);

CREATE INDEX idx_user_devices_user ON user_devices (user_id) WHERE is_active = TRUE;
```

### Tại sao lưu raw_signals (JSONB)?

```
Chỉ lưu hash (device_id):
  ✗ Browser update → hash đổi → thiết bị cũ thành "mới"
  ✗ Không thể so sánh "giống bao nhiêu %"

Lưu raw_signals:
  ✓ So sánh từng signal → tính similarity score
  ✓ Canvas + WebGL + audio giống 90% → cùng thiết bị (dù user-agent đổi)
  ✓ Debug: xem cụ thể signal nào thay đổi
```

---

## 6. Flow chi tiết: Từ mở browser đến allow/deny

```
User mở browser → vào app PREP
         │
         ▼
┌──────────────────────────────────────────────────────────────────┐
│  BƯỚC 1: Check Device-Bound Token (cookie)                       │
│                                                                   │
│  Có cookie device_token?                                          │
│    → CÓ: gửi token lên server                                    │
│           Server lookup: token tồn tại + active + đúng user?     │
│           → CÓ: ALLOW ngay (fast path, không cần fingerprint)    │
│           → KHÔNG: cookie cũ/invalid → sang BƯỚC 2               │
│    → KHÔNG: chưa đăng ký hoặc xóa cookie → sang BƯỚC 2          │
└──────────────────────┬───────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│  BƯỚC 2: Thu thập Fingerprint signals (JS SDK)                   │
│                                                                   │
│  JS SDK thu thập ~20 signals:                                     │
│    canvas hash, webgl renderer, audio hash,                      │
│    cpu cores, memory, screen, platform, timezone, fonts...       │
│                                                                   │
│  Gửi RAW signals lên server (KHÔNG hash ở client)               │
│  POST /api/v1/device/identify { signals: {...} }                 │
└──────────────────────┬───────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│  BƯỚC 3: Server tính fingerprint + TLS passive signals           │
│                                                                   │
│  Server nhận raw signals + tự thu thập:                          │
│    - TLS fingerprint (JA3/JA4) từ connection                    │
│    - HTTP header order                                           │
│                                                                   │
│  Hash tất cả → device_id = hash(signals + passive)              │
└──────────────────────┬───────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│  BƯỚC 4: Lookup device registry                                  │
│                                                                   │
│  Query: SELECT * FROM user_devices                               │
│         WHERE user_id = :userId AND is_active = TRUE             │
│                                                                   │
│  Case A: device_id EXACT MATCH → thiết bị đã đăng ký            │
│    → Update last_seen_at                                         │
│    → Cấp device_token mới (set cookie)                          │
│    → ALLOW ✓                                                     │
│                                                                   │
│  Case B: Không exact match → FUZZY MATCHING (xem bước 5)        │
└──────────────────────┬───────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│  BƯỚC 5: Fuzzy matching (nếu không exact match)                  │
│                                                                   │
│  So sánh raw_signals mới với raw_signals của các device đã lưu   │
│  Tính similarity score (xem section 7)                           │
│                                                                   │
│  Score >= 80% với 1 device cũ → CÓ THỂ là cùng thiết bị        │
│    (browser update, font thay đổi...)                            │
│    → Update device_id + raw_signals + last_seen_at               │
│    → Cấp device_token mới                                       │
│    → ALLOW ✓                                                     │
│                                                                   │
│  Score < 80% với tất cả device → THIẾT BỊ MỚI → bước 6         │
└──────────────────────┬───────────────────────────────────────────┘
                       │
                       ▼
┌──────────────────────────────────────────────────────────────────┐
│  BƯỚC 6: Check device limit                                      │
│                                                                   │
│  COUNT active devices cho user:                                  │
│    < 3 → ĐĂNG KÝ thiết bị mới                                   │
│           INSERT user_devices (...)                              │
│           Cấp device_token (set cookie)                          │
│           → ALLOW ✓                                              │
│                                                                   │
│    >= 3 → CHẶN                                                   │
│           → DENY ✗                                               │
│           → Trả message: "Bạn đã dùng 3 thiết bị.              │
│              Vui lòng xóa thiết bị cũ để tiếp tục."             │
│           → Redirect đến trang quản lý thiết bị                  │
└──────────────────────────────────────────────────────────────────┘
```

### Tóm tắt flow

| Case | Kết quả | Latency |
|------|---------|---------|
| Cookie device_token valid | ALLOW ngay | < 10ms |
| Cookie miss, fingerprint exact match | ALLOW | < 50ms |
| Cookie miss, fingerprint fuzzy match >= 80% | ALLOW (update fingerprint) | < 100ms |
| Thiết bị mới, count < 3 | Đăng ký + ALLOW | < 100ms |
| Thiết bị mới, count >= 3 | DENY | < 100ms |

---

## 7. Fuzzy Matching — Tại sao không so exact hash?

### Bài toán

```
Thứ 2: User dùng Chrome 124 trên laptop
  → Fingerprint hash: "fp_aaaa1111"
  → Đăng ký device thành công

Thứ 4: Chrome tự update lên 125
  → User-Agent thay đổi: "Chrome/124" → "Chrome/125"
  → Fingerprint hash: "fp_bbbb2222"  ← KHÁC!

Nếu so exact hash:
  → "fp_bbbb2222" != "fp_aaaa1111"
  → Hệ thống nghĩ đây là thiết bị MỚI
  → Nếu user đã có 3 devices → BỊ CHẶN OAN ✗
```

### Giải pháp: So từng signal, tính similarity score

```
Thay vì: hash(all_signals) == hash(all_signals) ?  (binary: match hoặc không)

Làm:     So TỪNG signal, tính % giống nhau

  Signal             Cũ                    Mới                  Match?
  ──────             ───                    ───                  ──────
  canvas_hash        "a8f3c2..."           "a8f3c2..."          ✓ (1.0)
  webgl_renderer     "NVIDIA GTX 1060"     "NVIDIA GTX 1060"    ✓ (1.0)
  audio_hash         "b7e4d1..."           "b7e4d1..."          ✓ (1.0)
  cpu_cores          8                      8                    ✓ (1.0)
  memory             16                     16                   ✓ (1.0)
  screen             "1920x1080"           "1920x1080"          ✓ (1.0)
  platform           "Win32"               "Win32"              ✓ (1.0)
  user_agent         "Chrome/124"          "Chrome/125"         ✗ (0.0)  ← đổi
  timezone           "Asia/Ho_Chi_Minh"    "Asia/Ho_Chi_Minh"   ✓ (1.0)
  fonts              [Arial,Verdana,...]   [Arial,Verdana,...]  ✓ (1.0)

  Similarity = weighted average = 95%  → > 80% threshold → CÙNG THIẾT BỊ ✓
```

### Weighted scoring — Signal stable thì trọng số cao hơn

```
Tín hiệu stable (phần cứng):     weight CAO
  canvas_hash:          0.15
  webgl_renderer:       0.15
  audio_hash:           0.10
  cpu_cores:            0.08
  memory:               0.08
  screen:               0.08
  platform:             0.08

Tín hiệu volatile (hay đổi):     weight THẤP
  user_agent:           0.03   ← đổi mỗi khi update browser
  timezone:             0.05
  language:             0.03
  fonts:                0.05
  ...

Tổng weights = 1.0
Score = SUM(weight × match) / SUM(weight)
```

**Kết quả**: Chrome update (user_agent đổi) → score chỉ giảm 0.03 → vẫn > 80% → cùng thiết bị.

### Threshold

| Score | Quyết định |
|-------|-----------|
| >= 90% | Chắc chắn cùng thiết bị → auto match |
| 80-90% | Có thể cùng thiết bị → match + log để review |
| < 80% | Thiết bị khác → coi là mới |

---

## 8. Device-Bound Token — Layer chính cho định danh

### Tại sao token là layer chính, fingerprint là phụ?

```
Device-Bound Token (cookie):
  ✓ Chính xác 100%: server cấp UUID, không trùng
  ✓ Nhanh: lookup by token = O(1)
  ✓ Không phụ thuộc browser/OS version
  ✗ User xóa cookie → mất token

Browser Fingerprint:
  ✓ Khó xóa (dựa vào phần cứng)
  ✓ Incognito vẫn giữ (không phụ thuộc storage)
  ✗ Không chính xác 100% (browser update, gắn monitor ngoài...)
  ✗ Tốn tài nguyên hơn (thu thập signals, tính similarity)

→ Kết hợp:
  Token = định danh chính (99% cases)
  Fingerprint = backup khi token mất (xóa cookie, incognito)
```

### Token hoạt động thế nào?

```
Lần đầu đăng ký device:
  1. Server generate UUID: device_token = "dt_7f3a..."
  2. Lưu vào DB: user_devices { device_token, device_id, user_id }
  3. Set cookie:
     Set-Cookie: device_token=dt_7f3a...;
       HttpOnly;     ← JS không đọc được (chống XSS)
       Secure;       ← chỉ gửi qua HTTPS
       SameSite=Lax; ← chống CSRF
       Max-Age=31536000  ← 1 năm
  4. Mỗi request sau → browser tự gửi cookie → server lookup → ALLOW ngay

Khi user xóa cookie:
  1. Request không có device_token
  2. → Thu thập fingerprint → fuzzy match → tìm lại device cũ
  3. → Cấp device_token MỚI cho cùng device record
  4. → User không bị chặn ✓
```

---

## 9. Thử thách & Edge Cases

### Thử thách 1: Incognito Mode

```
Vấn đề:
  Incognito không có cookie → device_token mất
  → Mỗi lần mở incognito = "không có token"?

Giải pháp:
  Fingerprint vẫn GIỐNG NHAU trong incognito (canvas, WebGL, CPU, GPU
  là thuộc tính phần cứng, không phụ thuộc cookie/storage)
  → Bước 2-5 trong flow: thu thập fingerprint → fuzzy match → tìm lại device
  → ALLOW ✓

Thí sinh mở incognito → bị check fingerprint (chậm hơn 50ms) nhưng không bị chặn.
```

### Thử thách 2: Browser Update thay đổi fingerprint

```
Vấn đề:
  Chrome 124 → 125: user_agent đổi, có thể canvas hash đổi nhẹ
  → Exact hash khác → "thiết bị mới"?

Giải pháp:
  Fuzzy matching (section 7): user_agent weight chỉ 0.03
  → Score giảm từ 100% xuống 97% → vẫn > 80% → cùng thiết bị
  → Update device_id + raw_signals trong DB

  Nếu major OS update (Windows 10 → 11) thay đổi nhiều signals:
  → Score có thể < 80% → coi là thiết bị mới
  → Nhưng device_token (cookie) vẫn còn → ALLOW qua layer 1
  → Chỉ khi XÓA COOKIE + MAJOR UPDATE cùng lúc mới bị nhầm
```

### Thử thách 3: Gắn monitor ngoài → screen thay đổi

```
Vấn đề:
  Laptop: 1920x1080 → gắn monitor: 2560x1440
  → screen_resolution đổi → hash đổi

Giải pháp:
  screen_resolution weight = 0.08 (thấp)
  → Chỉ giảm 0.08 → score vẫn > 80%
  → device_token (cookie) vẫn match → ALLOW qua layer 1

  Ngoài ra: lưu cả primary + secondary screen info
  → Nếu 1 trong 2 match → coi là cùng thiết bị
```

### Thử thách 4: Phòng máy (Lab) — Nhiều user cùng 1 máy

```
Vấn đề:
  30 học sinh dùng chung 10 máy tính trong phòng lab
  → 1 máy có cùng fingerprint cho tất cả user

Giải pháp:
  Device limit là PER USER, không phải per device:
  - User A đăng ký máy Lab01 → đếm 1/3 cho user A
  - User B đăng ký máy Lab01 → đếm 1/3 cho user B
  - Cùng máy nhưng khác user → không conflict

  Fingerprint giống nhau giữa nhiều users → KHÔNG SAO
  Vì check là: "user X có bao nhiêu device", không phải "device X có bao nhiêu user"
```

### Thử thách 5: Anti-fingerprint browser (Brave, Tor)

```
Vấn đề:
  Brave/Tor normalize nhiều signals (canvas trả kết quả uniform)
  → Nhiều user có fingerprint GIỐNG NHAU → collision cao

Giải pháp:
  - Device-bound token (cookie) vẫn là layer chính → ít bị ảnh hưởng
  - Nếu platform yêu cầu: recommend user dùng Chrome/Firefox/Safari
  - Đối với THI: có thể yêu cầu bắt buộc tắt anti-fingerprint
    (đối tác B2B accept được vì đây là môi trường controlled)
```

### Thử thách 6: User cố tình giả mạo fingerprint

```
Vấn đề:
  Extensions như "Canvas Blocker" randomize canvas mỗi lần load
  → Fingerprint thay đổi liên tục → mỗi lần = "thiết bị mới"

Giải pháp:
  1. Detect inconsistency:
     Canvas random → giá trị thay đổi mỗi lần reload trên cùng page
     → Bình thường: canvas giống nhau khi render cùng content
     → Random: khác nhau mỗi lần → FLAG ngay

  2. Device-bound token vẫn hoạt động (cookie không bị ảnh hưởng bởi extension)

  3. Nếu không có token + fingerprint bất thường:
     → Yêu cầu xác thực thêm (OTP, email verify)
     → Log cho admin review
```

---

## 10. Device Management UI

### Cho User

```
Trang "Quản lý thiết bị" (/settings/devices):

  Thiết bị của bạn (2/3):

  ┌──────────────────────────────────────────────────────┐
  │ 💻 Chrome trên Windows                    [Xóa]     │
  │    Đăng ký: 01/03/2026                               │
  │    Lần cuối: Hôm nay, 14:30                          │
  │    Thiết bị hiện tại ✓                               │
  ├──────────────────────────────────────────────────────┤
  │ 📱 Safari trên iPhone                     [Xóa]     │
  │    Đăng ký: 15/03/2026                               │
  │    Lần cuối: Hôm qua, 20:15                          │
  └──────────────────────────────────────────────────────┘

  User nhấn [Xóa] → is_active = FALSE → slot trống → đăng ký thiết bị mới được
```

### Cho Admin (đối tác B2B)

```
Admin dashboard:

  - Xem danh sách user + số device mỗi user
  - Detect: 1 device xuất hiện ở nhiều user → chia sẻ tài khoản?
  - Force remove device cho user (hỗ trợ khi user không tự xóa được)
  - Config: thay đổi max devices (3 → 5 nếu đối tác yêu cầu)
```

---

## 11. Interview Quick Reference

### Elevator Pitch

> "Giới hạn 3 thiết bị dùng defense-in-depth 2 layer: **device-bound token** (HttpOnly cookie) là layer chính — chính xác 100%, nhanh. **Browser fingerprint** (server-side hash) là layer phụ — nhận diện lại thiết bị khi cookie bị xóa. Fingerprint dùng fuzzy matching (weighted similarity score) thay vì exact hash — để browser update không biến thiết bị cũ thành mới. Phòng lab nhiều user cùng máy không conflict vì device limit là per-user."

### "Fingerprint hoạt động thế nào?"

> "Thu thập ~20 signals từ browser: canvas hash, WebGL renderer, audio context, CPU cores, RAM, screen size... Gửi raw signals lên server (không hash client-side để chống tamper). Server kết hợp thêm TLS fingerprint passive, rồi hash thành device_id. Kết hợp 20-30 signals → unique ~95-99% thiết bị."

### "Tại sao fuzzy matching chứ không exact hash?"

> "Browser update thay đổi user-agent → exact hash đổi → thiết bị cũ thành 'mới' → chặn oan. Fuzzy matching so từng signal với weighted score: canvas/WebGL/audio weight cao (stable), user-agent weight thấp (hay đổi). Score >= 80% → cùng thiết bị. Chrome update chỉ giảm 0.03 score → vẫn match."

### "User xóa cookie thì sao?"

> "Cookie mất → không có device_token → fallback sang fingerprint. Thu thập signals → fuzzy match với các device đã đăng ký → tìm lại device cũ → cấp token mới. Incognito mode cũng tương tự: fingerprint vẫn giống (dựa vào phần cứng, không phụ thuộc storage)."

### "Phòng lab nhiều user cùng máy?"

> "Device limit là per-user. User A đăng ký Lab01 → đếm 1/3 cho A. User B đăng ký cùng Lab01 → đếm 1/3 cho B. Không conflict vì check 'user X có bao nhiêu device', không phải 'device X có bao nhiêu user'."

### "Tại sao hash server-side?"

> "Client là untrusted. Nếu hash client-side, user intercept → gửi hash giả → bypass limit. Server hash: user không biết thuật toán, server validate signals + kết hợp TLS passive fingerprint. Tamper-resistant hơn nhiều."

### Bảng quyết định tổng hợp

| Quyết định | Chọn | Tại sao |
|-----------|------|---------|
| Layer chính | Device-bound token (cookie) | Chính xác 100%, nhanh, đơn giản |
| Layer phụ | Browser fingerprint | Backup khi cookie mất, incognito |
| Hash ở đâu | Server-side | Client untrusted, chống tamper |
| So sánh | Fuzzy matching (weighted) | Exact hash → chặn oan khi browser update |
| Threshold | 80% similarity | Cân bằng: không quá lỏng (bypass), không quá chặt (false positive) |
| Max devices | Configurable per tenant | Đối tác B2B khác nhau yêu cầu khác nhau |
| Phòng lab | Per-user limit | Cùng máy khác user → không conflict |
| Anti-spoof | Detect canvas randomization + require verify | Extension giả fingerprint → detect inconsistency |
