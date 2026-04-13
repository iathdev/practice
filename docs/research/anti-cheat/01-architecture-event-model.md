2# Anti-Cheat Service — Kiến trúc, Event Model & Data Flow

> Thiết kế hệ thống anti-cheating cho nền tảng thi tiếng Anh online (IELTS/TOEIC), phục vụ 5,000–10,000 thí sinh thi đồng thời. Service backend viết bằng Golang, sử dụng Redis buffer + PostgreSQL TimescaleDB.

---

## Mục lục

1. [Bối cảnh & Yêu cầu](#1-bối-cảnh--yêu-cầu)
2. [Kiến trúc tổng thể — 2 Lớp phát hiện](#2-kiến-trúc-tổng-thể--2-lớp-phát-hiện)
3. [Phân loại Event — Behavior vs Session](#3-phân-loại-event--behavior-vs-session)
4. [Event Data Structure](#4-event-data-structure)
5. [Flow chi tiết: Từ Browser đến Detection](#5-flow-chi-tiết-từ-browser-đến-detection)
6. [Tại sao Redis Buffer + PostgreSQL TimescaleDB?](#6-tại-sao-redis-buffer--postgresql-timescaledb)
7. [Database Schema](#7-database-schema)
8. [Project Structure (Golang)](#8-project-structure-golang)
9. [Interview Quick Reference](#9-interview-quick-reference)

---

## Tech Stack & Lý do lựa chọn

### Tại sao Golang?

Golang phù hợp với bài toán này vì **concurrency model của Go khớp tự nhiên với pipeline architecture**.

> **Pipeline architecture** = chuỗi các bước xử lý nối tiếp, mỗi bước làm 1 việc, chạy độc lập song song — giống dây chuyền sản xuất:
> ```
> Event → [Receive] → [Buffer] → [Flush] → [Detect] → [Alert]
> ```
> Ingestion goroutine không chờ flush xong. Flush goroutine không chờ rule engine xong. Mỗi bước chạy theo tốc độ của chính nó.

**Goroutine** — ingestion và flush chạy song song, không block nhau:
```
Ingestion goroutine:  nhận events → LPUSH Redis  (chạy liên tục)
Flush goroutine:      mỗi 60s → RPOPLPUSH → INSERT DB  (chạy song song)
```
Java cần thread pool + executor phức tạp hơn. Go chỉ cần `go func()`.

**`context`** — graceful shutdown tự nhiên:
```go
ctx, cancel := signal.NotifyContext(ctx, syscall.SIGTERM)
// flush goroutine lắng nghe ctx.Done() → final flush trước khi thoát
```
Cancel 1 chỗ → toàn bộ pipeline dừng đúng thứ tự → không mất event khi deploy.

**`select` + `ticker`** — adaptive flush interval gọn:
```go
select {
case <-time.After(interval):  // flush định kỳ
case <-bufferFullSignal:       // flush ngay khi buffer đầy
case <-ctx.Done():             // graceful shutdown
}
```

| | Go | Java/Spring |
|---|---|---|
| Concurrency | Goroutine nhẹ (~4KB) | Thread nặng (~1MB) |
| GC pause | Nhỏ, < 1ms | Stop-the-world có thể > 10ms |
| Graceful shutdown | Built-in context + signal | Cần ShutdownHook + phức tạp hơn |
| Memory footprint | Thấp, binary nhỏ | JVM overhead |
| Real-time check < 1ms | Đáp ứng tốt | Khó đảm bảo vì GC |

---

### Tại sao dùng Event Buffer (Redis)?

Bài toán: 10,000 thí sinh × ~2 events/giây = **20,000 events/giây**. Không thể ghi thẳng vào DB.

```
Không có buffer:
  Event → DB INSERT trực tiếp
  → 20K concurrent writes → DB quá tải → latency tăng → có thể down
  → DB down = MẤT EVENT

Có Redis buffer:
  Event → Redis (100K+ ops/s) → flush batch mỗi 60s → DB
  → Ingestion và DB write hoàn toàn độc lập (decouple)
  → DB down → events vẫn nằm trong Redis, không mất
```

**3 lợi ích cốt lõi:**

| Lợi ích | Giải thích |
|---|---|
| Decouple | Ingestion tốc độ Redis, DB write tốc độ DB — không phụ thuộc nhau |
| Absorb burst | Peak traffic đổ vào Redis trước, DB xử lý theo pace của mình |
| Crash safety | DB down → events giữ trong Redis → recover xong flush tiếp |

---

### Cơ chế cache: Write-back (Write-behind)

Redis buffer trong hệ thống này hoạt động theo pattern **write-back**:

```
Write-through:                      Write-back (anti-cheat):
Event → Redis + DB đồng thời        Event → Redis trước (ack ngay)
        (sync, chờ cả 2)                    ↓ async, 60s sau
                                            DB (batch flush)
```

**Tại sao write-back chứ không phải write-through?**
- Write-through: client phải chờ DB write → latency cao → exam bị lag
- Write-back: client chỉ chờ Redis (~0.1ms) → ack ngay → exam mượt

**Risk của write-back:** Redis crash trước khi flush → mất data.
**Giảm thiểu bằng:** `events:processing` pattern + idempotent insert (UUID v7 + ON CONFLICT DO NOTHING).

---

## 1. Bối cảnh & Yêu cầu

### Dự án

- **Platform**: Luyện thi & thi thử IELTS / TOEIC online (tham khảo: Công ty Cổ phần Công nghệ Prep)
- **Mô hình kinh doanh**: B2B — bán giải pháp thi cho trường học, trung tâm ngoại ngữ, tổ chức thi
- **Mục tiêu**: Đảm bảo kết quả thi phản ánh đúng năng lực thật của thí sinh

### Yêu cầu kỹ thuật 

| Tiêu chí | Yêu cầu |
|----------|---------|
| Concurrent users | 5,000–10,000 thí sinh thi đồng thời |
| Event throughput | ~20,000 events/giây (peak) |
| Real-time latency | Cảnh báo gian lận trong < 1 giây |
| Batch analysis | Phân tích pattern phức tạp sau mỗi 60 giây + cuối bài thi |
| Data retention | Giữ event data 1 năm (audit trail cho đối tác B2B) |
| Availability | Service không được làm ảnh hưởng trải nghiệm thi |

### Tại sao cần Anti-Cheat?

```
Không có Anti-Cheat:
  Thí sinh gian lận → điểm cao ảo → đối tác mất niềm tin
  → hủy hợp đồng B2B → mất doanh thu

Có Anti-Cheat:
  Phát hiện gian lận real-time + hậu kỳ
  → kết quả đáng tin cậy → đối tác tin tưởng
  → gia hạn hợp đồng + giới thiệu thêm khách hàng
```

**Trong mô hình B2B, niềm tin của đối tác là tài sản quan trọng nhất.** Nếu đối tác (trường học, trung tâm) phát hiện hệ thống dễ gian lận → họ không dùng nữa → mất cả batch khách hàng chứ không phải 1 user.

---

## 2. Kiến trúc tổng thể — 2 Lớp phát hiện

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

### Nguyên tắc thiết kế

| Nguyên tắc | Giải thích |
|-----------|-----------|
| **Non-blocking** | Anti-cheat KHÔNG được block exam UI. Event gửi async, response ngay |
| **2-layer detection** | Lớp 1 (Redis) bắt hành vi rõ ràng < 1ms. Lớp 2 (Rule Engine query TimescaleDB) phân tích pattern phức tạp |
| **Buffer decouple** | Redis buffer tách event ingestion khỏi DB write. DB down ≠ mất event |
| **Conservative** | Thà cảnh báo thừa còn hơn bỏ sót. Nhưng không quá aggressive → UX tệ |

### Tóm tắt kiến trúc 2 lớp

```
┌──────────────────────────────────────────────────────────────────┐
│                    LỚP 1: REAL-TIME (Redis)                       │
│                                                                   │
│  Mục đích: Cảnh báo/chặn ngay hành vi rõ ràng                   │
│  Latency:  < 1ms                                                  │
│  Cơ chế:   Redis INCR counters + threshold check (Lua script)    │
│  Ví dụ:    tab_switch > 5 lần → warning                          │
│            paste > 100 chars → flag                               │
│            devtools_open → flag                                    │
│            automation → void                                       │
│                                                                   │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│                    LỚP 2: BATCH ANALYSIS (Rule Engine + TimescaleDB) │
│                                                                   │
│  Mục đích: Phát hiện pattern phức tạp cần cross-reference         │
│  Latency:  60 giây (flush interval) + cuối bài thi               │
│  Cơ chế:   Rule Engine (Go code) query TimescaleDB                │
│            TimescaleDB là storage layer, hỗ trợ query nhanh       │
│            nhờ chunk pruning + continuous aggregates               │
│  Ví dụ:    Tab-then-correct pattern                               │
│            Answer speed anomaly                                    │
│            Cross-exam similarity                                   │
│            Paste ratio + typing rhythm                             │
│                                                                   │
│  Flow:  Flush xong → Rule Engine gửi SQL query                    │
│         → TimescaleDB trả data                                    │
│         → Rule Engine đánh giá → tạo detection                    │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘

Redis = Tốc độ + Buffer (real-time counter + event buffer)
TimescaleDB = Lưu trữ (storage, query nhanh nhờ time-series optimization)
Rule Engine = Phân tích (logic detection nằm ở đây, KHÔNG phải ở DB)
```

---

## 3. Phân loại Event

Events được chia thành **6 nhóm** theo bản chất hành vi, không chỉ 2 loại đơn giản. Mỗi nhóm có đặc điểm riêng về volume, cách thu thập, và logic detection khác nhau.

### Tổng quan 6 nhóm

```
┌────────────────────────────────────────────────────────────────────┐
│                        EXAM EVENTS                                  │
│                                                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │
│  │ 1. Navigation    │  │ 2. Input        │  │ 3. Answer       │    │
│  │    Events        │  │    Events       │  │    Events       │    │
│  │                  │  │                 │  │                 │    │
│  │ tab_switch       │  │ copy_attempt    │  │ answer_submit   │    │
│  │ tab_hidden       │  │ paste_detected  │  │ section_navigate│    │
│  │ focus_lost       │  │ typing_pattern  │  │                 │    │
│  │ focus_gained     │  │ keyboard_shortcut│ │                 │    │
│  │ mouse_leave      │  │ right_click     │  │                 │    │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘    │
│                                                                     │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐    │
│  │ 4. Environment   │  │ 5. Security     │  │ 6. Lifecycle    │    │
│  │    Events        │  │    Events       │  │    Events       │    │
│  │                  │  │                 │  │                 │    │
│  │ device_change    │  │ devtools_open   │  │ exam_start      │    │
│  │ ip_change        │  │ automation_signal│ │ exam_end        │    │
│  │ screen_resize    │  │                 │  │                 │    │
│  │ multiple_screen  │  │                 │  │                 │    │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘    │
└────────────────────────────────────────────────────────────────────┘
```

### 3.1. Navigation Events (Chuyển đổi ngữ cảnh)

Thí sinh **rời khỏi** hoặc **quay lại** cửa sổ thi. Nhóm này có volume cao nhất và là tín hiệu gian lận phổ biến nhất (tra cứu đáp án bên ngoài).

| Event Type | Mô tả | Gian lận tiềm năng | Detection |
|---|---|---|---|
| `tab_switch` | Chuyển sang tab/window khác | Tra cứu đáp án, Google Translate | Real-time: count > 5/10min |
| `tab_hidden` | Tab exam bị ẩn (visibilitychange) | Đọc tài liệu ở window khác | Real-time: duration > 30s |
| `focus_lost` | Browser window mất focus | Chuyển sang app khác (dict, translator) | Real-time: count > 10/section |
| `focus_gained` | Browser window được focus lại | Quay lại sau khi tra cứu | Batch: tương quan với answer_submit |
| `mouse_leave` | Mouse rời khỏi exam window | Thao tác bên ngoài exam | Batch: frequency pattern |

**Đặc điểm:**
- Volume cao (thí sinh chuyển tab liên tục nếu gian lận)
- Tương quan mạnh với Answer Events (chuyển tab → trả lời đúng = pattern rõ ràng)
- Cần cả real-time (cảnh báo sớm) + batch (xác nhận pattern)

### 3.2. Input Events (Nhập liệu & Tương tác)

Thí sinh **tương tác** với nội dung bài thi bằng keyboard/mouse. Phát hiện copy-paste, bot, hoặc người khác làm thay.

| Event Type | Mô tả | Gian lận tiềm năng | Detection |
|---|---|---|---|
| `copy_attempt` | Copy text từ đề thi | Chép đề ra ngoài nhờ người giải | Real-time: log + flag |
| `paste_detected` | Paste text vào answer | Dán đáp án từ ChatGPT / nguồn ngoài | Real-time: length > 100 chars |
| `typing_pattern` | Tốc độ và rhythm gõ phím | Bot (stddev thấp), người khác gõ (style khác) | Batch: stddev < 10ms |
| `keyboard_shortcut` | Ctrl+C, Ctrl+V, Alt+Tab, etc. | Shortcut liên quan gian lận | Real-time: log specific combos |
| `right_click` | Chuột phải trên đề thi | Inspect element hoặc copy | Real-time: log + warn |

**Đặc điểm:**
- Metadata phong phú: `pasted_text_length`, `keystroke_interval_ms`, `shortcut_key`
- Paste detection kết hợp với typing_pattern → `paste_ratio` (batch rule mạnh)
- Typing rhythm cần tích lũy đủ sample (>= 20 events) mới phân tích được

### 3.3. Answer Events (Làm bài)

Thí sinh **nộp câu trả lời** hoặc **di chuyển giữa các phần thi**. Đây là dữ liệu lõi để phân tích pattern gian lận ở Lớp 2.

| Event Type | Mô tả | Gian lận tiềm năng | Detection |
|---|---|---|---|
| `answer_submit` | Nộp câu trả lời | Trả lời quá nhanh, quá đều, hoặc đúng sau tab switch | Batch: speed anomaly, tab-then-correct |
| `section_navigate` | Chuyển giữa các section (L/R/W/S) | Skip pattern bất thường, xong section nhanh bất thường | Batch: section timing anomaly |

**Đặc điểm:**
- Volume thấp hơn (mỗi câu 1 event) nhưng **giá trị phân tích cao nhất**
- Metadata chứa: `question_id`, `selected_answer`, `is_correct`, `time_spent_ms`
- Cross-reference với Navigation Events → pattern mạnh nhất (tab-then-correct)
- Cross-reference giữa 2 thí sinh → cross-exam similarity

### 3.4. Environment Events (Môi trường & Thiết bị)

**Trạng thái kỹ thuật** của thiết bị và mạng thay đổi. Phát hiện thí sinh đổi thiết bị, dùng VPN, hoặc mở màn hình phụ.

| Event Type | Mô tả | Gian lận tiềm năng | Detection |
|---|---|---|---|
| `device_change` | Fingerprint thiết bị thay đổi | Chuyển sang device khác = người khác làm thay | Real-time: flag ngay |
| `ip_change` | IP thay đổi trong phiên thi | Dùng VPN/proxy để tránh tracking | Batch: pattern analysis |
| `screen_resize` | Thay đổi kích thước màn hình | Thu nhỏ để mở window bên cạnh | Batch: frequency + tương quan navigation |
| `multiple_screen` | Phát hiện nhiều màn hình | Xem tài liệu ở màn hình phụ | Real-time: flag + warn |

**Đặc điểm:**
- Volume thấp (chỉ khi trạng thái thay đổi)
- Mỗi event mang nhiều context: `ip`, `user_agent`, `device_fingerprint`, `screen_resolution`
- `device_change` giữa bài thi gần như chắc chắn đáng ngờ
- `ip_change` cần phân biệt: VPN ≠ chuyển từ wifi sang 4G (false positive)

### 3.5. Security Events (Bảo mật)

Phát hiện **hành vi kỹ thuật** nhằm bypass hệ thống thi hoặc trích xuất đáp án. Severity cao nhất — thường auto-void.

| Event Type | Mô tả | Gian lận tiềm năng | Detection |
|---|---|---|---|
| `devtools_open` | DevTools (F12) được mở | Inspect element, xem API response chứa đáp án | Real-time: flag ngay (severity=high) |
| `automation_signal` | Phát hiện WebDriver/headless browser | Bot tự động làm bài | Real-time: void ngay (severity=critical) |

**Đặc điểm:**
- Volume rất thấp (0 hoặc 1 event per session)
- **Gần như không có false positive** → auto-action ngay, không cần batch confirm
- `automation_signal`: detect qua `navigator.webdriver`, `window.chrome` absence, headless signals
- `devtools_open`: detect qua debugger timing, window size discrepancy, `toString` override

### 3.6. Lifecycle Events (Vòng đời phiên thi)

**Bắt đầu và kết thúc** phiên thi. Dùng để xác định time window cho analysis và phát hiện anomaly về timing.

| Event Type | Mô tả | Gian lận tiềm năng | Detection |
|---|---|---|---|
| `exam_start` | Bắt đầu bài thi | Bắt đầu từ device/IP bất thường | Batch: so sánh với baseline |
| `exam_end` | Kết thúc bài thi | Kết thúc bất thường nhanh (bot xong nhanh) | Batch: total exam duration anomaly |

**Đặc điểm:**
- Mỗi session chỉ có đúng 1 `exam_start` và 1 `exam_end`
- `exam_start` capture baseline: IP, device, screen → so sánh với Environment Events sau đó
- `exam_end` trigger batch analysis toàn diện (bao gồm cross-exam similarity)
- Nếu `exam_end` không đến sau timeout → session anomaly

### Tổng hợp: So sánh 6 nhóm

| Nhóm | Volume | Severity phổ biến | Detection layer | Metadata chính |
|------|--------|-------------------|----------------|----------------|
| Navigation | Cao | Medium | Real-time + Batch | `hidden_duration_ms`, `switch_count` |
| Input | Trung bình | High | Real-time + Batch | `pasted_text_length`, `keystroke_interval_ms` |
| Answer | Thấp | High (batch) | Chủ yếu Batch | `question_id`, `is_correct`, `time_spent_ms` |
| Environment | Rất thấp | High | Real-time + Batch | `ip`, `device_fingerprint`, `screen_resolution` |
| Security | Rất thấp | Critical | Real-time | `detection_method`, `browser` |
| Lifecycle | 2 events/session | — | Trigger batch | `ip`, `device_fingerprint` (baseline) |

### Tại sao chia 6 nhóm chứ không gom 2 loại?

```
Nếu chỉ chia "behavior" / "session":
  ✗ tab_switch và answer_submit cùng là "behavior" nhưng logic detection hoàn toàn khác
  ✗ devtools_open và device_change cùng là "session" nhưng severity khác xa
  ✗ Rule engine phải if-else theo action → không scalable
  ✗ Dashboard admin lẫn lộn, khó monitor

Chia 6 nhóm:
  ✓ Mỗi nhóm có đặc điểm riêng: volume, severity, detection layer
  ✓ Rule engine áp dụng strategy pattern theo nhóm
  ✓ Dashboard admin: tab riêng cho Navigation, Input, Security...
  ✓ Thêm event mới → xác định thuộc nhóm nào → biết ngay detection logic
  ✓ Storage query: mỗi nhóm có metadata khác nhau → optimize index theo nhóm
```

---

## 4. Event Data Structure

### Base Event

```go
// BaseEvent — mọi event đều có các field này
type BaseEvent struct {
    ID        string    `json:"id"`
    UserID    string    `json:"user_id"`
    ExamID    string    `json:"exam_id"`
    SessionID string    `json:"session_id"`
    EventType string    `json:"event_type"`   // "navigation" | "input" | "answer" | "environment" | "security" | "lifecycle"
    Action    string    `json:"action"`       // "tab_switch", "paste_detected", "answer_submit", etc.
    Timestamp time.Time `json:"timestamp"`
    Metadata  JSONB     `json:"metadata"`     // Flexible data per action type
}

// NavigationEvent — chuyển đổi ngữ cảnh (tab, focus, mouse)
type NavigationEvent struct {
    BaseEvent
    QuestionID  string `json:"question_id,omitempty"`   // câu hỏi đang làm khi chuyển
    SectionType string `json:"section_type,omitempty"`  // section đang làm
}

// InputEvent — nhập liệu & tương tác (copy, paste, typing, keyboard)
type InputEvent struct {
    BaseEvent
    QuestionID  string `json:"question_id,omitempty"`
    SectionType string `json:"section_type,omitempty"`
}

// AnswerEvent — nộp câu trả lời, chuyển section
type AnswerEvent struct {
    BaseEvent
    QuestionID  string `json:"question_id"`
    SectionType string `json:"section_type"`
}

// EnvironmentEvent — thay đổi thiết bị, mạng, màn hình
type EnvironmentEvent struct {
    BaseEvent
    IP                string `json:"ip"`
    UserAgent         string `json:"user_agent"`
    DeviceFingerprint string `json:"device_fingerprint"`
    ScreenResolution  string `json:"screen_resolution"`
}

// SecurityEvent — devtools, automation detection
type SecurityEvent struct {
    BaseEvent
    IP                string `json:"ip,omitempty"`
    DeviceFingerprint string `json:"device_fingerprint,omitempty"`
}

// LifecycleEvent — exam_start, exam_end
type LifecycleEvent struct {
    BaseEvent
    IP                string `json:"ip"`
    UserAgent         string `json:"user_agent"`
    DeviceFingerprint string `json:"device_fingerprint"`
    ScreenResolution  string `json:"screen_resolution"`
}
```

### Tại sao dùng `Metadata JSONB`?

- Mỗi action có data riêng: `tab_switch` cần `hidden_duration_ms`, `paste_detected` cần `pasted_text_length`
- Thêm action mới chỉ cần define metadata mới, không cần ALTER TABLE
- TimescaleDB query JSONB hiệu quả: `metadata->>'pasted_text_length'`

### Ví dụ Event JSON

**Navigation — Tab Switch (thí sinh chuyển tab):**
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

**Input — Paste Detected (dán text vào writing):**
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

**Answer — Submit (nộp câu trả lời):**
```json
{
  "id": "01912f4f-...",
  "user_id": "user_1234",
  "exam_id": "ielts_mock_0056",
  "session_id": "sess_abc",
  "event_type": "answer",
  "action": "answer_submit",
  "timestamp": "2026-04-06T14:32:45Z",
  "question_id": "reading_q12",
  "section_type": "reading",
  "metadata": {
    "selected_answer": "B",
    "is_correct": "true",
    "time_spent_ms": 14000
  }
}
```

**Environment — Device Change (đổi thiết bị giữa bài thi):**
```json
{
  "id": "01912f50-...",
  "user_id": "user_9999",
  "exam_id": "toeic_official_012",
  "session_id": "sess_def",
  "event_type": "environment",
  "action": "device_change",
  "timestamp": "2026-04-06T15:10:30Z",
  "ip": "203.113.152.10",
  "device_fingerprint": "fp_NEW_abcdef",
  "screen_resolution": "1920x1080",
  "metadata": {
    "previous_fingerprint": "fp_OLD_xyz789",
    "previous_ip": "203.113.152.10"
  }
}
```

**Security — DevTools Open (mở developer tools):**
```json
{
  "id": "01912f4e-...",
  "user_id": "user_5678",
  "exam_id": "toeic_official_012",
  "session_id": "sess_xyz",
  "event_type": "security",
  "action": "devtools_open",
  "timestamp": "2026-04-06T14:40:10Z",
  "ip": "203.113.152.10",
  "device_fingerprint": "fp_abcdef",
  "metadata": {
    "detection_method": "debugger_timing",
    "browser": "Chrome 124"
  }
}
```

**Lifecycle — Exam Start (bắt đầu bài thi):**
```json
{
  "id": "01912f40-...",
  "user_id": "user_1234",
  "exam_id": "ielts_mock_0056",
  "session_id": "sess_abc",
  "event_type": "lifecycle",
  "action": "exam_start",
  "timestamp": "2026-04-06T14:00:00Z",
  "ip": "203.113.152.10",
  "user_agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) Chrome/124",
  "device_fingerprint": "fp_abc123",
  "screen_resolution": "1920x1080",
  "metadata": {
    "exam_type": "ielts_academic",
    "total_sections": 4,
    "time_limit_minutes": 180
  }
}
```

---

## 5. Flow chi tiết: Từ Browser đến Detection

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
  │  - Performance.now() để đo timing chính xác                     │
  │  - Gửi batch mỗi 5 giây hoặc gửi ngay khi có event critical   │
  └──────────────────────┬──────────────────────────────────────────┘
                         │ POST /api/v1/events/batch
                         ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  BƯỚC 2: Anti-Cheat Service nhận event                          │
  │  - Validate required fields                                     │
  │  - Parse event type (behavior / session)                        │
  │  - Response 202 Accepted ngay (non-blocking)                    │
  └──────────────────────┬──────────────────────────────────────────┘
                         │
                         ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  BƯỚC 3: Real-time Check — Lớp 1 (Redis, < 1ms)                │
  │                                                                  │
  │  Critical events xử lý ngay:                                     │
  │  - tab_switch count > 5 trong 10 phút → warning cho thí sinh    │
  │  - tab_hidden > 30 giây → warning + log                         │
  │  - paste > 100 chars trong writing → flag ngay                   │
  │  - devtools_open → flag ngay + cảnh báo nghiêm trọng            │
  │  - automation_signal → void exam ngay                            │
  │  - focus_lost > 10 lần / section → warning                      │
  │                                                                  │
  │  Cơ chế: Redis INCR + Lua script (atomic threshold check)       │
  │  Kết quả: detections → Action Executor (async)                   │
  └──────────────────────┬──────────────────────────────────────────┘
                         │
                         ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  BƯỚC 4: Push vào Redis Event Buffer (LPUSH, ~0.1ms)            │
  │  - Event serialize thành JSON                                    │
  │  - LPUSH vào Redis list "events:buffer"                          │
  │  - Backpressure: LLEN check trước, reject 429 nếu >100K events  │
  └──────────────────────┬──────────────────────────────────────────┘
                         │
                         │  Flush mỗi 60 giây (goroutine ticker)
                         ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  BƯỚC 5: Flush — Reliable Queue Pattern                         │
  │  1. RPOPLPUSH từ "events:buffer" → "events:processing"          │
  │     (Lua script, atomic — không mất event nếu crash)            │
  │  2. Batch INSERT vào TimescaleDB (COPY protocol, 100K+ rows/s)  │
  │  3. DEL "events:processing" chỉ sau khi DB insert thành công    │
  │  4. Nếu DB fail → events ở lại processing list → retry lần sau  │
  └──────────────────────┬──────────────────────────────────────────┘
```

### BƯỚC 4 chi tiết: Tại sao LPUSH?

```
LPUSH = đẩy event vào đầu TRÁI của Redis list (O(1), ~0.1ms)

  LPUSH →  [E_new, ..., ..., ..., E_old]  → RPOP (consumer lấy từ phải)
  (trái)                                     (phải)

→ Event vào trước được xử lý trước (FIFO queue)
→ Producer (API handler) chỉ cần LPUSH, không quan tâm consumer
→ Consumer (flusher) chỉ cần RPOP/RPOPLPUSH từ phải
→ 2 bên hoạt động độc lập = DECOUPLE
```

### BƯỚC 5 chi tiết: Tại sao RPOPLPUSH mà không RPOP?

**RPOPLPUSH = RPOP + LPUSH trong 1 lệnh atomic:**

```
RPOPLPUSH "events:buffer" "events:processing"

= Lấy event từ bên PHẢI của buffer (RPOP)
+ Đẩy vào bên TRÁI của processing (LPUSH)
→ Trong cùng 1 lệnh, không thể bị ngắt giữa chừng
```

**Vấn đề nếu dùng RPOP thông thường:**

```
1. RPOP events:buffer       → event RỜI KHỎI Redis
2. INSERT DB                ← crash ở đây
3. (không bao giờ chạy)

→ Event đã bị xóa khỏi buffer, chưa vào DB → MẤT ✗
```

**RPOPLPUSH giải quyết — event luôn tồn tại ở 1 trong 2 list:**

```
Trước:
  events:buffer:      [E5, E4, E3, E2, E1]
  events:processing:  []

Chạy RPOPLPUSH:
  events:buffer:      [E5, E4, E3, E2]     ← E1 rời đi
  events:processing:  [E1]                  ← E1 nằm ở đây (atomic)
```

**Crash ở bất kỳ bước nào đều an toàn:**

| Crash tại | Event ở đâu? | Recovery |
|-----------|-------------|----------|
| Sau bước 1 (move xong, chưa INSERT) | `events:processing` | Consumer restart → thấy processing list còn data → INSERT lại |
| Sau bước 2 (INSERT xong, chưa DEL) | `events:processing` + DB | Consumer restart → INSERT lại → **ON CONFLICT DO NOTHING** (idempotent) |
| Sau bước 3 (DEL xong) | Chỉ ở DB | Hoàn thành ✓ |

**Không có scenario nào event biến mất.**

### Flush locking: flush tiếp theo khi flush trước chưa xong?

```
Flush lần 1 (đang chạy, INSERT DB mất lâu):
  events:buffer:      [E200K, ..., E100K+1]    ← events mới vẫn LPUSH vào đây
  events:processing:  [E100K, ..., E1]          ← đang INSERT DB

Flush lần 2 (60s sau, timer trigger):
  → Check: LLEN "events:processing" > 0?
  → CÓ → flush trước chưa xong → SKIP lần này
  → Hoặc: retry INSERT từ processing list trước
```

```go
func flush() {
    // Flush trước đã xong chưa?
    len := redis.LLen("events:processing")
    if len > 0 {
        // Chưa xong (hoặc crash lần trước) → retry processing list
        retryFromProcessing()
        return
    }
    // Processing rỗng → flush bình thường
    // RPOPLPUSH → INSERT DB → DEL processing
}
```

`events:processing` **không có TTL, không tự xóa**. Nó là "bằng chứng flush đang diễn ra". Còn data = chưa xong. Rỗng = đã xong.

```
                         │
                         ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  BƯỚC 6: Rule Engine phân tích — Lớp 2 (Batch)                  │
  │                                                                  │
  │  Trigger: sau mỗi flush + khi exam kết thúc                     │
  │  - Answer speed anomaly (trả lời nhanh bất thường)              │
  │  - Tab-then-correct pattern (chuyển tab rồi trả lời đúng)       │
  │  - Paste ratio (bao nhiêu % bài writing là paste?)              │
  │  - Typing rhythm anomaly (gõ đều bất thường = bot)              │
  │  - Cross-exam similarity (2 thí sinh cùng đáp án pattern)       │
  │  - Section timing anomaly (xong section nhanh bất thường)       │
  │  - Device/IP anomaly (đổi thiết bị giữa bài thi)               │
  └──────────────────────┬──────────────────────────────────────────┘
                         │
                         ▼
  ┌─────────────────────────────────────────────────────────────────┐
  │  BƯỚC 7: Action theo severity                                   │
  │                                                                  │
  │  severity=low      → Flag cho admin review sau                   │
  │  severity=medium   → Cảnh báo thí sinh (hiện warning trên UI)   │
  │  severity=high     → Flag exam result + notify admin             │
  │  severity=critical → Void exam result + lock account             │
  │                                                                  │
  │  Thông báo thí sinh: Redis Pub/Sub → JS SDK hiện warning        │
  │  Thông báo admin: Redis Pub/Sub → Admin dashboard               │
  │  Persist: INSERT vào cheat_detections table (audit trail)        │
  └─────────────────────────────────────────────────────────────────┘
```

### Tại sao real-time check TRƯỚC buffer push?

```
Nếu check SAU buffer:
  Event vào buffer → flush sau 60 giây → check → phát hiện devtools_open
  → 60 giây thí sinh đã xem hết đáp án rồi → quá trễ

Nếu check TRƯỚC buffer:
  Event vào → check Redis ngay (< 1ms) → devtools_open → void exam NGAY
  → event vẫn push vào buffer để lưu audit trail
```

---

## 6. Tại sao Redis Buffer + PostgreSQL TimescaleDB?

### Tại sao cần Redis Buffer?

```
          Không có buffer                         Có Redis buffer
    ┌─────────────────────┐                ┌─────────────────────┐
    │  5,000 thí sinh thi  │                │  5,000 thí sinh thi  │
    │  cùng lúc            │                │  cùng lúc            │
    │  ~20,000 events/giây │                │  ~20,000 events/giây │
    │         │            │                │         │            │
    │         ▼            │                │         ▼            │
    │  20,000 INSERT/giây  │                │  20,000 LPUSH/giây   │
    │  vào PostgreSQL      │                │  vào Redis (0.1ms)   │
    │         │            │                │         │            │
    │         ▼            │                │         ▼            │
    │  ❌ DB quá tải       │                │  ~17 BATCH INSERT    │
    │  ❌ Connection pool  │                │  /phút vào PostgreSQL│
    │     cạn kiệt         │                │         │            │
    │  ❌ Exam client lag   │                │         ▼            │
    │  ❌ Peak: TOEIC      │                │  ✅ DB khỏe           │
    │     monthly exam     │                │  ✅ Client nhanh      │
    └─────────────────────┘                │  ✅ DB down = event   │
                                           │     an toàn trong Redis│
                                           └─────────────────────┘
```

**4 lý do cốt lõi:**

| # | Lý do | Chi tiết |
|---|-------|---------|
| 1 | **Client không bị block** | Exam UI phải smooth. LPUSH ~0.1ms vs INSERT 5-50ms. Thí sinh không được cảm thấy lag vì anti-cheat |
| 2 | **Giảm I/O cho DB** | 1.2M events riêng lẻ/phút → ~17 COPY batch/phút. Giảm 99.99% số lần write |
| 3 | **Chịu peak** | TOEIC monthly exam: hàng ngàn thí sinh thi cùng lúc. Redis xử lý 100K+ ops/s |
| 4 | **Decouple** | DB down tạm thời → events vẫn an toàn trong Redis. DB recover → flush bình thường |

### Tại sao PostgreSQL + TimescaleDB?

**Vai trò: TimescaleDB là storage layer** — lưu trữ events và trả data khi Rule Engine query. Logic phân tích nằm trong Rule Engine (Go code), không phải trong DB.

**Nhưng tại sao không dùng PostgreSQL thường?** Vì exam events là time-series data (timestamp là trục chính). TimescaleDB là PostgreSQL extension tối ưu cho time-series, mang lại những lợi ích mà PostgreSQL thường không có:

#### 1. Chunk Pruning — Query nhanh, không scan toàn bộ bảng

```
TimescaleDB tự chia bảng thành chunks theo thời gian (1 chunk = 1 ngày):

  exam_events:
    chunk_2026_04_06  (hôm nay)    ← 1.2M events
    chunk_2026_04_05  (hôm qua)    ← 800K events
    chunk_2026_04_04               ← 500K events
    ... (365 chunks cho 1 năm)

Query: "đếm tab_switch trong 10 phút qua"
  PostgreSQL thường:  scan TOÀN BỘ bảng (hàng trăm triệu rows) → chậm
  TimescaleDB:        chỉ scan chunk hôm nay → nhanh gấp 100x+
```

#### 2. Compression — Nén 90%+, giữ 1 năm audit trail rẻ

```
Không nén:
  10,000 thí sinh/ngày × 365 ngày × ~500 bytes/event × ~200 events/thí sinh
  = ~365 GB/năm → tốn storage, query chậm

Nén (TimescaleDB compression):
  Data cũ hơn 7 ngày tự động nén → còn ~36 GB/năm (giảm 90%+)
  Vẫn query được bình thường (decompress on-read)

Tại sao quan trọng?
  - Đối tác B2B yêu cầu giữ audit trail 1 năm
  - Không nén → storage cost cao, hoặc phải xóa sớm → mất bằng chứng
  - PostgreSQL thường không có native compression cho table data
```

#### 3. Continuous Aggregates — Tính sẵn, Rule Engine không scan raw data

```
Không có continuous aggregate:
  Rule Engine query: "đếm tab_switch của 10,000 user trong 10 phút"
  → Scan 1.2M raw events → mất 5 giây

Có continuous aggregate:
  TimescaleDB tự tính sẵn mỗi phút: "user X có Y events loại Z"
  → Rule Engine query materialized view → mất 50ms (nhanh gấp 100x)

  CREATE MATERIALIZED VIEW exam_event_counts_per_minute
  WITH (timescaledb.continuous) AS
  SELECT time_bucket('1 minute', created_at), user_id, action, COUNT(*)
  FROM exam_events
  GROUP BY ...;

PostgreSQL thường:
  Phải tự tạo materialized view + cron job REFRESH
  → Manual, dễ quên, không incremental (refresh toàn bộ)
```

#### 4. Retention Policy — Tự xóa data cũ, không cần cron job

```
SELECT add_retention_policy('exam_events', INTERVAL '365 days');
→ Tự động drop chunks cũ hơn 1 năm
→ Không cần viết cron DELETE FROM ... WHERE created_at < ...
→ Drop chunk = O(1), DELETE row by row = chậm + lock table
```

#### 5. Window Functions + SQL quen thuộc

```
TimescaleDB là PostgreSQL extension → dùng SQL bình thường:
  - LAG/LEAD: so sánh event trước/sau (tab_switch → answer_submit)
  - STDDEV: typing rhythm bất thường (stddev < 10ms = bot)
  - time_bucket: group by khoảng thời gian (mỗi 1 phút, 5 phút)

Team quen PostgreSQL → không cần học query language mới
  (ClickHouse dùng SQL nhưng khác biệt nhiều)
  (InfluxDB dùng InfluxQL/Flux → học từ đầu)
```

#### 6. Không cần infra mới

```
TimescaleDB = PostgreSQL extension (CREATE EXTENSION timescaledb)
  → Chạy trên cùng PostgreSQL instance
  → Dùng chung connection pool, backup, monitoring
  → Team quen, ops quen

ClickHouse/InfluxDB = cluster riêng
  → Cần setup, monitor, backup riêng
  → Team phải học vận hành hệ thống mới
  → Overkill cho 10K concurrent
```

#### Tổng hợp: TimescaleDB vs PostgreSQL thường

| Vấn đề | PostgreSQL thường | TimescaleDB |
|--------|------------------|-------------|
| Bảng lớn (100M+ rows) | Full table scan → chậm dần | Chunk pruning → chỉ scan chunk cần thiết |
| Data cũ chiếm storage | Phải tự DELETE + VACUUM | Compression 90%+, retention auto drop chunk |
| Pre-computed aggregates | Tự tạo materialized view + cron refresh | Continuous aggregates (incremental, auto) |
| Xóa data cũ hơn 1 năm | DELETE FROM ... (chậm, lock) | Drop chunk O(1) |
| Time-series query | Tự partition, tự optimize | Native: time_bucket, chunk pruning |
| Ops | Quen | Quen (cùng PostgreSQL) |

### So sánh với các lựa chọn khác

| Lựa chọn | Ưu | Nhược | Khi nào dùng |
|----------|-----|-------|-------------|
| **Redis + TimescaleDB (chọn)** | Đơn giản, team quen, đủ scale | Không phải native streaming | 5K–50K concurrent, đội nhỏ |
| Kafka + ClickHouse | Throughput cực cao, near real-time | Ops phức tạp, cần Kafka cluster | 100K+ concurrent, đội lớn |
| Kafka + InfluxDB | Time-series tốt | InfluxDB không mạnh SQL join | IoT, metrics |
| Direct INSERT (không buffer) | Đơn giản nhất | DB quá tải khi peak | POC, < 100 concurrent |

---

## 7. Database Schema

```sql
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- Main event table (hypertable)
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

-- Convert to TimescaleDB hypertable, 1 chunk = 1 ngày
SELECT create_hypertable('exam_events', 'created_at',
    chunk_time_interval => INTERVAL '1 day'
);

-- Indexes cho các query pattern chính
CREATE INDEX idx_events_exam     ON exam_events (exam_id, user_id, created_at DESC);
CREATE INDEX idx_events_session  ON exam_events (session_id, created_at DESC);
CREATE INDEX idx_events_action   ON exam_events (action, created_at DESC);
```

### Tại sao dùng 1 table flat thay vì tách behavior/session?

```
Phương án 1: 2 bảng (behavior_events, session_events)
  ✗ Rule engine cần JOIN 2 bảng (tab_switch + focus_lost cùng session)
  ✗ TimescaleDB hypertable tối ưu nhất khi 1 bảng lớn, không phải nhiều bảng nhỏ
  ✗ COPY batch insert phức tạp hơn (route event vào đúng bảng)

Phương án 2: 1 bảng flat (chọn)
  ✓ Mọi event cùng 1 hypertable → chunk pruning hiệu quả
  ✓ COPY batch đơn giản (1 destination)
  ✓ Cross-type query dễ (behavior + session trong 1 query)
  ✗ Nullable fields (question_id, ip...) — chấp nhận được
```

### Continuous Aggregate

```sql
-- Tự tính sẵn: event count per user per exam per minute
CREATE MATERIALIZED VIEW exam_event_counts_per_minute
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 minute', created_at) AS bucket,
    user_id, exam_id, action, section_type,
    COUNT(*) AS event_count
FROM exam_events
GROUP BY bucket, user_id, exam_id, action, section_type;

-- Policy: tính mỗi 1 phút, cover data từ 10 phút trước đến 1 phút trước
SELECT add_continuous_aggregate_policy('exam_event_counts_per_minute',
    start_offset    => INTERVAL '10 minutes',
    end_offset      => INTERVAL '1 minute',
    schedule_interval => INTERVAL '1 minute'
);
```

**Tại sao dùng continuous aggregate?**
- Rule engine query "đếm tab_switch của user X trong 10 phút" → query materialized view thay vì scan raw data
- Giảm query load đáng kể khi chạy batch analysis cho 5,000+ thí sinh

### Compression & Retention

```sql
ALTER TABLE exam_events SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'exam_id, user_id',
    timescaledb.compress_orderby = 'created_at DESC'
);

-- Compress data cũ hơn 7 ngày (exam đã xong)
SELECT add_compression_policy('exam_events', INTERVAL '7 days');

-- Xóa data cũ hơn 365 ngày (hết thời hạn audit)
SELECT add_retention_policy('exam_events', INTERVAL '365 days');
```

### Cheat detection results

```sql
CREATE TABLE cheat_detections (
    id              UUID        PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id         TEXT        NOT NULL,
    exam_id         TEXT        NOT NULL,
    rule_id         TEXT        NOT NULL,
    severity        TEXT        NOT NULL,     -- 'low', 'medium', 'high', 'critical'
    action_taken    TEXT        NOT NULL,     -- 'flag', 'warn', 'void_result', 'lock_account'
    evidence        JSONB       NOT NULL,     -- bằng chứng chi tiết
    reviewed        BOOLEAN     DEFAULT FALSE,
    reviewed_by     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

---

## 8. Project Structure (Golang)

```
anti-cheat-service/
├── cmd/
│   └── server/
│       └── main.go                  # Entry point, wire dependencies, graceful shutdown
├── internal/
│   ├── handler/
│   │   └── event_handler.go         # HTTP handler nhận events từ JS SDK
│   ├── buffer/
│   │   └── redis_buffer.go          # Redis event buffer + periodic flusher
│   ├── store/
│   │   └── postgres_store.go        # TimescaleDB batch insert (COPY protocol)
│   ├── realtime/
│   │   └── checker.go               # Lớp 1: real-time checks (Redis INCR + Lua)
│   ├── rules/
│   │   ├── engine.go                # Rule engine orchestrator
│   │   ├── tab_switch_pattern.go    # Rule: tab-then-correct
│   │   ├── paste_ratio.go           # Rule: paste ratio trong writing
│   │   ├── answer_speed.go          # Rule: answer speed anomaly
│   │   ├── typing_rhythm.go         # Rule: typing pattern (bot detection)
│   │   └── cross_exam_similarity.go # Rule: similarity giữa 2 thí sinh
│   ├── model/
│   │   ├── event.go                 # BaseEvent, BehaviorEvent, SessionEvent, ExamEvent
│   │   └── detection.go             # Detection result, severity, action enums
│   └── action/
│       ├── executor.go              # Thực thi: flag, warn, void, lock
│       └── notifier.go              # Notify user (Redis Pub/Sub), notify admin
├── migrations/
│   └── 001_create_tables.sql
├── config/
│   └── config.go                    # Config from env vars (12-factor)
├── go.mod
└── go.sum
```

### Dependency flow

```
handler → realtime/checker (Lớp 1, sync)
       → buffer (push event, async)
              → store (batch insert khi flush)
              → rules/engine (batch analysis sau flush)
                     → store (query TimescaleDB)
                     → action/executor (xử lý detection)
                            → store (persist detection)
                            → action/notifier (warn user, alert admin)
```

---

## 9. Interview Quick Reference

### Elevator Pitch (30 giây)

> "Em thiết kế anti-cheat service cho nền tảng thi IELTS/TOEIC online, đáp ứng 5,000–10,000 thí sinh đồng thời. Kiến trúc 2 lớp: **Lớp 1 dùng Redis counters** phát hiện gian lận real-time < 1ms (chuyển tab, paste, devtools). **Lớp 2: Rule Engine query TimescaleDB** để phân tích pattern phức tạp sau mỗi 60 giây (tab-then-correct, typing rhythm, cross-exam similarity) — TimescaleDB là storage layer, logic phân tích nằm trong Rule Engine (Go code). Redis buffer decouple ingestion khỏi DB — exam client không bị lag bởi anti-cheat."

### "Tại sao Redis + TimescaleDB?"

> "Redis cho 2 việc: (1) real-time counter cho Lớp 1 detection, (2) event buffer decouple ingestion khỏi DB write. TimescaleDB cho persistent storage — exam events là time-series, chunk pruning và continuous aggregates giúp query nhanh. Logic phân tích nằm trong Rule Engine (Go code), TimescaleDB chỉ là nơi lưu trữ và trả data. Đơn giản hơn Kafka+ClickHouse mà đủ cho 10K concurrent."

### "Flow tổng thể?"

> "JS SDK batch gửi events mỗi 5 giây → Service nhận, chạy real-time check (Redis INCR, < 1ms) → push vào Redis buffer → flush mỗi 60s bằng COPY protocol vào TimescaleDB → trigger batch rule engine → detections xử lý theo severity. Real-time check chạy TRƯỚC buffer push để bắt critical events ngay."

### "Có những loại event nào?"

> "6 nhóm: (1) **Navigation** — chuyển tab, mất focus, mouse leave — volume cao, tín hiệu gian lận phổ biến nhất. (2) **Input** — paste, typing pattern, keyboard shortcut — phát hiện copy-paste và bot. (3) **Answer** — nộp câu trả lời — volume thấp nhưng giá trị phân tích cao nhất (cross-reference với Navigation → tab-then-correct). (4) **Environment** — đổi device, IP, screen — phát hiện nhờ người khác làm. (5) **Security** — devtools, automation — severity cao nhất, gần 0 false positive. (6) **Lifecycle** — exam start/end — capture baseline và trigger batch analysis."

### "Tại sao không chia 2 loại behavior/session?"

> "Vì tab_switch và answer_submit tuy đều là 'hành vi' nhưng detection logic khác hoàn toàn. devtools_open và ip_change tuy đều là 'session' nhưng severity khác xa. Chia 6 nhóm giúp rule engine áp strategy pattern theo nhóm, dashboard admin chia tab monitor rõ ràng, và thêm event mới biết ngay thuộc nhóm nào → detection logic nào."

### "Tại sao 2 lớp detection?"

> "Lớp 1 bắt hành vi rõ ràng ngay (devtools, automation) — không cần context. Lớp 2 phân tích pattern cần cross-reference nhiều events (chuyển tab rồi đúng 8/8 câu) — cần data tích lũy trong DB. Tách 2 lớp vì latency requirement khác nhau: < 1ms vs 60 giây."

---

### "Tại sao chọn Golang thay vì PHP hay Java?"

| | PHP | Java/Spring | Go |
|---|---|---|---|
| Concurrency | Mỗi request 1 process/thread — nặng | Thread nặng (~1MB), thread pool phức tạp | Goroutine nhẹ (~4KB), tạo hàng nghìn dễ dàng |
| GC pause | Không có GC (request-scoped) | Stop-the-world có thể > 10ms | GC pause < 1ms — đảm bảo real-time check |
| Long-running process | Không phù hợp — thiết kế request/response | Được, nhưng nặng | Phù hợp — binary chạy liên tục 24/7 |
| Pipeline / background job | Cần thêm queue worker riêng (Laravel Queue) | Được nhưng boilerplate nhiều | Goroutine + channel built-in, gọn |
| Graceful shutdown | Khó — không có lifecycle hook chuẩn | ShutdownHook có nhưng phức tạp | `context` + `signal.NotifyContext` tự nhiên |
| Memory footprint | Thấp nhưng không có concurrency tốt | JVM overhead ~200MB+ khi start | Binary nhỏ, start nhanh, RAM thấp |
| Startup time | Nhanh | Chậm (JVM warm-up) | Rất nhanh — phù hợp container/K8s |

**Tóm lại:**
- **PHP**: không phù hợp cho long-running service cần xử lý 20K events/giây liên tục
- **Java**: được, nhưng GC pause có thể vi phạm SLA real-time < 1ms, boilerplate nhiều hơn
- **Go**: goroutine + context + select khớp tự nhiên với pipeline architecture của bài toán này
