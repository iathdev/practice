# Offline-First Exam — Sync Challenges & Conflict Resolution

> Client làm bài offline, server là source of truth. Ở giữa là sync layer — nơi mọi thứ có thể sai: mất mạng, duplicate, clock drift, data corrupt. File này đi sâu vào từng thử thách sync và cách giải quyết.

---

## Mục lục

1. [Thử thách 1: Idempotent Sync — Gửi 2 lần không sao](#1-thử-thách-1-idempotent-sync--gửi-2-lần-không-sao)
2. [Thử thách 2: Conflict Resolution — Cùng câu hỏi, khác đáp án](#2-thử-thách-2-conflict-resolution--cùng-câu-hỏi-khác-đáp-án)
3. [Thử thách 3: Clock Drift — Client clock sai](#3-thử-thách-3-clock-drift--client-clock-sai)
4. [Thử thách 4: Timer Integrity — Thí sinh gian lận thời gian](#4-thử-thách-4-timer-integrity--thí-sinh-gian-lận-thời-gian)
5. [Thử thách 5: Large Sync Payload — Offline lâu, queue lớn](#5-thử-thách-5-large-sync-payload--offline-lâu-queue-lớn)
6. [Thử thách 6: Partial Sync — Sync giữa chừng thì mất mạng](#6-thử-thách-6-partial-sync--sync-giữa-chừng-thì-mất-mạng)
7. [Thử thách 7: Stale Exam Token — Token hết hạn khi offline](#7-thử-thách-7-stale-exam-token--token-hết-hạn-khi-offline)
8. [Interview Quick Reference](#8-interview-quick-reference)

---

## 1. Thử thách 1: Idempotent Sync — Gửi 2 lần không sao

### Bài toán

```
Client gửi sync batch:
  POST /sync { answers: [A1, A2, A3] }
  → Server nhận, INSERT 3 answers
  → Server trả 200 OK
  → Network glitch: response không đến client
  → Client nghĩ sync fail → retry
  → POST /sync { answers: [A1, A2, A3] } lần 2
  → Nếu không idempotent → 6 answers thay vì 3 → data sai
```

### Giải pháp: sync_id + UNIQUE constraint

```
Mỗi answer change có sync_id (UUID), client generate:
  { sync_id: "uuid-1", question_id: "q12", answer: "B" }

Server INSERT:
  INSERT INTO exam_answers (sync_id, question_id, answer, ...)
  VALUES (...)
  ON CONFLICT (session_id, sync_id) DO NOTHING;
  → Gửi 2 lần → insert 1 lần → safe

Server response trả accepted[]:
  { accepted: ["uuid-1", "uuid-2", "uuid-3"] }
  → Client chỉ xóa sync_queue entries có sync_id trong accepted[]
  → Nếu response miss → client retry → server deduplicate → eventually consistent
```

### Tại sao sync_id ở client, không phải server generate?

```
Server generate ID:
  ✗ Client gửi request → server generate ID → response miss
  ✗ Client retry → server generate ID MỚI → duplicate insert
  ✗ Client không biết server đã nhận hay chưa

Client generate sync_id:
  ✓ Client biết chính xác ID nào đã gửi
  ✓ Retry cùng sync_id → server deduplicate
  ✓ Client control lifecycle: pending → synced → removed
```

---

## 2. Thử thách 2: Conflict Resolution — Cùng câu hỏi, khác đáp án

### Bài toán

```
Thí sinh đổi đáp án câu 12:
  10:05:00 - Chọn "B" → sync_queue entry (seq=47)
  10:05:30 - Đổi sang "C" → sync_queue entry (seq=48)
  10:05:35 - Đổi lại "B" → sync_queue entry (seq=49)

3 sync entries cho cùng question_id = "reading_q12"
Sync batch gửi cả 3 → server nhận cả 3 → đáp án cuối cùng là gì?
```

### Giải pháp: Last-Write-Wins bằng sequence number

```
Server lưu TẤT CẢ answer changes (INSERT, không UPDATE):
  exam_answers: [
    { sync_id: "uuid-1", question_id: "q12", answer: "B", sequence: 47 },
    { sync_id: "uuid-2", question_id: "q12", answer: "C", sequence: 48 },
    { sync_id: "uuid-3", question_id: "q12", answer: "B", sequence: 49 }
  ]

Chấm bài dùng latest_answers (view):
  DISTINCT ON (session_id, question_id)
  ORDER BY sequence DESC
  → question_id: "q12", answer: "B", sequence: 49 ✓

Tại sao sequence, không phải timestamp?
  - Timestamp phụ thuộc clock → client clock sai = order sai
  - Sequence là monotonic counter → luôn tăng → order chính xác
  - Client tự quản lý: mỗi answer change → sequence++
```

### Tại sao không có multi-user conflict?

```
Bài thi khác collaborative editing:
  Google Docs: user A sửa dòng 5, user B sửa dòng 5 → CONFLICT
  Exam: mỗi thí sinh chỉ sửa ĐÁP ÁN CỦA MÌNH → KHÔNG conflict giữa users

  Conflict duy nhất: 1 user, cùng question, nhiều changes
  → Last-Write-Wins đủ tốt (thí sinh muốn đáp án cuối cùng mình chọn)

  Đây là lý do offline-first PHÙ HỢP cho bài thi:
  - Single-writer per answer → no cross-user conflict
  - Simple resolution: sequence-based LWW
  - Không cần CRDT, OT, hay consensus phức tạp
```

---

## 3. Thử thách 3: Clock Drift — Client clock sai

### Bài toán

```
Client clock nhanh 5 phút:
  Client: 10:35:00 (thực tế: 10:30:00)
  → answered_at = 10:35:00
  → Server nhận lúc 10:30:05 (server time)
  → Answer timestamp TRƯỚC server receive time? Bất thường

Client clock chậm 10 phút:
  Client: 10:20:00 (thực tế: 10:30:00)
  → answered_at = 10:20:00
  → Trông như thí sinh trả lời CŨ hơn thực tế
  → Timer tính sai → thí sinh có thêm 10 phút gian lận?
```

### Giải pháp: Server timestamp + Clock offset tracking

```
Mỗi lần sync, server tính clock offset:

  client_timestamp:        10:35:00.000
  server_received_at:      10:30:05.500
  offset = client - server = +4 phút 54.5 giây (client nhanh)

Lưu offset vào sync_log:
  → Track offset theo thời gian → detect drift pattern

Chấm bài:
  → Dùng sequence number cho ordering (KHÔNG dùng timestamp)
  → Timestamp chỉ dùng cho audit/analysis
  → answered_at có thể adjusted: answered_at - offset = server-equivalent time

Server response trả server_timestamp:
  → Client có thể tự tính offset
  → Nếu offset > 5 giây → hiện warning cho thí sinh: "Đồng hồ thiết bị có thể chưa chính xác"
```

### Tại sao không bắt buộc client sync clock?

```
Bắt buộc sync clock:
  ✗ Cần NTP protocol → phức tạp cho browser
  ✗ Offline → không sync được
  ✗ Thí sinh không control được system clock

Chấp nhận drift + compensate:
  ✓ Track offset mỗi lần sync
  ✓ Dùng sequence cho ordering (không phụ thuộc clock)
  ✓ Adjusted timestamp cho analysis khi cần
  ✓ Đơn giản, robust
```

---

## 4. Thử thách 4: Timer Integrity — Thí sinh gian lận thời gian

### Bài toán

```
Timer chạy local trên client. Thí sinh có thể:
  1. Sửa system clock → timer chạy chậm hoặc dừng
  2. DevTools → pause JavaScript → timer đứng
  3. Hibernate laptop → timer dừng khi sleep
  4. Mở 2 tab → timer chạy trên tab không active (throttled)

Bài thi 120 phút → thí sinh làm 180 phút → không công bằng
```

### Giải pháp: Dual Timer (Client + Server) + Wall-clock check

**Timer trên Client (UX):**

```
Không dùng setInterval đếm mỗi giây (drift khi tab inactive):

// SAI: drift khi tab bị throttle
setInterval(() => { remaining -= 1000; }, 1000);

// ĐÚNG: tính từ wall-clock
const deadline = startTime + durationMs;
setInterval(() => {
  remaining = deadline - Date.now();  // luôn chính xác theo system clock
  if (remaining <= 0) handleTimeUp();
}, 1000);
```

**Phát hiện system clock manipulation:**

```
Mỗi giây, check:
  expected elapsed = interval count * 1000ms
  actual elapsed = Date.now() - start_time

  Nếu |expected - actual| > 3 giây:
    → Clock đã bị sửa hoặc system sleep
    → Recalculate remaining từ wall-clock
    → Log anomaly event cho anti-cheat

Performance.now() (monotonic, không bị ảnh hưởng bởi system clock change):
  → Dùng Performance.now() để detect clock jump
  → Nếu Date.now() jump > 5 giây nhưng Performance.now() chỉ +1 giây
    → System clock bị sửa → flag
```

**Timer trên Server (Source of truth):**

```
Server biết:
  - started_at: khi thí sinh bắt đầu (Phase 1)
  - duration: thời lượng bài thi
  - deadline_at = started_at + duration

Server enforce:
  - Mỗi sync request: check server_time <= deadline_at + grace_period
  - Submit request: check server_time <= deadline_at + 5 phút (grace)
  - Quá deadline + grace → reject sync, auto-submit với answers đã có

Mỗi sync response trả server_timer_remaining:
  → Client điều chỉnh display timer nếu drift > 2 giây với server
  → Thí sinh thấy timer adjust → biết hệ thống track thời gian

Grace period 5 phút:
  → Chấp nhận clock drift nhỏ + network delay
  → KHÔNG cho phép làm thêm 5 phút → chỉ chấp nhận answers
    đã trả lời trước deadline (based on sequence)
```

**Hibernate / Sleep detection:**

```
Khi tab regain visibility (Page Visibility API):
  1. Check: elapsed = Date.now() - last_visibility_time
  2. Nếu elapsed > 30 giây → có khả năng laptop sleep
  3. Recalculate timer từ server deadline (nếu online)
     hoặc từ local wall-clock (nếu offline)
  4. Log sleep event cho anti-cheat
  5. Nếu timer đã hết trong lúc sleep → trigger time_up ngay
```

---

## 5. Thử thách 5: Large Sync Payload — Offline lâu, queue lớn

### Bài toán

```
Thí sinh mất mạng 30 phút (cả phần Reading):
  → 40 câu × trung bình 2 changes/câu = 80 sync entries
  → Writing: 2 tasks × nhiều typing changes = có thể 200+ entries
  → Tổng: 200-300 entries tích lũy

Online lại → flush tất cả → 1 request lớn:
  - Timeout risk: request lớn + mạng yếu → timeout
  - Server memory: parse large JSON payload
  - Nếu fail giữa chừng → toàn bộ batch retry
```

### Giải pháp: Chunked Sync

```
sync_queue có 250 entries → chia chunks:

Chunk 1: entries 1-50   → POST /sync (batch 1)
  ← 200 OK, accepted: [1..50]
  → Remove entries 1-50 from queue

Chunk 2: entries 51-100 → POST /sync (batch 2)
  ← 200 OK, accepted: [51..100]
  → Remove entries 51-100 from queue

...

Chunk 5: entries 201-250 → POST /sync (batch 5)
  ← Network error
  → Entries 201-250 ở lại queue → retry sau

Kết quả: 200/250 entries synced, 50 entries retry
  → Tốt hơn retry toàn bộ 250
```

### Chunk size

```
Mặc định: 50 entries per chunk
Adaptive:
  - Mạng tốt (response < 500ms) → tăng chunk size lên 100
  - Mạng yếu (response > 2s) → giảm chunk size xuống 20
  - Timeout → giảm chunk size xuống 10

Writing section:
  - 1 entry có thể chứa vài KB text (essay)
  - Chunk by total payload size thay vì count
  - Max payload: 100 KB per chunk
```

---

## 6. Thử thách 6: Partial Sync — Sync giữa chừng thì mất mạng

### Bài toán

```
Client gửi batch [A1, A2, A3, A4, A5]:
  → Server nhận request, bắt đầu insert
  → INSERT A1 OK
  → INSERT A2 OK
  → INSERT A3 OK
  → Network drop giữa chừng
  → Server: transaction commit OK (A1-A5 inserted)
  → Client: không nhận response → nghĩ fail → retry

Hoặc:
  → Server: connection drop TRƯỚC commit
  → Transaction rollback → A1-A5 KHÔNG insert
  → Client retry → OK, insert lại → không mất data
```

### Giải pháp: Transaction + Idempotent retry

```
Server side:
  BEGIN;
    INSERT INTO exam_answers ... ON CONFLICT (session_id, sync_id) DO NOTHING;
    INSERT INTO exam_answers ... ON CONFLICT (session_id, sync_id) DO NOTHING;
    ...
  COMMIT;
  → Tất cả hoặc không gì → atomic

Client side:
  Không nhận response → retry toàn bộ batch
  → Server: sync_id duplicate → ON CONFLICT DO NOTHING → safe
  → Response: accepted[] chứa cả entries đã insert lần trước

Kết quả:
  - Server commit trước khi drop → entries đã insert → retry = DO NOTHING → OK
  - Server rollback → entries chưa insert → retry = insert fresh → OK
  - Cả 2 case đều correct nhờ idempotency
```

---

## 7. Thử thách 7: Stale Exam Token — Token hết hạn khi offline

### Bài toán

```
Bài thi 120 phút. Exam token (JWT) expire sau 150 phút (buffer 30 phút).
Thí sinh mất mạng 90 phút → online lại → sync:
  → Token đã expire → 401 Unauthorized → sync fail
  → Đáp án ở local nhưng không gửi được lên server
```

### Giải pháp: Token refresh + Graceful expiry

```
Token strategy:
  - Exam token expire: exam_duration + 30 phút grace
  - Mỗi sync thành công → server trả refreshed token (nếu gần expire)
  - Token chứa: user_id, exam_id, session_id, deadline_at

Offline quá lâu → token expire:
  1. Sync nhận 401
  2. Client gửi refresh request: POST /auth/exam-token/refresh
     { session_id, device_fingerprint }
  3. Server verify: session exists + user matches + within grace period
     → Issue new token
  4. Client retry sync với token mới

Quá grace period (deadline + 30 phút):
  → Server reject refresh
  → Client hiện: "Phiên thi đã kết thúc. Liên hệ giám thị."
  → Nhưng: đáp án vẫn ở local IndexedDB → recovery possible bởi admin
```

### Admin recovery flow

```
Thí sinh đến giám thị: "Em mất mạng, không nộp được bài"
  1. Admin verify: exam_session tồn tại, status = IN_PROGRESS
  2. Admin trigger: "Accept late submission" cho session này
  3. Server issue special recovery token
  4. Thí sinh mở browser → client detect recovery mode
     → Flush sync_queue với recovery token
  5. Server accept answers, mark session as SUBMITTED_LATE
  6. Chấm bài bình thường nhưng flag là late submission
```

---

## 8. Interview Quick Reference

### "Sync có vấn đề gì nếu gửi 2 lần?"

> "Idempotent by design. Mỗi answer change có sync_id (UUID) do client generate. Server INSERT với ON CONFLICT DO NOTHING trên (session_id, sync_id). Gửi 2 lần → insert 1 lần. Client chỉ xóa sync_queue khi nhận accepted[] từ server."

### "Conflict resolution thế nào?"

> "Last-Write-Wins bằng sequence number (monotonic counter). Server lưu TẤT CẢ answer changes, chấm bài dùng DISTINCT ON question_id ORDER BY sequence DESC. Không cần CRDT vì bài thi là single-writer per answer — không có cross-user conflict."

### "Clock drift thì sao?"

> "Track offset mỗi lần sync: client_timestamp - server_timestamp. Ordering dùng sequence number (không phụ thuộc clock). Timer trên client dùng wall-clock (Date.now()) kết hợp Performance.now() để detect clock manipulation. Server timer là source of truth — enforce deadline + 5 phút grace."

### "Timer gian lận thế nào?"

> "Dual timer: client timer cho UX (wall-clock based, detect sleep/clock jump), server timer cho enforcement (deadline_at). Mỗi sync, server trả server_timer_remaining → client adjust. Quá deadline + grace → server reject, auto-submit với answers đã có."

### "Offline lâu, queue lớn?"

> "Chunked sync: chia 50 entries/chunk, sync lần lượt. Mỗi chunk thành công → xóa khỏi queue. Chunk fail → retry, không ảnh hưởng chunks đã sync. Adaptive chunk size: mạng tốt → chunk lớn hơn, mạng yếu → chunk nhỏ hơn."

### Bảng quyết định

| Quyết định | Chọn | Tại sao |
|-----------|------|---------|
| Idempotency key | Client-generated sync_id (UUID) | Server-generated → response miss → duplicate |
| Conflict resolution | Sequence-based LWW | Single-writer, no cross-user conflict, đơn giản |
| Clock handling | Track offset, dùng sequence cho order | Không force NTP, chấp nhận drift + compensate |
| Timer source of truth | Server deadline_at | Client clock có thể bị sửa |
| Large queue | Chunked sync (50/chunk) | Partial failure không ảnh hưởng chunks đã sync |
| Token expiry | Auto-refresh + grace period + admin recovery | Offline lâu không mất bài |
