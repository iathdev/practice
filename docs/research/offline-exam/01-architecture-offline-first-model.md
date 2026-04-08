# Offline-First Exam — Kiến trúc, Data Model & Sync Protocol

> Hệ thống thi online IELTS/TOEIC nhưng thiết kế theo mô hình offline-first: client hoạt động đầy đủ khi mất mạng, sync khi có kết nối. Bài toán: đảm bảo thí sinh thi liên tục, không mất bài, dù mạng không ổn định.

---

## Mục lục

1. [Bối cảnh & Tại sao Offline-First?](#1-bối-cảnh--tại-sao-offline-first)
2. [Kiến trúc tổng thể — Client-Heavy, Server-Verify](#2-kiến-trúc-tổng-thể--client-heavy-server-verify)
3. [Client Storage — Lưu gì ở local?](#3-client-storage--lưu-gì-ở-local)
4. [Exam Lifecycle — Từ tải đề đến nộp bài](#4-exam-lifecycle--từ-tải-đề-đến-nộp-bài)
5. [Sync Protocol — Đồng bộ đáp án lên server](#5-sync-protocol--đồng-bộ-đáp-án-lên-server)
6. [Data Model](#6-data-model)
7. [Interview Quick Reference](#7-interview-quick-reference)

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
  Toàn bộ đề thi tải trước → làm bài hoàn toàn local
  Mất mạng 3 giây → thí sinh KHÔNG BIẾT (vẫn làm bài bình thường)
  Mất mạng 30 phút → vẫn làm bài, đáp án lưu local
  Có mạng lại → sync đáp án lên server (background)

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
│   └── Toàn bộ đề thi (câu hỏi, đáp án choices, audio URLs, images)
│   └── Tải 1 lần khi bắt đầu thi
│   └── ~2-5 MB cho 1 bài thi IELTS (text + metadata)
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

## 4. Exam Lifecycle — Từ tải đề đến nộp bài

```
Phase 1: AUTHENTICATE & PREPARE
═══════════════════════════════════════
1. Thí sinh đăng nhập → server verify → issue exam token (JWT)
2. Token chứa: user_id, exam_id, start_time, duration, exam_version
3. ★ BẮT BUỘC ONLINE ★ — không thể bắt đầu thi khi offline

Phase 2: DOWNLOAD EXAM (online required)
═══════════════════════════════════════
4. GET /exams/{id}/content → exam content (câu hỏi, choices, audio refs)
5. Lưu exam_content vào IndexedDB
6. Download audio files → IndexedDB media_cache
7. Lưu exam_state: { status: READY, downloaded_at, version }
8. Verify: checksum exam content == server checksum
   → Đảm bảo đề thi không bị corrupt khi download
9. ★ Từ đây trở đi, client có thể hoạt động OFFLINE ★

Phase 3: EXAM IN PROGRESS (offline-capable)
═══════════════════════════════════════
10. Thí sinh bắt đầu làm bài
11. Mỗi lần chọn/thay đổi đáp án:
    a. Write → IndexedDB "answers" (immediate)
    b. Push → IndexedDB "sync_queue" (pending sync)
    c. UI phản hồi ngay (optimistic)
12. Timer đếm ngược local (interval 1 giây)
13. Sync Engine (background):
    a. Check online status (navigator.onLine + heartbeat)
    b. Online → pop sync_queue → POST /exams/{id}/answers/batch
    c. Server confirm → remove from sync_queue
    d. Offline → giữ trong queue, retry khi online

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

## 5. Sync Protocol — Đồng bộ đáp án lên server

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

## 6. Data Model

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

## 7. Interview Quick Reference

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
