# Offline-First Exam — Reliability, Edge Cases & Data Integrity

> Offline-first nghe đơn giản: làm bài local, sync lên server. Nhưng edge cases là nơi hệ thống production khác toy project: browser crash, IndexedDB corrupt, thí sinh mở 2 tab, submit offline, data mất — và cách thiết kế để không bao giờ mất bài thi.

---

## Mục lục

1. [Thử thách 1: Browser Crash — Mất bài giữa chừng](#1-thử-thách-1-browser-crash--mất-bài-giữa-chừng)
2. [Thử thách 2: IndexedDB Corrupt / Bị xóa](#2-thử-thách-2-indexeddb-corrupt--bị-xóa)
3. [Thử thách 3: Multi-Tab — Thí sinh mở 2 tab thi](#3-thử-thách-3-multi-tab--thí-sinh-mở-2-tab-thi)
4. [Thử thách 4: Submit khi Offline — Nộp bài không có mạng](#4-thử-thách-4-submit-khi-offline--nộp-bài-không-có-mạng)
5. [Thử thách 5: Exam Content Integrity — Đề thi bị tamper](#5-thử-thách-5-exam-content-integrity--đề-thi-bị-tamper)
6. [Thử thách 6: Server-Side Validation — Không tin client](#6-thử-thách-6-server-side-validation--không-tin-client)
7. [Thử thách 7: Network Detection — Biết lúc nào online/offline](#7-thử-thách-7-network-detection--biết-lúc-nào-onlineoffline)
8. [Thử thách 8: Data Recovery — Khôi phục khi mất data](#8-thử-thách-8-data-recovery--khôi-phục-khi-mất-data)
9. [Thử thách 9: Concurrent Exams — 5,000 thí sinh cùng sync](#9-thử-thách-9-concurrent-exams--5000-thí-sinh-cùng-sync)
10. [Thử thách 10: Versioning — Cập nhật client khi đang thi](#10-thử-thách-10-versioning--cập-nhật-client-khi-đang-thi)
11. [Interview Quick Reference](#11-interview-quick-reference)

---

## 1. Thử thách 1: Browser Crash — Mất bài giữa chừng

### Bài toán

```
Thí sinh đang làm bài:
  - Đã trả lời 30/40 câu Reading
  - Browser crash (out of memory, OS kill tab, power off)
  - Mở lại browser → bài thi đâu rồi?

Nếu đáp án chỉ ở memory (React state):
  → Crash = mất hết 30 câu → thí sinh phải làm lại
  → Không thể chấp nhận được
```

### Giải pháp: Write-Through to IndexedDB

```
Mỗi lần thí sinh chọn/thay đổi đáp án:
  1. Update React state (UI phản hồi ngay)
  2. Write to IndexedDB "answers" store (persist ngay)
  3. Push to IndexedDB "sync_queue" (chờ sync)

  Bước 2 là CRITICAL: IndexedDB persist qua crash/restart
  → Browser crash → reopen → read IndexedDB → restore toàn bộ đáp án

Write-through, KHÔNG write-back:
  Write-back: batch write mỗi 30 giây → crash giữa chừng = mất 30 giây data
  Write-through: mỗi change write ngay → crash = mất tối đa 1 thao tác cuối

IndexedDB write latency:
  - Đơn lẻ: 1-5ms (chấp nhận được, user không cảm nhận)
  - Không block UI vì IndexedDB API là async
```

### Recovery Flow

```
Mở browser sau crash:
  1. Check IndexedDB "exam_state" → có exam IN_PROGRESS?
  2. Có → read exam_content, answers, timer state
  3. Recalculate timer: remaining = deadline - Date.now()
     (deadline đã lưu trong exam_state)
  4. Nếu timer > 0 → restore exam UI, thí sinh tiếp tục
  5. Nếu timer <= 0 → exam đã hết giờ → trigger submit flow
  6. Resume sync engine → flush pending sync_queue

UI: hiện "Phiên thi đã được khôi phục" (1 giây) → tiếp tục làm bài
```

### Lưu exam_state thường xuyên

```
exam_state lưu gì:
  {
    status: "IN_PROGRESS",
    current_section: "reading",
    current_question_index: 29,
    timer_deadline: 1712401200000,     // absolute deadline (ms)
    last_updated_at: 1712400900000
  }

Khi nào update exam_state:
  - Chuyển câu hỏi → update current_question_index
  - Chuyển section → update current_section
  - Mỗi 30 giây → update last_updated_at (heartbeat)

Crash recovery → restore đến đúng câu hỏi thí sinh đang làm
```

---

## 2. Thử thách 2: IndexedDB Corrupt / Bị xóa

### Bài toán

```
IndexedDB có thể mất vì:
  1. User clear browser data ("Clear browsing data")
  2. Browser incognito/private mode → đóng = mất
  3. Storage pressure: browser tự xóa khi ổ cứng đầy
  4. IndexedDB corruption (rare nhưng xảy ra)
  5. Antivirus/cleaner tools xóa browser data

Hậu quả:
  → Exam content mất → không hiện đề
  → Answers mất → đáp án biến mất
  → Sync queue mất → answers chưa sync lên server = mất vĩnh viễn
```

### Giải pháp: Defense in Depth — 3 lớp bảo vệ

```
Lớp 1: Persistent Storage API
  → navigator.storage.persist()
  → Yêu cầu browser KHÔNG tự xóa data
  → Hầu hết browser: granted nếu user đã tương tác
  → Không 100% guarantee nhưng giảm risk đáng kể

Lớp 2: Server đã có answers (từ sync)
  → Sync mỗi 10 giây → server có hầu hết answers
  → IndexedDB mất → client query server: GET /exams/{id}/my-answers
  → Restore answers từ server → tiếp tục làm bài

Lớp 3: Incognito detection + Warning
  → Detect private browsing mode
  → Hiện warning: "Bạn đang dùng chế độ ẩn danh.
    Dữ liệu có thể mất khi đóng trình duyệt.
    Vui lòng dùng trình duyệt thường."
```

### Recovery khi IndexedDB mất

```
Mở browser → IndexedDB empty hoặc corrupt:
  1. Detect: exam_state không tồn tại nhưng server session = IN_PROGRESS
  2. Online?
     Có → GET /exams/{id}/content → re-download đề
          GET /exams/{id}/my-answers → restore answers từ server
          Resume exam
     Không → Hiện: "Dữ liệu bài thi bị mất. Vui lòng kết nối mạng để khôi phục."
             Khi online → auto-recover

  Answers đã sync lên server → không mất
  Answers CHƯA sync (trong sync_queue) → MẤT
  → Đây là lý do sync thường xuyên (mỗi 10 giây) quan trọng
  → Worst case: mất 10 giây đáp án cuối cùng
```

---

## 3. Thử thách 3: Multi-Tab — Thí sinh mở 2 tab thi

### Bài toán

```
Thí sinh mở 2 tab cùng bài thi:
  Tab 1: đang làm câu 12, chọn "B"
  Tab 2: đang làm câu 12, chọn "C"

  Cả 2 tab write vào IndexedDB → data conflict
  Cả 2 tab sync lên server → server nhận 2 đáp án khác nhau
  Timer chạy trên cả 2 tab → cái nào đúng?
```

### Giải pháp: Single-Tab Lock (BroadcastChannel API)

```
Khi mở exam page:
  1. Tạo BroadcastChannel("exam_lock")
  2. Post message: { type: "CLAIM_LOCK", tab_id: "random-uuid", timestamp }
  3. Listen for responses

  Nếu tab khác đang active:
    → Tab cũ nhận CLAIM_LOCK → reply: { type: "LOCK_HELD", tab_id: "old-tab" }
    → Tab mới hiện: "Bài thi đang mở ở tab khác. Đóng tab này hoặc chuyển sang tab kia."
    → Option 1: User đóng tab mới
    → Option 2: User nhấn "Chuyển sang đây" → tab mới gửi FORCE_CLAIM
       → Tab cũ nhận FORCE_CLAIM → đóng exam, hiện "Bài thi đã chuyển sang tab khác"

Fallback nếu BroadcastChannel không hỗ trợ:
  - IndexedDB lock: write { tab_id, heartbeat_at } mỗi 5 giây
  - Tab mới check: nếu heartbeat_at < 10 giây trước → tab cũ còn sống → block
  - Nếu heartbeat_at > 10 giây → tab cũ đã chết → claim lock
```

### Tại sao không cho phép multi-tab?

```
Multi-tab cho phép:
  ✗ IndexedDB: concurrent write từ 2 tab → potential corruption
  ✗ Sync engine: 2 tab sync cùng lúc → duplicate data, race condition
  ✗ Timer: 2 tab đếm ngược → inconsistent display
  ✗ Anti-cheat: 2 tab = hành vi đáng ngờ (mở tab 2 để tra cứu?)

Single-tab enforce:
  ✓ 1 source of truth cho IndexedDB writes
  ✓ 1 sync engine → no duplicate sync
  ✓ 1 timer → consistent
  ✓ Đơn giản, ít bug
```

---

## 4. Thử thách 4: Submit khi Offline — Nộp bài không có mạng

### Bài toán

```
Timer hết → thí sinh nhấn "Nộp bài" → offline:
  → Sync_queue vẫn có pending answers
  → POST /submit fail → không nộp được
  → Thí sinh: "Đã hết giờ nhưng không nộp được bài!"
```

### Giải pháp: Submit Intent + Auto-submit + Grace Period

```
Bước 1: Lưu submit intent local
  → IndexedDB exam_state: { status: "SUBMITTING", submit_requested_at }
  → UI: "Bài thi đã lưu. Đang chờ kết nối để nộp..."
  → Thí sinh có thể đóng browser → data an toàn trong IndexedDB

Bước 2: Auto-submit khi online
  → Service Worker detect online
  → Hoặc: mở lại browser → check exam_state = SUBMITTING
  → Flush sync_queue → POST /submit

Bước 3: Server-side grace period
  → Server chấp nhận submit trong deadline + 5 phút
  → Quá 5 phút → server auto-submit với answers đã sync trước đó
  → Thí sinh không bị mất bài, chỉ có thể mất vài answers cuối
     (những cái chưa sync)
```

### Server Auto-Submit (safety net)

```
Scheduled job trên server, chạy mỗi phút:
  SELECT * FROM exam_sessions
  WHERE status = 'IN_PROGRESS'
    AND deadline_at + INTERVAL '5 minutes' < NOW();

  Với mỗi session quá hạn:
    1. Mark status = 'AUTO_SUBMITTED'
    2. Chấm bài với answers đã sync (latest_answers view)
    3. Log: auto-submitted, reason: deadline_exceeded

Thí sinh không cần làm gì:
  - Nếu đã sync đủ answers → điểm chính xác
  - Nếu chưa sync hết → mất vài answers cuối
  - Nhưng KHÔNG BAO GIỜ mất toàn bộ bài thi
```

### Flow tổng hợp: Submit scenarios

```
Scenario 1: Online, submit OK
  Timer hết → flush sync_queue → POST /submit → 200 OK → Done

Scenario 2: Online, submit fail (server error)
  Timer hết → flush → POST /submit → 500 → retry 3 lần
  → Vẫn fail → lưu submit intent → retry sau → server grace period bảo vệ

Scenario 3: Offline khi submit
  Timer hết → lưu submit intent → "Đang chờ kết nối..."
  → Online lại trong 5 phút → flush + submit → OK
  → Online lại sau 5 phút → flush + submit → server check grace
    → Trong grace → OK
    → Quá grace → server đã auto-submit → trả: "Bài đã được nộp tự động"

Scenario 4: Thí sinh đóng browser khi offline
  → Data ở IndexedDB
  → Mở lại → detect SUBMITTING state → auto-retry
  → Hoặc: server auto-submit sau grace period
```

---

## 5. Thử thách 5: Exam Content Integrity — Đề thi bị tamper

### Bài toán

```
Đề thi lưu ở IndexedDB → client-side → có thể bị sửa:
  1. DevTools → Application → IndexedDB → sửa trực tiếp
  2. Script injection → modify exam content
  3. Man-in-the-middle → sửa response khi download

Hậu quả:
  - Thí sinh sửa đề để dễ hơn
  - Thí sinh xem đáp án (nếu đáp án trong content)
  - Đề thi khác giữa các thí sinh → không công bằng
```

### Giải pháp

```
1. KHÔNG bao giờ gửi đáp án đúng trong exam content
   → Server chấm bài, không phải client
   → Exam content chỉ chứa: câu hỏi + choices, KHÔNG chứa correct answer
   → Dù client bị tamper → không xem được đáp án

2. Checksum verify
   → Server gửi exam content + SHA-256 checksum
   → Client verify checksum sau khi download
   → Mỗi lần load exam từ IndexedDB → verify lại checksum
   → Checksum mismatch → re-download hoặc block exam

3. HTTPS (TLS)
   → Chống MITM: content không bị sửa trên đường truyền

4. Content Signing (optional, high-security)
   → Server sign content bằng private key
   → Client verify bằng public key (embedded in app)
   → Tamper IndexedDB → signature mismatch → block
```

### Đáp án ở đâu?

```
KHÔNG ở client:
  ✗ Exam content KHÔNG chứa correct answers
  ✗ Client KHÔNG tự chấm bài

Ở server:
  ✓ Server giữ answer key riêng
  ✓ Chấm bài: so sánh exam_answers với answer key trên server
  ✓ Client chỉ biết kết quả sau khi server chấm + trả về

Tại sao không cho client tự chấm (offline)?
  → Nếu gửi answer key xuống client → thí sinh xem được
  → Dù encrypt → client có key để decrypt → reverse engineer được
  → Server chấm = an toàn duy nhất
```

---

## 6. Thử thách 6: Server-Side Validation — Không tin client

### Bài toán

```
Client gửi gì lên cũng có thể bị tamper:
  - answered_at: sửa timestamp để trông như trả lời nhanh
  - sequence: sửa sequence number
  - client_timer_remaining: fake timer
  - answer: gửi answer cho câu hỏi không tồn tại
```

### Server validate mọi sync request

```
1. Session validation:
   - exam_session exists + status = IN_PROGRESS
   - Token user_id matches session user_id
   - Exam chưa quá deadline + grace period

2. Answer validation:
   - question_id tồn tại trong exam content
   - answer format hợp lệ (choice question: A/B/C/D, writing: text)
   - sequence > 0, monotonic (không nhảy ngược)
   - sync_id là valid UUID

3. Timing validation:
   - answered_at reasonable: >= exam_start, <= now + clock_drift_tolerance
   - Nếu answered_at > deadline → reject (trả lời sau khi hết giờ)

4. Rate limiting:
   - Max 100 answers per sync request (chống abuse)
   - Max 10 sync requests per phút per session (chống spam)

5. Anomaly detection:
   - Gửi answer cho câu đã submit rất nhiều lần (> 20 changes per question → flag)
   - Tất cả answers gửi cùng answered_at → bulk paste → flag cho anti-cheat
```

---

## 7. Thử thách 7: Network Detection — Biết lúc nào online/offline

### Bài toán

```
navigator.onLine:
  ✗ Chỉ biết có kết nối network adapter, KHÔNG biết có internet
  ✗ WiFi connected nhưng no internet → navigator.onLine = true
  ✗ VPN connected nhưng server unreachable → navigator.onLine = true

→ Sync engine nghĩ online → gửi request → timeout → retry → timeout...
→ Lãng phí resources + confusing UX
```

### Giải pháp: Heartbeat + navigator.onLine combined

```
Layer 1: navigator.onLine (instant detection)
  → offline event: chắc chắn offline → pause sync ngay
  → online event: CÓ THỂ online → start heartbeat verify

Layer 2: Heartbeat (accurate detection)
  → GET /api/v1/ping (lightweight, < 1KB response)
  → Mỗi 15 giây khi navigator.onLine = true
  → Response OK → confirmed online → sync enabled
  → Timeout/error → effectively offline → pause sync

Combined:
  navigator.onLine = false → OFFLINE (chắc chắn)
  navigator.onLine = true + heartbeat OK → ONLINE (confirmed)
  navigator.onLine = true + heartbeat fail → EFFECTIVELY OFFLINE
```

### UI Indicator

```
┌──────────────────────────────────────┐
│  ONLINE  ── green dot ── sync OK     │
│  OFFLINE ── red dot ── "Đang offline │
│             Bài thi vẫn được lưu"    │
│  SYNCING ── yellow dot ── "Đang      │
│             đồng bộ..."              │
└──────────────────────────────────────┘

Tại sao hiện indicator?
  - Thí sinh biết trạng thái → không hoang mang
  - "Đang offline — bài thi vẫn được lưu" → yên tâm làm bài
  - Ngược lại: không hiện gì → mất mạng → thí sinh lo lắng
```

---

## 8. Thử thách 8: Data Recovery — Khôi phục khi mất data

### Recovery Matrix

```
┌────────────────────┬──────────────────┬───────────────────────────┐
│   Mất gì?          │  Server có data? │  Recovery                 │
├────────────────────┼──────────────────┼───────────────────────────┤
│ IndexedDB answers  │ Có (đã sync)     │ GET /my-answers → restore │
│ IndexedDB answers  │ Không (chưa sync)│ ✗ MẤT — lý do sync 10s   │
│ IndexedDB content  │ Server có đề     │ Re-download exam content  │
│ IndexedDB state    │ Server có session│ Recalculate từ session    │
│ Exam token (JWT)   │ Server có session│ Refresh token             │
│ Service Worker     │ —                │ Re-register + re-cache    │
│ Server DB          │ —                │ DB backup + client data   │
└────────────────────┴──────────────────┴───────────────────────────┘
```

### Worst case: Server DB crash

```
Server DB crash giữa kỳ thi:
  → Client vẫn hoạt động (offline-first!)
  → Thí sinh vẫn làm bài bình thường
  → Sync fail → sync_queue tích lũy
  → DB recover → client flush queue → data sync thành công

Đây chính là sức mạnh của offline-first:
  Server down 30 phút → 5,000 thí sinh vẫn thi bình thường
  Không ai biết server có vấn đề
  DB recover → sync lại → không mất gì

So sánh server-first:
  Server down 30 giây → 5,000 thí sinh UI đứng
  30 phút → session timeout → mất bài
  → Gọi điện từ 50 trường: "Hệ thống lại sập!"
```

---

## 9. Thử thách 9: Concurrent Exams — 5,000 thí sinh cùng sync

### Bài toán

```
5,000 thí sinh × sync mỗi 10 giây = 500 sync requests/giây
Mỗi sync có 5-10 answers = 2,500-5,000 INSERT/giây

Peak: đầu giờ thi (tất cả bắt đầu cùng lúc)
  - 5,000 GET /exam/content cùng lúc → CDN/cache
  - 5,000 audio download cùng lúc → bandwidth spike

Peak: cuối giờ (tất cả submit cùng lúc)
  - 5,000 POST /submit trong 1-2 phút → spike
```

### Giải pháp: Staggering + Batching + CDN

```
1. Sync staggering (jitter):
   Không sync đúng mỗi 10 giây cho TẤT CẢ client:
   → Sync interval = 10s + random(0-3s)
   → Spread requests ra → giảm spike

2. Exam content qua CDN:
   → Đề thi là static sau khi generate
   → Cache tại CDN edge → 5,000 GET requests không đụng server
   → Audio files qua CDN → giảm bandwidth tại origin

3. Server batch processing:
   → Sync API nhận batch answers → 1 transaction insert nhiều rows
   → Hiệu quả hơn 5,000 single-row INSERT

4. Submit window staggering:
   → Cuối giờ: server auto-submit chạy batch mỗi 30 giây
   → Không phải xử lý 5,000 submit cùng lúc
   → Client submit retry nếu server busy (429 + retry-after)
```

### Back-of-envelope calculation

```
5,000 thí sinh, sync mỗi 10 giây:
  Ingestion: 500 req/s × 10 answers/req = 5,000 rows/s
  PostgreSQL batch INSERT: dễ dàng handle 50,000+ rows/s
  → Bottleneck không ở DB

  API server: 500 req/s × ~10ms/req = 5 concurrent connections
  → 1 instance đủ, 2 instances cho HA

  CDN: 5,000 exam content requests → served from edge
  → Origin server không bị load

  Audio download: 5,000 × 50MB = 250 GB bandwidth
  → CDN handle, spread across exam start window (5-10 phút)
  → ~500 MB/s sustained → CDN tier cần đủ lớn
```

---

## 10. Thử thách 10: Versioning — Cập nhật client khi đang thi

### Bài toán

```
Deploy bản client mới khi 5,000 thí sinh đang thi:
  - Service Worker update → client reload → mất exam state?
  - Nếu không update → 2 versions client chạy cùng lúc
  - New client format != old client format → sync fail?
```

### Giải pháp: Service Worker Lifecycle Control

```
Khi phát hiện new Service Worker available:
  1. Check: exam đang IN_PROGRESS?
     Có → KHÔNG activate new SW → chờ exam kết thúc
     Không → activate new SW → reload

  Implement:
    - SW install → wait (không skipWaiting)
    - Exam kết thúc → postMessage("EXAM_COMPLETE") → SW activate
    - Hoặc: next page load sau exam → new SW tự activate

Không bao giờ reload app khi thí sinh đang thi:
  → Mất focus = mất đáp án đang nhập (writing)
  → Reload = re-render = confusing UX
  → Timer visual jump = thí sinh hoang mang
```

### API versioning

```
Sync API có version: /api/v1/exams/{id}/sync
  - New client dùng /api/v2 (if needed)
  - Old client vẫn gửi /api/v1 → server handle cả 2
  - Backward compatible: v2 accepts v1 format
  - Breaking change → deploy server trước, v1 vẫn hoạt động
    → Exam kết thúc → client update → dùng v2
```

---

## 11. Interview Quick Reference

### Elevator Pitch (challenges)

> "Thử thách lớn nhất của offline-first exam không phải sync — mà là **không bao giờ mất bài thi**. Write-through IndexedDB đảm bảo crash recovery. Sync mỗi 10 giây đảm bảo server có gần đủ answers. Grace period 5 phút + server auto-submit là safety net. Kết quả: server down 30 phút → 5,000 thí sinh vẫn thi bình thường, không ai biết."

### "Browser crash thì sao?"

> "Write-through IndexedDB: mỗi answer change persist ngay (1-5ms, async, không block UI). Crash → reopen → read IndexedDB → restore đến đúng câu hỏi đang làm. Exam state (current section, question, timer deadline) lưu riêng, update mỗi 30 giây."

### "IndexedDB bị xóa?"

> "3 lớp: (1) navigator.storage.persist() — yêu cầu browser không tự xóa. (2) Server có answers đã sync — query server restore. (3) Detect incognito mode → warning. Worst case: mất sync_queue chưa sync = mất 10 giây cuối. Đây là lý do sync thường xuyên."

### "5,000 thí sinh cùng sync?"

> "500 req/s. PostgreSQL batch INSERT dễ handle 50K+ rows/s. Exam content qua CDN. Sync interval + jitter spread requests. 1-2 API instances đủ. Bottleneck thật sự là audio download bandwidth → CDN tier."

### "Server down khi đang thi?"

> "Đây chính là lý do chọn offline-first. Server down 30 phút → client vẫn hoạt động. Sync queue tích lũy. Server recover → flush queue. Không ai biết server có vấn đề. So sánh: server-first down 30 giây → 5,000 thí sinh UI đứng."

### Bảng quyết định tổng hợp

| Quyết định | Chọn | Tại sao | Nếu bị challenge |
|-----------|------|---------|------------------|
| IndexedDB write | Write-through (mỗi change) | Crash chỉ mất 1 thao tác cuối | "Chậm không?" → 1-5ms async, user không cảm nhận |
| Multi-tab | Single-tab lock (BroadcastChannel) | Tránh concurrent write conflict | "Nếu không support?" → IndexedDB heartbeat fallback |
| Network detection | navigator.onLine + heartbeat | onLine alone unreliable (WiFi no internet) | "Heartbeat overhead?" → GET /ping < 1KB, 15 giây/lần |
| Submit offline | Intent + auto-submit + grace | Không mất bài dù offline | "Grace 5 phút đủ không?" → Đủ cho 99%, server auto-submit là safety net |
| Exam content integrity | Checksum + no answers in content | Tamper detected, no answer leak | "Signing?" → Optional high-security, checksum đủ cho thi thử |
| Service Worker update | Block update during exam | Reload = mất focus + confusing | "Hotfix cần thiết?" → Wait until exam ends |
| Concurrent sync | Jitter + CDN + batch | Spread load, CDN for content | "Scale 50K?" → CDN tier lớn hơn + horizontal scale API |
