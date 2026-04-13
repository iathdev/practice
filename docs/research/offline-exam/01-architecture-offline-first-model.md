# Offline-First Exam — Kiến trúc, Data Model & Sync Protocol

> Hệ thống thi online IELTS/TOEIC nhưng thiết kế theo mô hình offline-first: client hoạt động đầy đủ khi mất mạng, sync khi có kết nối. Bài toán: đảm bảo thí sinh thi liên tục, không mất bài, dù mạng không ổn định.

---

## Mục lục

1. [Bối cảnh & Tại sao Offline-First?](#1-bối-cảnh--tại-sao-offline-first)
2. [Kiến trúc tổng thể — Client-Heavy, Server-Verify](#2-kiến-trúc-tổng-thể--client-heavy-server-verify)
3. [Client Storage — Lưu gì ở local?](#3-client-storage--lưu-gì-ở-local)
4. [Bảo mật dữ liệu local — Đề thi & Đáp án](#4-bảo-mật-dữ-liệu-local--đề-thi--đáp-án)
5. [Exam Lifecycle — Từ tải đề đến nộp bài](#5-exam-lifecycle--từ-tải-đề-đến-nộp-bài)
6. [Sync Protocol — Đồng bộ liên tục](#6-sync-protocol--đồng-bộ-liên-tục)
7. [Checkpoint Service — Điểm chạm](#7-checkpoint-service--điểm-chạm)
8. [Cross-device Sync — Đổi thiết bị giữa chừng](#8-cross-device-sync--đổi-thiết-bị-giữa-chừng)
9. [Data Model](#9-data-model)
10. [Interview Quick Reference](#10-interview-quick-reference)

---

## 1. Bối cảnh & Tại sao Offline-First?

### Dự án

- **Platform**: Thi online IELTS/TOEIC (cùng nền tảng với Anti-Cheat Service)
- **Mô hình**: B2B — bán cho trường học, trung tâm ngoại ngữ
- **Thực tế mạng Việt Nam**: Wifi trường học không ổn định, 4G có dead zone, 5,000 thí sinh cùng kết nối = bandwidth cạn kiệt

### Bài toán

```
Thi online truyền thống (server-first):
  Mỗi thao tác → gọi API → chờ response → tiếp tục
  Mất mạng 3 giây → UI đứng → thí sinh hoang mang
  Mất mạng 30 giây → session timeout → mất bài thi
  5,000 thí sinh cùng lúc → server quá tải → ai cũng lag

  → Trường học gọi: "Hệ thống các bạn lại sập rồi!"
  → Mất hợp đồng B2B

Offline-first:
  Section đang thi tải trước → làm bài hoàn toàn local (Partial Reveal)
  Có mạng → sync đáp án liên tục lên server (debounce 3s + interval 10s)
  Mất mạng 3 giây → thí sinh KHÔNG BIẾT (vẫn làm bài, đáp án tích local)
  Mất mạng 30 phút → vẫn làm bài, sync_queue tích lũy → flush khi có mạng
  Chuyển section → force sync + tải section mới (cần online tại mốc này)

  → Trải nghiệm thi mượt mà bất kể chất lượng mạng
  → Đối tác tin tưởng
```

### Tại sao offline-first phù hợp bài thi?

| Đặc điểm bài thi | Phù hợp offline-first |
|------------------|----------------------|
| Đề thi cố định sau khi bắt đầu | Tải 1 lần, dùng suốt bài |
| Đáp án thuộc về 1 người duy nhất | Không có multi-user conflict |
| Thời gian giới hạn (2-3 giờ) | Local data không cần lưu lâu |
| Read-heavy (đọc đề) + Write-light (chọn đáp án) | Sync payload nhỏ |
| Kết quả không cần real-time | Chấm bài sau khi nộp, không cần instant |

### So sánh: Server-First vs Offline-First

| | Server-First | Offline-First |
|---|---|---|
| **Mất mạng** | UI đứng / session timeout | Thí sinh không bị ảnh hưởng |
| **Latency** | Mỗi thao tác = 1 API call | Mọi thao tác = local (< 1ms) |
| **Server load** | Mỗi click = 1 request | Chỉ sync batch khi có mạng |
| **Phức tạp** | Đơn giản (CRUD) | Sync protocol, conflict resolution, data integrity |
| **Data loss risk** | Server crash = mất data | Client crash = mất local data |
| **Khi nào dùng** | Mạng ổn định, real-time cần thiết | Mạng không ổn định, UX > real-time |

---

## 2. Kiến trúc tổng thể — Client-Heavy, Server-Verify

```
┌──────────────────────────────────────────────────────────────────────┐
│                          EXAM CLIENT (Browser)                        │
│                                                                       │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────────┐ │
│  │  Exam Engine │  │  Answer     │  │  Timer      │  │  Sync      │ │
│  │  (UI + Logic)│  │  Store      │  │  Manager    │  │  Engine    │ │
│  │              │  │  (IndexedDB)│  │  (local)    │  │  (background)││
│  └──────┬───────┘  └──────┬──────┘  └──────┬──────┘  └──────┬─────┘ │
│         │                 │                 │                │       │
│         │   read/write    │    tick/check   │    push/pull   │       │
│         └────────────────►│◄────────────────┘                │       │
│                           │                                  │       │
│                           │         ┌────────────────────────┘       │
│                           │         │ sync khi có mạng               │
│                           ▼         ▼                                │
│                    ┌──────────────────────┐                          │
│                    │   Service Worker      │                         │
│                    │   (offline cache +    │                         │
│                    │    background sync)   │                         │
│                    └──────────┬────────────┘                         │
└───────────────────────────────┼───────────────────────────────────────┘
                                │
                    ════════════╪════════════  Network boundary
                                │
                    ┌───────────▼────────────┐
                    │      API Gateway       │
                    └───────────┬────────────┘
                                │
                    ┌───────────▼────────────┐
                    │     Exam Service       │
                    │  (validate + store)    │
                    └───────────┬────────────┘
                                │
                    ┌───────────▼────────────┐
                    │   PostgreSQL           │
                    │   (source of truth)    │
                    └────────────────────────┘
```

### Nguyên tắc thiết kế

| Nguyên tắc | Giải thích |
|-----------|-----------|
| **Local-first** | Mọi thao tác xử lý local trước, sync lên server sau. UI không bao giờ chờ network |
| **Server = Source of Truth** | Local data là bản nháp. Server xác nhận mới chính thức. Chấm bài từ server data |
| **Optimistic UI** | Chọn đáp án → lưu local ngay → UI phản hồi ngay. Sync chạy background |
| **Idempotent sync** | Cùng đáp án gửi 2 lần → server xử lý 1 lần. Retry an toàn |
| **Graceful degradation** | Mất mạng → vẫn thi. Mất local storage → có server backup. Mất cả 2 → recovery protocol |

### 4 thành phần chính trên Client

| Component | Trách nhiệm | Khi offline |
|-----------|-------------|-------------|
| **Exam Engine** | Render đề thi, xử lý tương tác | Hoạt động bình thường (data đã có local) |
| **Answer Store** | Lưu đáp án vào IndexedDB | Hoạt động bình thường |
| **Timer Manager** | Đếm ngược thời gian thi | Hoạt động bình thường (local clock) |
| **Sync Engine** | Đồng bộ đáp án lên server | Tạm dừng, queue changes, resume khi có mạng |

---

## 3. Client Storage — Lưu gì ở local?

### Tại sao IndexedDB?

| Storage | Dung lượng | Async | Structured Query | Phù hợp? |
|---------|-----------|-------|-----------------|-----------|
| localStorage | 5-10 MB | Sync (block UI) | Không | Quá nhỏ, block UI |
| sessionStorage | 5-10 MB | Sync | Không | Mất khi đóng tab |
| **IndexedDB** | **50 MB - unlimited** | **Async** | **Có (index, cursor)** | **Phù hợp nhất** |
| Cache API | Unlimited | Async | Không (key-value) | Cho static assets, không cho structured data |

### Data cần lưu local

```
IndexedDB: "exam_db"
│
├── Object Store: "exam_content"
│   └── Section đang thi (Partial Reveal — không tải toàn bộ đề)
│   └── Tải khi bắt đầu section, xóa khi section kết thúc
│   └── ~0.5-1 MB per section (text + metadata)
│
├── Object Store: "answers"
│   └── Đáp án thí sinh đã chọn/nhập
│   └── Mỗi thay đổi → write ngay
│   └── ~10-50 KB (nhỏ, write thường xuyên)
│
├── Object Store: "sync_queue"
│   └── Queue các changes chưa sync lên server
│   └── Mỗi entry = 1 answer change + metadata
│   └── Sync thành công → xóa khỏi queue
│
├── Object Store: "exam_state"
│   └── Trạng thái hiện tại: section, question index, timer remaining
│   └── Để recover khi refresh/crash
│   └── ~1 KB
│
└── Object Store: "media_cache"
    └── Audio files cho Listening section (pre-downloaded)
    └── ~20-50 MB cho IELTS Listening
```

### Service Worker — Cache static assets

```
Service Worker cache (Cache API):
  - Exam UI (HTML, CSS, JS) → app hoạt động offline
  - Font files, icons
  - KHÔNG cache exam content (dùng IndexedDB, structured + queryable)

Tại sao tách Cache API và IndexedDB?
  - Cache API: cho HTTP responses (assets, HTML) → URL-based lookup
  - IndexedDB: cho structured data (answers, exam content) → query by index
  - Dùng đúng công cụ cho đúng loại data
```

### Pre-download Listening Audio

```
Khi bắt đầu thi:
  1. Tải exam content (text) → IndexedDB
  2. Tải audio files → IndexedDB media_cache (blob)
  3. Khi cần play → tạo Object URL từ blob
  4. Nếu audio chưa tải xong khi đến Listening section
     → Hiện progress bar "Đang tải audio..."
     → Nếu offline → "Audio chưa sẵn sàng. Vui lòng kiểm tra kết nối."

Pre-download quan trọng vì:
  - Audio file 20-50 MB → không thể stream khi mạng yếu
  - Listening section có time limit → không thể chờ buffer
  - Tải trước khi bắt đầu Listening → trải nghiệm mượt mà
```

---

## 4. Bảo mật dữ liệu local — Đề thi & Đáp án

### Vấn đề

```
IndexedDB đọc được qua DevTools (F12 → Application → IndexedDB):

  exam_content → thí sinh xem trước câu hỏi section chưa tới
  answers      → người khác ngồi cạnh đọc đáp án đã chọn

Tải toàn bộ đề trước khi thi:
  → Kỳ thi nhiều ca (sáng/chiều cùng đề)
  → Thí sinh ca sáng thi xong → chia sẻ đề cho ca chiều
  → Đề bị lộ trước khi ca chiều bắt đầu
```

### Giải pháp 1: Server-side Content Lock — Server kiểm soát nội dung

Bảo mật thật sự phải đến từ **server**, không phải client. Client chạy trên máy thí sinh — mọi thao tác client-side đều có thể bị can thiệp.

**Nguyên tắc:** Server chỉ trả nội dung section khi session đang ở đúng section đó. Client không thể lấy section 2 trước khi hoàn thành section 1, dù có cố tình gọi API.

```
Server kiểm tra mỗi request lấy content:
  GET /exams/{id}/sections/2/content
    → Server check: session.current_section == 2?
    → Không → 403 Forbidden (dù token hợp lệ)
    → Có → trả content

Session chỉ advance khi checkpoint confirmed:
  POST /exams/{id}/checkpoint { from: section_1, to: section_2 }
    → Server: confirm sync + update session.current_section = 2
    → Từ đây mới có thể lấy section 2 content
```

**Trade-off:**

| | Tải toàn bộ | Server-side Lock |
|---|---|---|
| Bảo mật đề | Thấp — toàn bộ đề trong IndexedDB | Cao — server từ chối cấp content sai section |
| Offline capability | Thi hoàn toàn offline | Cần online tại mốc chuyển section |
| UX | Mượt mà | Delay tại mốc chuyển section nếu mất mạng |

**Mất mạng khi chuyển section:**

```
Mất mạng TRONG section → vẫn làm bài bình thường (content đã có local)
Mất mạng KHI CHUYỂN section → block tại đây:
  → Hiện: "Đang đồng bộ và tải section tiếp theo..."
  → Retry tự động khi có mạng
  → Pre-fetch chạy ngầm khi còn 5 phút trong section hiện tại
     → giảm thiểu trường hợp phải chờ
```

---

### Giải pháp 2: Encrypt answers trong IndexedDB

Bảo vệ đáp án khỏi **casual inspection** (người ngồi cạnh mở DevTools). Không phải bảo mật tuyệt đối — key vẫn có thể bị lấy từ memory nếu attacker đủ quyết tâm.

Đáp án lưu local dạng mã hóa — đọc qua DevTools chỉ thấy ciphertext.

```
Session key = derive từ exam token (JWT):
  key = HKDF(jwt_secret_portion, exam_id + user_id)
  → Key chỉ tồn tại trong JS memory (biến), KHÔNG lưu vào IndexedDB
  → Đóng tab / refresh → key mất → cần re-derive từ token

Ghi đáp án:
  answer_plaintext → AES-GCM encrypt (session key) → lưu IndexedDB

Đọc đáp án:
  ciphertext trong IndexedDB → AES-GCM decrypt (session key) → answer_plaintext
```

**Khi refresh / reopen tab:**

```
IndexedDB vẫn còn ciphertext (không mất)
Session key mất (chỉ ở memory)
  → Re-derive key từ exam token (vẫn còn hiệu lực)
  → Decrypt lại đáp án → resume bình thường
```

**Giới hạn:**

```
Encrypt IndexedDB không chống được:
  ✗ Thí sinh tự build tool để hook vào JS runtime → đọc key từ memory
  ✗ Attacker có full OS access (malware)

Chống được:
  ✓ Người ngồi cạnh mở DevTools → chỉ thấy ciphertext
  ✓ Dump IndexedDB file trực tiếp từ disk → vô nghĩa không có key
  ✓ Extension độc hại đọc storage → không decrypt được
```

---

### Tóm tắt

| Mối đe dọa | Giải pháp | Mức độ bảo vệ |
|---|---|---|
| Xem trước câu hỏi section sau | Server lock — chỉ cấp content khi đúng section | Mạnh — server từ chối, không bypass được |
| Lộ đề giữa các ca thi | Server lock theo session state + time-lock per section | Mạnh — phải có valid session đang ở đúng section |
| Đọc đáp án qua DevTools | Encrypt IndexedDB bằng session key in-memory | Trung bình — chặn casual, không chặn attacker quyết tâm |
| Lấy đáp án từ disk | Encrypt — key không lưu local | Trung bình — không decrypt được nếu không có token |
| Chụp màn hình / ghi chép tay | Không thể ngăn bằng kỹ thuật | Cần camera proctoring |

---

## 5. Exam Lifecycle — Từ tải đề đến nộp bài

```
Phase 1: AUTHENTICATE & PREPARE
═══════════════════════════════════════
1. Thí sinh đăng nhập → server verify → issue exam token (JWT)
2. Token chứa: user_id, exam_id, start_time, duration, exam_version
3. ★ BẮT BUỘC ONLINE ★ — không thể bắt đầu thi khi offline

Phase 2: DOWNLOAD SECTION HIỆN TẠI (online required — Partial Reveal)
═══════════════════════════════════════
4. GET /exams/{id}/sections/current → nội dung section 1 (câu hỏi, choices)
5. Lưu exam_content vào IndexedDB (chỉ section hiện tại)
6. Download audio files section này → IndexedDB media_cache
7. Lưu exam_state: { status: READY, current_section: 1, downloaded_at, version }
8. Verify: checksum section content == server checksum
9. Pre-fetch section tiếp theo khi còn 5 phút (tải ngầm)
10. ★ Từ đây trở đi, client có thể hoạt động OFFLINE trong section này ★

Phase 3: EXAM IN PROGRESS (offline-capable trong section)
═══════════════════════════════════════
11. Thí sinh bắt đầu làm bài
12. Mỗi lần chọn/thay đổi đáp án:
    a. Write → IndexedDB "answers" (encrypt, immediate)
    b. Push → IndexedDB "sync_queue" (pending sync)
    c. UI phản hồi ngay (optimistic)
13. Timer đếm ngược local (interval 1 giây)
14. Sync Engine (background — LIÊN TỤC khi có mạng):
    a. Check online status (navigator.onLine + heartbeat)
    b. Online → trigger: debounce 3s sau answer change, hoặc interval 10s
    c. Pop sync_queue → POST /exams/{id}/sync (batch)
    d. Server confirm → remove from sync_queue
    e. Offline → đáp án tích lũy trong queue, flush ngay khi online lại
    ★ Không phải "xong mới gửi" — sync chạy liên tục khi có mạng ★

Phase 3b: CHECKPOINT tại điểm chạm (next page / next part / next section)
═══════════════════════════════════════
15. Thí sinh nhấn Next Page / Next Part:
    a. Force sync sync_queue ngay (không chờ debounce/interval)
    b. Server log checkpoint snapshot
    c. Không block navigation — cho qua ngay, sync chạy background
16. Thí sinh nhấn Next Section:
    a. Force sync sync_queue (blocking — chờ confirm)
    b. Server log checkpoint + trả nội dung section mới
    c. Client lưu section mới vào IndexedDB → navigate
    d. Nếu offline → retry, hiện "Đang tải section tiếp theo..."

Phase 4: SUBMIT (online required — with grace period)
═══════════════════════════════════════
14. Timer hết HOẶC thí sinh nhấn "Nộp bài"
15. Final sync: flush toàn bộ sync_queue lên server
16. POST /exams/{id}/submit { final_answers, client_timer_remaining }
17. Server verify: tất cả answers đã nhận, exam chưa quá thời gian
18. Server confirm: { status: SUBMITTED, server_received_at }
19. Xóa local data (IndexedDB cleanup)

    Nếu offline khi submit:
    → Lưu submit intent vào IndexedDB
    → Hiện: "Bài thi đã lưu. Đang chờ kết nối để nộp..."
    → Khi online → auto-submit
    → Grace period: chấp nhận submit muộn trong 5 phút sau deadline
```

### Tại sao Phase 1 & 2 bắt buộc online?

```
Phase 1 (Auth):
  - Phải verify identity trên server → không thể offline
  - Token phải fresh → không dùng cached token
  - Nếu cho phép cached auth → risk: token expired, user bị ban

Phase 2 (Download):
  - Đề thi phải lấy từ server → không cache đề cũ
  - Mỗi bài thi có version → đảm bảo đúng đề
  - Checksum verify → đề thi không bị corrupt

Phase 3-4 (Exam + Submit):
  - Offline-capable → UX mượt mà
  - Submit cần online nhưng có grace period
```

### State Diagram — Exam trên Client

```
              Download OK
                  │
                  ▼
            ┌──────────┐
            │   READY   │ ── Đề thi đã tải, chưa bắt đầu
            └─────┬─────┘
                  │ Thí sinh nhấn "Bắt đầu"
                  ▼
            ┌──────────────┐
            │ IN_PROGRESS   │ ── Đang làm bài (offline-capable)
            └─────┬────────┘
                  │
        ┌─────────┼──────────┐
        │         │          │
        ▼         ▼          ▼
  ┌──────────┐ ┌─────────┐ ┌──────────────┐
  │SUBMITTING│ │TIME_UP  │ │ INTERRUPTED  │
  │(user nộp)│ │(hết giờ)│ │(crash/close) │
  └────┬─────┘ └────┬────┘ └──────┬───────┘
       │             │             │
       │      sync + submit        │ reopen browser
       │             │             │ → recover from IndexedDB
       ▼             ▼             ▼
  ┌──────────┐                ┌──────────────┐
  │SUBMITTED │                │ IN_PROGRESS   │ (resume)
  │(nộp OK)  │                └──────────────┘
  └──────────┘
```

---

## 6. Sync Protocol — Đồng bộ liên tục

### Mô hình sync đúng

```
Có mạng:   answer change → queue → debounce 3s → batch sync → server confirm → xóa queue
                                 hoặc interval 10s ──────────────────────────────────↑
                                 hoặc checkpoint force sync ────────────────────────↑

Mất mạng:  answer change → queue tích lũy (không mất)
Có mạng lại: flush toàn bộ queue ngay lập tức
Nộp bài:   final flush những gì còn sót trong queue
```

**KHÔNG phải:** "làm xong → nộp → mới gửi đáp án lên server"
**ĐÚNG là:** sync chạy liên tục background, nộp bài chỉ là final flush

### Sync Queue Model

```
Mỗi thay đổi đáp án tạo 1 sync entry:

{
  "sync_id": "uuid-v4",           // unique ID cho idempotency
  "exam_id": "ielts_mock_0056",
  "question_id": "reading_q12",
  "answer": "B",                   // hoặc text cho writing
  "answered_at": 1712400120000,    // client timestamp (ms)
  "sequence": 47,                  // monotonic counter per session
  "status": "pending"              // pending | synced | failed
}
```

### Sync Strategy: Batch + Debounce

```
Không sync từng answer riêng lẻ:
  - 40 câu reading → 40 API calls → lãng phí
  - Thí sinh thay đổi đáp án liên tục → spam server

Batch + debounce:
  1. Thí sinh chọn đáp án → push vào sync_queue (local)
  2. Debounce 3 giây: nếu không có thay đổi mới → trigger sync
  3. Sync: gom tất cả pending entries → 1 API call
  4. Hoặc: mỗi 10 giây sync 1 lần (interval), bất kể debounce
  5. Hoặc: khi chuyển section → force sync ngay

Kết hợp 3 trigger:
  ┌─────────────────────────────────────────────────┐
  │  Trigger 1: Debounce 3s sau answer change       │
  │  Trigger 2: Interval mỗi 10 giây                │
  │  Trigger 3: Force sync khi chuyển section/nộp bài│
  │                                                   │
  │  → Lấy trigger nào đến trước → batch sync        │
  └─────────────────────────────────────────────────┘
```

### Sync API

```
POST /api/v1/exams/{examId}/sync
Authorization: Bearer {exam_token}

Request:
{
  "session_id": "sess_abc",
  "answers": [
    {
      "sync_id": "uuid-1",
      "question_id": "reading_q12",
      "answer": "B",
      "answered_at": 1712400120000,
      "sequence": 47
    },
    {
      "sync_id": "uuid-2",
      "question_id": "reading_q13",
      "answer": "C",
      "answered_at": 1712400125000,
      "sequence": 48
    }
  ],
  "client_timer_remaining_ms": 3542000,
  "client_timestamp": 1712400130000
}

Response:
{
  "accepted": ["uuid-1", "uuid-2"],
  "rejected": [],
  "server_timestamp": 1712400130500,
  "server_timer_remaining_ms": 3541500
}
```

### Sau khi sync thành công

```
1. Server trả accepted[] → client xóa entries khỏi sync_queue
2. Server trả rejected[] → client giữ lại, log lý do reject
3. Server trả server_timer_remaining_ms → client điều chỉnh timer nếu drift > 2 giây
4. Nếu network fail → entries ở lại sync_queue → retry lần sau
```

### Sync khi offline → online (reconnect)

```
navigator.onLine = false (hoặc heartbeat fail)
  → Sync Engine tạm dừng
  → Đáp án tích lũy trong sync_queue
  → Thí sinh vẫn làm bài bình thường

navigator.onLine = true (hoặc heartbeat OK)
  → Sync Engine resume
  → Gom TẤT CẢ pending entries → batch sync
  → Nếu queue lớn (> 50 entries) → chia thành chunks, sync lần lượt
  → Mỗi chunk sync thành công → xóa khỏi queue
```

---

## 7. Checkpoint Service — Điểm chạm

### Điểm chạm là gì?

Mỗi khi thí sinh chuyển **next page / next part / next section**, hệ thống thực hiện:
1. Force sync đáp án pending lên server
2. Server log checkpoint snapshot (đáp án tại thời điểm đó)
3. Server trả nội dung tiếp theo (với next section — Partial Reveal)

### Hành vi tại từng loại checkpoint

| Checkpoint | Force sync | Block navigation? | Server action |
|---|---|---|---|
| **Next Page** | Có (fire-and-forget) | Không — cho qua ngay | Log checkpoint |
| **Next Part** | Có (fire-and-forget) | Không — cho qua ngay | Log checkpoint |
| **Next Section** | Có (blocking) | Có — chờ confirm + tải section mới | Log checkpoint + trả section content |

**Tại sao Next Page / Part không block?**
```
Sync queue vẫn chạy background → đáp án không mất
Block navigation → UX tệ → thí sinh bực bội
Checkpoint chỉ cần đảm bảo: server có snapshot tại mốc đó, không cần real-time
```

**Tại sao Next Section block?**
```
Cần tải nội dung section mới (Partial Reveal) → bắt buộc online
Force sync trước khi tải → đảm bảo server có đủ đáp án section vừa xong
→ Nếu client crash sau đó → server đã có full data của section đã làm
```

### Flow Next Section chi tiết

```
Thí sinh nhấn "Next Section":
  1. Client: force flush sync_queue → POST /exams/{id}/sync
  2. Server: confirm sync + log checkpoint + trả section_content mới
  3. Client: lưu section_content mới vào IndexedDB (xóa section cũ)
  4. Client: navigate sang section mới

Nếu offline khi nhấn Next Section:
  → Hiện: "Đang đồng bộ và tải section tiếp theo..."
  → Retry tự động khi có mạng
  → Pre-fetch đã chạy trước đó (còn 5 phút) → section mới có thể đã sẵn sàng
```

### Checkpoint API

```
POST /api/v1/exams/{examId}/checkpoint
Authorization: Bearer {exam_token}

Request:
{
  "session_id": "sess_abc",
  "checkpoint_type": "next_section",   // next_page | next_part | next_section
  "from": "section_1",
  "to": "section_2",
  "answers": [...],                    // pending answers chưa sync
  "client_timestamp": 1712400200000
}

Response:
{
  "checkpoint_id": "cp_uuid",
  "accepted_answers": ["uuid-1", ...],
  "section_content": { ... },          // chỉ có khi checkpoint_type = next_section
  "server_timestamp": 1712400200500
}
```

---

## 8. Cross-device Sync — Đổi thiết bị giữa chừng

### Bài toán

```
Thí sinh đang thi trên máy A → máy A hỏng / hết pin giữa chừng
→ Chuyển sang máy B
→ Máy B không có IndexedDB của bài thi đang dở
→ Cần restore state từ server → tiếp tục thi
```

### Luồng Resume trên thiết bị mới

```
Máy B:
  1. Đăng nhập → server verify → phát hiện session đang IN_PROGRESS
  2. Server trả: "Bạn đang có bài thi dở. Tiếp tục không?"
  3. Thí sinh xác nhận → GET /exams/{id}/resume
  4. Server trả:
     - exam_state: { current_section, current_question, timer_remaining }
     - latest_answers: đáp án mới nhất của từng câu (từ server DB)
     - section_content: nội dung section hiện tại (Partial Reveal)
  5. Client hydrate IndexedDB từ server data
  6. Re-derive session key từ exam token mới → decrypt/encrypt lại answers
  7. Resume bài thi từ đúng vị trí
```

### Sync 2 chiều

| Chiều | Khi nào | Cơ chế |
|---|---|---|
| **Local → Server** | Liên tục khi có mạng (debounce 3s + interval 10s + checkpoint) | POST /sync batch |
| **Server → Local** | Khi mở máy mới / refresh / resume sau crash | GET /resume hydrate IndexedDB |

### Edge case: máy A vẫn còn sync_queue chưa flush

```
Máy A crash → sync_queue còn 5 entries chưa gửi lên server
Máy B resume → server chỉ có answers đã sync trước đó

Giải pháp:
  → Server dùng latest checkpoint snapshot làm baseline
  → 5 entries chưa sync = mất (chấp nhận được — window nhỏ)
  → Nếu muốn zero loss: máy A cần flush trước khi crash (không đảm bảo được)
  → Checkpoint tại next page/part giúp giảm window mất data xuống tối thiểu
```

### Resume API

```
GET /api/v1/exams/{examId}/resume
Authorization: Bearer {exam_token}

Response:
{
  "session_id": "sess_abc",
  "status": "IN_PROGRESS",
  "current_section": 2,
  "current_question": 15,
  "timer_remaining_ms": 2145000,
  "latest_answers": [
    { "question_id": "reading_q1", "answer": "B", "sequence": 12 },
    ...
  ],
  "section_content": { ... },
  "last_checkpoint": { "type": "next_part", "at": "2026-04-11T09:30:00Z" }
}
```

---

## 9. Data Model

### Server-side

```sql
-- Exam session: mỗi lần thí sinh thi
CREATE TABLE exam_sessions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         UUID NOT NULL,
    exam_id         UUID NOT NULL,
    status          TEXT NOT NULL DEFAULT 'READY',
        -- READY | IN_PROGRESS | SUBMITTED | EXPIRED | VOIDED
    started_at      TIMESTAMPTZ,
    deadline_at     TIMESTAMPTZ,        -- started_at + duration
    submitted_at    TIMESTAMPTZ,
    exam_version    INT NOT NULL,       -- version đề thi
    client_info     JSONB,              -- device, browser, IP khi bắt đầu
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE (user_id, exam_id)           -- 1 user chỉ thi 1 lần per exam
);

-- Đáp án: server lưu mọi answer đã sync
CREATE TABLE exam_answers (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES exam_sessions(id),
    question_id     TEXT NOT NULL,
    answer          TEXT NOT NULL,       -- selected choice hoặc written text
    answered_at     TIMESTAMPTZ NOT NULL, -- client timestamp
    sequence        INT NOT NULL,        -- monotonic order từ client
    sync_id         UUID NOT NULL,       -- idempotency key từ client
    synced_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    UNIQUE (session_id, sync_id)        -- idempotent: cùng sync_id chỉ insert 1 lần
);

-- Chỉ giữ đáp án MỚI NHẤT per question (view cho chấm bài)
CREATE VIEW latest_answers AS
SELECT DISTINCT ON (session_id, question_id)
    session_id, question_id, answer, answered_at, sequence
FROM exam_answers
ORDER BY session_id, question_id, sequence DESC;

-- Sync log: track mọi lần client sync (audit trail)
CREATE TABLE sync_logs (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id      UUID NOT NULL REFERENCES exam_sessions(id),
    answers_count   INT NOT NULL,
    client_timestamp TIMESTAMPTZ NOT NULL,
    server_timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    client_timer_remaining_ms BIGINT,
    ip              INET,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Checkpoint log: snapshot tại mỗi điểm chạm
CREATE TABLE exam_checkpoints (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id       UUID NOT NULL REFERENCES exam_sessions(id),
    checkpoint_type  TEXT NOT NULL,     -- next_page | next_part | next_section
    from_ref         TEXT NOT NULL,     -- page/part/section vừa rời
    to_ref           TEXT NOT NULL,     -- page/part/section sắp vào
    answers_count    INT NOT NULL,      -- số answers đã sync tại thời điểm này
    answers_snapshot JSONB,             -- snapshot đáp án tại mốc (latest per question)
    client_timestamp TIMESTAMPTZ NOT NULL,
    server_timestamp TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### Tại sao lưu MỌI answer change, không chỉ answer cuối?

```
Chỉ lưu answer cuối:
  ✗ Mất audit trail (thí sinh đổi đáp án bao nhiêu lần?)
  ✗ Không detect được pattern bất thường (đổi hàng loạt đáp án trong 10 giây cuối)
  ✗ Anti-cheat mất signal quan trọng
  ✗ Nếu answer cuối bị corrupt → không recover được

Lưu mọi change:
  ✓ Audit trail đầy đủ
  ✓ Anti-cheat: phân tích answer change pattern
  ✓ Recovery: nếu answer cuối lỗi → lấy answer trước đó
  ✓ Chấm bài: dùng latest_answers view → đáp án mới nhất per question
  ✓ Storage cost: rất nhỏ (mỗi answer change ~200 bytes)
```

### Client-side (IndexedDB schema)

```javascript
// IndexedDB schema
const DB_NAME = "exam_db";
const DB_VERSION = 1;

// Object stores
{
  "exam_content": {
    keyPath: "exam_id",
    // { exam_id, version, sections: [...], questions: [...], checksum }
  },
  "answers": {
    keyPath: "question_id",
    indexes: ["section_type", "answered_at"],
    // { question_id, answer, answered_at, section_type }
  },
  "sync_queue": {
    keyPath: "sync_id",
    indexes: ["status", "sequence"],
    // { sync_id, question_id, answer, answered_at, sequence, status }
  },
  "exam_state": {
    keyPath: "key",
    // { key: "current", status, current_section, current_question,
    //   timer_remaining_ms, last_updated_at }
  },
  "media_cache": {
    keyPath: "url",
    // { url, blob, content_type, size, downloaded_at }
  }
}
```

---

## 10. Interview Quick Reference

### Elevator Pitch (30 giây)

> "Hệ thống thi online theo mô hình offline-first: đề thi tải trước vào IndexedDB, thí sinh làm bài hoàn toàn local — mất mạng không ảnh hưởng. Đáp án sync lên server qua batch API mỗi 10 giây khi có mạng. Server là source of truth, chấm bài từ server data. Sync idempotent (sync_id unique), lưu mọi answer change cho audit trail."

### "Tại sao offline-first cho bài thi?"

> "Thực tế mạng VN: wifi trường học không ổn định, 5,000 thí sinh cùng kết nối = bandwidth cạn kiệt. Server-first → mất mạng 3 giây = UI đứng. Offline-first → mất mạng 30 phút thí sinh vẫn làm bài bình thường. Bài thi phù hợp offline-first vì: đề cố định, đáp án thuộc 1 người (không multi-user conflict), payload nhỏ."

### "Sync protocol hoạt động thế nào?"

> "Mỗi answer change push vào sync_queue (IndexedDB). Sync trigger bởi 3 cơ chế: debounce 3s, interval 10s, hoặc force khi chuyển section. Batch POST lên server, mỗi entry có sync_id (UUID) cho idempotency. Server confirm → xóa khỏi queue. Offline → queue tích lũy, online lại → flush tất cả."

### "Data lưu ở đâu?"

> "Client: IndexedDB — exam content, answers, sync queue, exam state, media cache. Server: PostgreSQL — exam_sessions, exam_answers (mọi change, không chỉ cuối cùng), sync_logs. Chấm bài dùng latest_answers view (DISTINCT ON question_id, ORDER BY sequence DESC)."

### "Offline khi nộp bài thì sao?"

> "Lưu submit intent vào IndexedDB. Hiện thông báo 'đang chờ kết nối'. Auto-submit khi online. Grace period 5 phút sau deadline — server chấp nhận nộp muộn trong window này. Quá grace period → server dùng answers đã sync trước đó để chấm."

### "Sync hoạt động thế nào — có phải xong mới gửi không?"

> "Không. Sync chạy liên tục khi có mạng: debounce 3s sau answer change, hoặc interval 10s. Mất mạng → queue tích lũy, flush ngay khi online lại. Nộp bài chỉ là final flush những gì còn sót. Ngoài ra có checkpoint force sync tại next page/part/section."

### "Checkpoint service là gì?"

> "Tại mỗi điểm chạm next page/part/section, client force sync toàn bộ pending answers. Next page/part: non-blocking, cho qua ngay. Next section: blocking vì cần tải nội dung section mới (Partial Reveal). Server log checkpoint snapshot — đảm bảo có đủ data dù client crash sau đó."

### "Thí sinh đổi thiết bị giữa chừng thì sao?"

> "GET /exams/{id}/resume — server trả exam_state + latest_answers + section_content hiện tại. Client hydrate IndexedDB từ data này, re-derive session key, resume đúng vị trí. Answers chưa flush trước khi crash có thể mất (window nhỏ), checkpoint tại next page/part giúp minimize mất mát."

### "Tại sao không tải toàn bộ đề từ đầu?"

> "Bảo mật đề thi: nếu tải toàn bộ, thí sinh ca sáng có thể chia sẻ đề cho ca chiều qua IndexedDB. Partial Reveal: chỉ tải section đang thi, pre-fetch section sau khi còn 5 phút. Trade-off: cần online tại mốc chuyển section — acceptable vì không cần online liên tục."
