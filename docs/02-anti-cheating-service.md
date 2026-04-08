# Anti-Cheating Service — Online Exam Platform (IELTS / TOEIC)

## 1. Tổng quan hệ thống

Anti-Cheating Service là service backend (Golang) chịu trách nhiệm **thu thập, buffer, phân tích hành vi thí sinh** trong quá trình làm bài thi online để phát hiện gian lận.

### Bối cảnh dự án

- **Platform**: Luyện thi & thi thử IELTS / TOEIC online
- **Mục tiêu**: Đảm bảo kết quả thi phản ánh đúng năng lực thật của thí sinh
- **Yêu cầu**: Phát hiện gian lận real-time (trong lúc thi) + phân tích hậu kỳ (sau khi thi)

### Kiến trúc tổng thể

```
┌──────────────┐     ┌──────────────────┐     ┌──────────────┐     ┌──────────────────────┐
│  Exam Client  │────▶│  API Gateway     │────▶│  Anti-Cheat   │────▶│  Redis Event Buffer  │
│  (Browser)    │     │  (gRPC/REST)     │     │  Service      │     │  (In-memory queue)   │
│               │     └──────────────────┘     │  (Golang)     │     └──────────┬───────────┘
│  JS SDK gửi   │                              └──────┬────────┘                │
│  exam events  │                                     │                        │ Flush mỗi 60s
└──────────────┘                                      │                        ▼
                                               ┌──────▼────────┐     ┌──────────────────────┐
                                               │  Rule Engine   │◀───│  PostgreSQL +         │
                                               │  (Analyzer)    │    │  TimescaleDB          │
                                               └──────┬────────┘     │  (Persistent Storage) │
                                                      │              └──────────────────────┘
                                                      ▼
                                               ┌──────────────┐
                                               │  Action:      │
                                               │  - Flag exam  │
                                               │  - Warn user  │
                                               │  - Void result│
                                               │  - Alert admin│
                                               └──────────────┘
```

---

## 2. Phân loại Event (6 nhóm)

### 2.1. Navigation Events (Chuyển đổi ngữ cảnh)

Thí sinh rời khỏi hoặc quay lại cửa sổ thi — tín hiệu gian lận phổ biến nhất.

| Event Type | Mô tả | Gian lận tiềm năng |
|---|---|---|
| `tab_switch` | Chuyển sang tab/window khác | Tra cứu đáp án, dùng Google Translate |
| `tab_hidden` | Tab exam bị ẩn (visibilitychange) | Đọc tài liệu ở window khác |
| `focus_lost` | Browser window mất focus | Chuyển sang app khác |
| `focus_gained` | Browser window được focus lại | Quay lại sau khi tra cứu |
| `mouse_leave` | Mouse rời khỏi exam window | Thao tác bên ngoài exam |

### 2.2. Input Events (Nhập liệu & Tương tác)

Tương tác với nội dung bài thi — phát hiện copy-paste, bot, người khác làm thay.

| Event Type | Mô tả | Gian lận tiềm năng |
|---|---|---|
| `copy_attempt` | Thí sinh copy text từ đề thi | Chép đề ra ngoài để nhờ người giải |
| `paste_detected` | Thí sinh paste text vào answer | Dán đáp án từ nguồn bên ngoài |
| `typing_pattern` | Tốc độ và rhythm gõ phím | Copy-paste ngụy trang, hoặc người khác gõ |
| `keyboard_shortcut` | Ctrl+C, Ctrl+V, Alt+Tab, etc. | Shortcut liên quan gian lận |
| `right_click` | Chuột phải trên đề thi | Cố inspect element hoặc copy |

### 2.3. Answer Events (Làm bài)

Nộp câu trả lời — giá trị phân tích cao nhất, cross-reference với Navigation Events.

| Event Type | Mô tả | Gian lận tiềm năng |
|---|---|---|
| `answer_submit` | Nộp câu trả lời | Pattern bất thường (quá nhanh, quá đều) |
| `section_navigate` | Chuyển giữa các section | Skip pattern bất thường |

### 2.4. Environment Events (Môi trường & Thiết bị)

Thay đổi trạng thái kỹ thuật — phát hiện đổi thiết bị, VPN, màn hình phụ.

| Event Type | Mô tả | Gian lận tiềm năng |
|---|---|---|
| `device_change` | Fingerprint thiết bị thay đổi | Chuyển sang device khác giữa chừng |
| `ip_change` | IP thay đổi trong phiên thi | Dùng VPN/proxy |
| `screen_resize` | Thay đổi kích thước màn hình | Thu nhỏ để mở window bên cạnh |
| `multiple_screen` | Phát hiện nhiều màn hình | Xem tài liệu ở màn hình phụ |

### 2.5. Security Events (Bảo mật)

Hành vi kỹ thuật nhằm bypass hệ thống — severity cao nhất, gần 0 false positive.

| Event Type | Mô tả | Gian lận tiềm năng |
|---|---|---|
| `devtools_open` | DevTools được mở | Inspect element, xem API response chứa đáp án |
| `automation_signal` | Phát hiện WebDriver/headless | Bot tự động làm bài |

### 2.6. Lifecycle Events (Vòng đời phiên thi)

Bắt đầu và kết thúc phiên thi — capture baseline và trigger batch analysis.

| Event Type | Mô tả | Gian lận tiềm năng |
|---|---|---|
| `exam_start` | Bắt đầu bài thi | Bắt đầu từ device/IP bất thường |
| `exam_end` | Kết thúc bài thi | Kết thúc bất thường nhanh |

---

## 3. Chi tiết Event Data Structure

### Base Event (Golang struct)

```go
package model

import "time"

type BaseEvent struct {
    ID        string    `json:"id" db:"id"`
    UserID    string    `json:"user_id" db:"user_id"`
    ExamID    string    `json:"exam_id" db:"exam_id"`         // ID bài thi
    SessionID string    `json:"session_id" db:"session_id"`   // Phiên thi
    EventType string    `json:"event_type" db:"event_type"`   // "navigation" | "input" | "answer" | "environment" | "security" | "lifecycle"
    Action    string    `json:"action" db:"action"`           // "tab_switch", "paste_detected", etc.
    Timestamp time.Time `json:"timestamp" db:"timestamp"`
    Metadata  JSONB     `json:"metadata" db:"metadata"`
}

type BehaviorEvent struct {
    BaseEvent
    QuestionID  string `json:"question_id" db:"question_id"`   // Đang ở câu hỏi nào
    SectionType string `json:"section_type" db:"section_type"` // "listening", "reading", "writing", "speaking"
}

type SessionEvent struct {
    BaseEvent
    IP                string `json:"ip" db:"ip"`
    UserAgent         string `json:"user_agent" db:"user_agent"`
    DeviceFingerprint string `json:"device_fingerprint" db:"device_fingerprint"`
    ScreenResolution  string `json:"screen_resolution" db:"screen_resolution"`
}
```

### Ví dụ Event JSON

**Tab Switch (thí sinh chuyển tab):**
```json
{
  "id": "01912f4c-...",
  "user_id": "user_1234",
  "exam_id": "ielts_mock_0056",
  "session_id": "sess_abc",
  "event_type": "navigation",
  "action": "tab_switch",
  "timestamp": "2026-04-06T14:32:01Z",
  "question_id": "reading_q12",
  "section_type": "reading",
  "metadata": {
    "hidden_duration_ms": 8500,
    "switch_count_in_session": 3,
    "previous_action": "answer_submit"
  }
}
```

**Paste Detected (dán text vào writing):**
```json
{
  "id": "01912f4d-...",
  "user_id": "user_1234",
  "exam_id": "ielts_mock_0056",
  "session_id": "sess_abc",
  "event_type": "input",
  "action": "paste_detected",
  "timestamp": "2026-04-06T14:35:22Z",
  "question_id": "writing_task2",
  "section_type": "writing",
  "metadata": {
    "pasted_text_length": 320,
    "pasted_text_hash": "sha256:abc123...",
    "total_typed_chars": 45,
    "paste_ratio": 0.88
  }
}
```

---

## 4. Flow chi tiết: Từ Event đến Detection

```
  Thí sinh làm bài thi
         │
         ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  BƯỚC 1: JS SDK thu thập event                                  │
  │  - visibilitychange listener → tab_switch, tab_hidden           │
  │  - paste event listener → paste_detected                        │
  │  - keydown listener → keyboard_shortcut, typing_pattern         │
  │  - focus/blur listener → focus_lost, focus_gained               │
  │  - Gửi batch mỗi 5 giây hoặc khi có event critical             │
  └──────────────────────┬──────────────────────────────────────────┘
                         │
                         ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  BƯỚC 2: Real-time Check (Lớp 1 — Redis)                        │
  │                                                                  │
  │  Critical events xử lý ngay:                                     │
  │  - tab_switch count > 5 trong 10 phút → warning cho thí sinh    │
  │  - paste > 200 chars trong writing → flag ngay                   │
  │  - devtools_open → flag ngay + cảnh báo                          │
  │  - automation_signal → void exam ngay                            │
  └──────────────────────┬──────────────────────────────────────────┘
                         │
                         ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  BƯỚC 3: Push vào Redis Event Buffer (LPUSH, ~0.1ms)             │
  └──────────────────────┬──────────────────────────────────────────┘
                         │
                         │  Flush mỗi 60 giây
                         ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  BƯỚC 4: Batch INSERT vào TimescaleDB (COPY protocol)           │
  └──────────────────────┬──────────────────────────────────────────┘
                         │
                         ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  BƯỚC 5: Rule Engine phân tích (Lớp 2 — Batch)                  │
  │                                                                  │
  │  Phân tích sau mỗi flush + phân tích cuối bài thi:              │
  │  - Answer speed anomaly (trả lời nhanh bất thường)              │
  │  - Tab switch pattern (chuyển tab rồi trả lời đúng)             │
  │  - Paste ratio (bao nhiêu % bài writing là paste?)              │
  │  - Typing rhythm anomaly (gõ đều bất thường = bot)              │
  │  - Cross-exam similarity (2 thí sinh cùng đáp án pattern)       │
  └──────────────────────┬──────────────────────────────────────────┘
                         │
                         ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  BƯỚC 6: Action                                                  │
  │                                                                  │
  │  severity=low      → Flag cho admin review sau                   │
  │  severity=medium   → Cảnh báo thí sinh (hiện warning trên UI)   │
  │  severity=high     → Flag exam result + notify admin             │
  │  severity=critical → Void exam result + lock account             │
  └─────────────────────────────────────────────────────────────────┘
```

---

## 5. Tại sao Redis Buffer + PostgreSQL TimescaleDB?

### Tại sao cần Redis Buffer?

```
          Không có buffer                         Có Redis buffer
    ┌─────────────────────┐                ┌─────────────────────┐
    │  50 thí sinh thi     │                │  50 thí sinh thi     │
    │  cùng lúc            │                │  cùng lúc            │
    │  ~5,000 events/phút  │                │  ~5,000 events/phút  │
    │         │            │                │         │            │
    │         ▼            │                │         ▼            │
    │  5,000 INSERT/phút  │                │  5,000 LPUSH/phút   │
    │  vào PostgreSQL      │                │  vào Redis (0.1ms)  │
    │         │            │                │         │            │
    │         ▼            │                │         ▼            │
    │  ❌ DB load cao      │                │  1 BATCH INSERT/phút│
    │  ❌ Latency cho      │                │  vào PostgreSQL     │
    │     exam client      │                │         │            │
    │  ❌ Peak exam time   │                │         ▼            │
    │     (TOEIC monthly)  │                │  ✅ DB khỏe          │
    └─────────────────────┘                │  ✅ Client nhanh     │
                                           └─────────────────────┘
```

**Lý do cốt lõi:**
1. **Client không bị block** — exam UI phải smooth, LPUSH ~0.1ms vs INSERT 5-50ms
2. **Giảm I/O cho DB** — 5,000 events riêng lẻ → 1 COPY batch
3. **Chịu peak** — TOEIC monthly exam, hàng trăm thí sinh thi cùng lúc
4. **Decouple** — DB down tạm thời, events vẫn an toàn trong Redis

### Tại sao PostgreSQL + TimescaleDB?

1. **Time-series native** — Exam events là time-series. Query "đếm tab_switch trong 10 phút qua" cực nhanh
2. **Continuous aggregates** — Tự tính sẵn: số event/thí sinh/phút, rule engine query nhanh
3. **Compression** — Data cũ compress 90%+, tiết kiệm storage
4. **Retention** — Giữ data 1 năm (audit), tự xóa sau đó
5. **Window functions** — Phân tích pattern phức tạp (answer timing, similarity)
6. **Dùng chung PostgreSQL** — Team quen, không cần cluster riêng

---

## 6. Database Schema

```sql
CREATE EXTENSION IF NOT EXISTS timescaledb;

CREATE TABLE exam_events (
    id              UUID        NOT NULL DEFAULT gen_random_uuid(),
    user_id         TEXT        NOT NULL,
    exam_id         TEXT        NOT NULL,
    session_id      TEXT        NOT NULL,
    event_type      TEXT        NOT NULL,   -- 'navigation' | 'input' | 'answer' | 'environment' | 'security' | 'lifecycle'
    action          TEXT        NOT NULL,   -- 'tab_switch', 'paste_detected', etc.
    section_type    TEXT,                   -- 'listening', 'reading', 'writing', 'speaking'
    question_id     TEXT,
    ip              INET,
    device_fingerprint TEXT,
    metadata        JSONB       DEFAULT '{}',
    risk_score      SMALLINT    DEFAULT 0,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (id, created_at)
);

SELECT create_hypertable('exam_events', 'created_at',
    chunk_time_interval => INTERVAL '1 day'
);

CREATE INDEX idx_events_exam     ON exam_events (exam_id, user_id, created_at DESC);
CREATE INDEX idx_events_session  ON exam_events (session_id, created_at DESC);
CREATE INDEX idx_events_action   ON exam_events (action, created_at DESC);

-- Continuous Aggregate: event count per user per exam per minute
CREATE MATERIALIZED VIEW exam_event_counts_per_minute
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 minute', created_at) AS bucket,
    user_id,
    exam_id,
    action,
    COUNT(*)               AS event_count,
    section_type
FROM exam_events
GROUP BY bucket, user_id, exam_id, action, section_type;

SELECT add_continuous_aggregate_policy('exam_event_counts_per_minute',
    start_offset    => INTERVAL '10 minutes',
    end_offset      => INTERVAL '1 minute',
    schedule_interval => INTERVAL '1 minute'
);

-- Compression & Retention
ALTER TABLE exam_events SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'exam_id, user_id',
    timescaledb.compress_orderby = 'created_at DESC'
);
SELECT add_compression_policy('exam_events', INTERVAL '7 days');
SELECT add_retention_policy('exam_events', INTERVAL '365 days');

-- Cheat detection results
CREATE TABLE cheat_detections (
    id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         TEXT        NOT NULL,
    exam_id         TEXT        NOT NULL,
    rule_id         TEXT        NOT NULL,
    severity        TEXT        NOT NULL,     -- 'low', 'medium', 'high', 'critical'
    action_taken    TEXT        NOT NULL,     -- 'flag', 'warn', 'void_result', 'lock_account'
    evidence        JSONB       NOT NULL,
    reviewed        BOOLEAN     DEFAULT FALSE,
    reviewed_by     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## 7. Detection Rules cho Exam

### 7.1. Real-time Rules (Lớp 1 — Redis, trong lúc thi)

| Rule | Trigger | Action |
|---|---|---|
| Tab switch > 5 lần / 10 phút | `tab_switch` count | Warning hiện trên UI |
| Tab hidden > 30 giây liên tục | `tab_hidden` duration | Warning + log |
| Paste > 100 chars trong Writing | `paste_detected` length | Flag ngay |
| DevTools mở | `devtools_open` | Flag + cảnh báo nghiêm trọng |
| Automation detected | `automation_signal` | Void exam ngay |
| Focus lost > 10 lần / section | `focus_lost` count | Warning |

### 7.2. Batch Analysis Rules (Lớp 2 — TimescaleDB, sau mỗi flush + cuối bài)

| Rule | Phân tích | Phát hiện |
|---|---|---|
| **Answer speed anomaly** | Thời gian trả lời mỗi câu vs trung bình section | Reading câu 500 words trả lời trong 5 giây = bất thường |
| **Tab-then-correct pattern** | Tab switch → quay lại → trả lời đúng | Tra cứu đáp án bên ngoài |
| **Paste ratio** | % bài Writing là paste vs typed | paste_ratio > 50% → gian lận |
| **Typing rhythm** | Variance của keystroke interval | Quá đều (< 10ms stddev) = bot. Burst typing = paste ngụy trang |
| **Cross-exam similarity** | So sánh answer pattern giữa 2 thí sinh | Cùng phòng thi, answer sequence giống > 90% |
| **Section timing anomaly** | Tổng thời gian per section vs bình thường | Listening section xong trong 5 phút (bình thường 30 phút) |
| **Device/IP anomaly** | IP hoặc device thay đổi giữa bài thi | Chuyển device = người khác làm thay |

### 7.3. Ví dụ SQL — Tab-then-correct pattern

```sql
WITH tab_switches AS (
    SELECT user_id, exam_id, question_id, created_at as switch_time
    FROM exam_events
    WHERE action = 'tab_switch'
      AND exam_id = :examId
),
answers AS (
    SELECT user_id, exam_id, question_id, created_at as answer_time,
           metadata->>'is_correct' as is_correct
    FROM exam_events
    WHERE action = 'answer_submit'
      AND exam_id = :examId
)
SELECT
    a.user_id,
    COUNT(*) as tab_then_correct_count,
    COUNT(*) FILTER (WHERE a.is_correct = 'true') as correct_after_tab
FROM answers a
JOIN tab_switches t
    ON a.user_id = t.user_id
    AND a.question_id = t.question_id
    AND a.answer_time BETWEEN t.switch_time AND t.switch_time + INTERVAL '60 seconds'
GROUP BY a.user_id
HAVING COUNT(*) FILTER (WHERE a.is_correct = 'true') > 5;
-- Thí sinh chuyển tab rồi trả lời đúng > 5 lần = rất đáng ngờ
```

---

## 8. Ví dụ kịch bản thực tế

### Kịch bản 1: Thí sinh tra Google Translate trong Reading

```
Timeline:
  14:00:00 - Exam start (IELTS Reading Mock Test)
  14:12:30 - Reading passage 2, question 12
  14:12:31 - tab_switch (hidden_duration: 12s)         ← chuyển tab
  14:12:43 - focus_gained                               ← quay lại
  14:12:45 - answer_submit (question 12, correct: true) ← trả lời đúng

  Pattern lặp lại 8 lần trong 20 phút

Lớp 1 (Real-time):
  tab_switch count = 3 → OK
  tab_switch count = 6 → WARNING hiện trên UI:
    "Bạn đã chuyển tab nhiều lần. Hành vi này được ghi nhận."

Lớp 2 (Batch — cuối bài thi):
  Rule: tab-then-correct pattern
  → 8/8 lần chuyển tab đều trả lời đúng ngay sau đó
  → Tỷ lệ đúng sau tab: 100% (bình thường ~40%)
  → Detection: severity=high, action=flag_result
  → Admin review: xác nhận gian lận → void result
```

### Kịch bản 2: Paste bài Writing từ ChatGPT

```
Timeline:
  15:00:00 - Writing Task 2 bắt đầu
  15:00:00 đến 15:02:00 - typing_pattern: 0 keystrokes
  15:02:01 - paste_detected (length: 287 chars)          ← paste đoạn 1
  15:02:05 - typing_pattern: 12 keystrokes (sửa lỗi nhỏ)
  15:03:00 - paste_detected (length: 195 chars)          ← paste đoạn 2
  15:03:02 - answer_submit

Lớp 1 (Real-time):
  paste > 100 chars → FLAG ngay
  Cảnh báo: "Hành vi paste đã được ghi nhận."

Lớp 2 (Batch):
  paste_ratio = 482 / (482 + 12) = 97.6% → gian lận rõ ràng
  typing_pattern entropy rất thấp (chỉ sửa, không gõ)
  Detection: severity=critical, action=void_result
```

### Kịch bản 3: Hai thí sinh chia sẻ đáp án (cùng phòng)

```
Thí sinh A: answers = [B, A, C, D, B, A, C, C, D, A, ...]
Thí sinh B: answers = [B, A, C, D, B, A, C, C, D, A, ...]
Similarity: 95% (50 câu giống nhau trên 52 câu)

Lớp 2 (Batch — sau bài thi):
  Cross-exam similarity check
  → IP khác nhau nhưng cùng exam_id và time window
  → Answer sequence similarity > 90%
  → Detection: severity=high cho cả 2 thí sinh
  → Admin review
```

---

## 9. Project Structure (Golang)

```
anti-cheat-service/
├── cmd/
│   └── server/
│       └── main.go
├── internal/
│   ├── handler/
│   │   └── event_handler.go         # HTTP handler nhận events từ JS SDK
│   ├── buffer/
│   │   └── redis_buffer.go          # Redis event buffer + flusher
│   ├── store/
│   │   └── postgres_store.go        # TimescaleDB batch insert
│   ├── realtime/
│   │   └── checker.go               # Lớp 1: real-time checks (Redis counters)
│   ├── rules/
│   │   ├── engine.go                # Rule engine orchestrator
│   │   ├── tab_switch_pattern.go    # Rule: tab-then-correct
│   │   ├── paste_ratio.go           # Rule: paste ratio trong writing
│   │   ├── answer_speed.go          # Rule: answer speed anomaly
│   │   ├── typing_rhythm.go         # Rule: typing pattern
│   │   └── cross_exam_similarity.go # Rule: similarity giữa 2 thí sinh
│   ├── model/
│   │   ├── event.go
│   │   └── detection.go
│   └── action/
│       ├── executor.go              # Thực thi: flag, warn, void
│       └── notifier.go              # Notify admin, notify thí sinh
├── migrations/
│   └── 001_create_tables.sql
├── config/
│   └── config.go
├── go.mod
└── go.sum
```

---

## 10. Tóm tắt kiến trúc 2 lớp

```
┌──────────────────────────────────────────────────────────────────┐
│                    LỚP 1: REAL-TIME (Redis)                       │
│                                                                   │
│  Mục đích: Cảnh báo/chặn ngay hành vi rõ ràng                   │
│  Latency:  < 1ms                                                  │
│  Ví dụ:    tab_switch > 5 lần → warning                          │
│            paste > 100 chars → flag                               │
│            devtools_open → flag                                    │
│            automation → void                                       │
│                                                                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│                    LỚP 2: BATCH ANALYSIS (TimescaleDB)            │
│                                                                   │
│  Mục đích: Phát hiện pattern phức tạp cần cross-reference         │
│  Latency:  60 giây (flush interval) + cuối bài thi               │
│  Ví dụ:    Tab-then-correct pattern                               │
│            Answer speed anomaly                                    │
│            Cross-exam similarity                                   │
│            Paste ratio + typing rhythm                             │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘

Redis = Tốc độ + Buffer (real-time warning trong lúc thi)
TimescaleDB = Phân tích sâu + Lưu trữ (pattern analysis + audit trail)
```
