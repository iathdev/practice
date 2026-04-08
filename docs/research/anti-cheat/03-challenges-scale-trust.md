# Anti-Cheat Service — Khó khăn, Thử thách & Niềm tin B2B

> Thiết kế anti-cheat cho 5,000–10,000 thí sinh đồng thời không chỉ là viết detection rules. File này đi sâu vào những thử thách thực tế: scale, reliability, false positive, buffer overflow, Redis down, và đặc biệt — niềm tin của đối tác B2B.

---

## Mục lục

1. [Thử thách 1: Scale — 10,000 thí sinh đồng thời](#1-thử-thách-1-scale--10000-thí-sinh-đồng-thời)
2. [Thử thách 2: Buffer Overflow — Redis đầy thì sao?](#2-thử-thách-2-buffer-overflow--redis-đầy-thì-sao)
3. [Thử thách 3: Redis Down — Mất buffer, mất real-time check](#3-thử-thách-3-redis-down--mất-buffer-mất-real-time-check)
4. [Thử thách 4: False Positive — Void nhầm thì sao?](#4-thử-thách-4-false-positive--void-nhầm-thì-sao)
5. [Thử thách 5: Flush Fail — DB down giữa chừng](#5-thử-thách-5-flush-fail--db-down-giữa-chừng)
6. [Thử thách 6: Event Order — Events đến không đúng thứ tự](#6-thử-thách-6-event-order--events-đến-không-đúng-thứ-tự)
7. [Thử thách 7: Anti-Anti-Cheat — Thí sinh bypass detection](#7-thử-thách-7-anti-anti-cheat--thí-sinh-bypass-detection)
8. [Thử thách 8: Niềm tin B2B — Tài sản quan trọng nhất](#8-thử-thách-8-niềm-tin-b2b--tài-sản-quan-trọng-nhất)
9. [Thử thách 9: Graceful Shutdown — Không mất event khi deploy](#9-thử-thách-9-graceful-shutdown--không-mất-event-khi-deploy)
10. [Thử thách 10: Monitoring & Observability](#10-thử-thách-10-monitoring--observability)
11. [Interview Quick Reference](#11-interview-quick-reference)

---

## 1. Thử thách 1: Scale — 10,000 thí sinh đồng thời

### Bài toán

```
10,000 thí sinh × ~2 events/giây = 20,000 events/giây (peak)
Mỗi event ~500 bytes JSON
→ ~10 MB/giây ingestion
→ ~1.2 triệu events mỗi 60 giây flush
→ ~600 MB data per flush batch
```

### Bottleneck ở đâu?

```
                   Throughput capacity
                   ──────────────────
Redis LPUSH:       100,000+ ops/s     ✅ dư sức (cần 20K)
Redis real-time:   50,000+ ops/s      ✅ dư sức (Lua script)
DB COPY batch:     100,000+ rows/s    ✅ dư sức (1.2M rows/60s = 20K/s)
Rule engine query: ???                ⚠️ BOTTLENECK tiềm năng

Rule engine chạy query trên 1.2M events vừa flush
  + events trước đó (accumulate trong bài thi)
  → Cross-exam similarity: O(n²) cho n thí sinh
  → Continuous aggregate giúp, nhưng vẫn cần attention
```

### Buffer Decouple — Tại sao cần Redis buffer?

**Bài toán gốc**: Nếu ghi thẳng DB mỗi event:

```
Event → DB INSERT → DB INSERT → DB INSERT → ...
         ↑
   10,000 thí sinh × hàng chục event/phút = hàng trăm nghìn writes/phút
   → DB quá tải, latency tăng, có thể down
   → DB down = MẤT EVENT
```

**Giải pháp: Redis làm buffer tách ingestion khỏi DB write (decouple)**

```
                    Decouple tại đây
                         ↓
Event ──▶ Redis Buffer ──┃──▶ Consumer ──▶ DB
  (write cực nhanh)      ┃    (batch read, ghi DB theo tốc độ DB chịu được)
  100K+ ops/s            ┃
                         ┃
              Ingestion  ┃  Processing
              (nhận vào) ┃  (xử lý + lưu)
```

**2 phần hoạt động độc lập:**

| Phần | Tốc độ | Phụ thuộc |
|------|--------|-----------|
| **Ingestion** (event → Redis) | Cực nhanh, chịu burst | Chỉ cần Redis sống |
| **Processing** (Redis → DB) | Theo tốc độ DB | Cần DB sống |

**Tại sao DB down ≠ mất event?**

```
Không có buffer:
  Event → DB (down) → MẤT ✗

Có Redis buffer:
  Event → Redis ✓ (vẫn nhận được, DB down kệ)
         ↓
  Redis giữ event trong memory/list
         ↓
  DB recover → Consumer đọc Redis → ghi DB ✓
```

Redis đóng vai trò **"bể chứa tạm"**. Event đến bao nhiêu cứ đổ vào Redis trước. Consumer phía sau đọc ra và ghi DB **theo tốc độ DB chịu được** (back-pressure). DB down thì consumer dừng, nhưng event **không mất** vì vẫn nằm trong Redis.

**Redis down thì sao?** → Xem [Thử thách 3: Redis Down](#3-thử-thách-3-redis-down--mất-buffer-mất-real-time-check) — Circuit breaker chuyển sang ghi thẳng DB làm fallback.

**Keywords**: Decouple, Buffer, Back-pressure, Batch write, At-least-once delivery.

### Giải pháp

| Bottleneck | Giải pháp | Khi nào cần |
|-----------|----------|-------------|
| Ingestion (Redis) | Đã đủ với single Redis | Luôn đủ cho 10K |
| DB batch insert | COPY protocol đã tối ưu | Luôn đủ cho 10K |
| Rule engine queries | Continuous aggregates → query materialized view | Cần từ 5K+ |
| Cross-exam similarity | Chỉ chạy cuối bài + shard theo exam_id | Cần từ 1K+ users/exam |
| Buffer memory | Set max buffer size + backpressure | Cần khi peak > expected |

### Nếu scale lên 50K+ thí sinh?

```
Theo thứ tự, chỉ thêm khi cần:

1. Shard Redis buffer theo exam_id
   → Mỗi shard chứa events của 1 nhóm exams
   → Flush từng shard song song

2. Multiple Anti-Cheat Service instances
   → Load balance events → mỗi instance handle 1 nhóm exams
   → Stateless (state ở Redis + DB)

3. Read replica PostgreSQL
   → Rule engine query read replica
   → Write path (COPY batch) vẫn primary

4. Nếu vẫn không đủ → chuyển sang Kafka + ClickHouse
   → Native streaming, columnar storage
   → Nhưng ops phức tạp hơn nhiều
```

---

## 2. Thử thách 2: Buffer Overflow — Redis đầy thì sao?

### Bài toán: Tại sao buffer tăng?

```
Bình thường:
  Events vào Redis (LPUSH):  20,000/giây     ← thí sinh thi, không dừng được
  Events ra DB (flush):      20,000/giây     ← COPY batch mỗi 60s
  → Vào = Ra → buffer ổn định (~1.2M events, ~600MB)

DB slow (query nặng, lock, disk IO cao):
  Events vào Redis:  20,000/giây     ← KHÔNG ĐỔI, thí sinh vẫn thi
  Events ra DB:      5,000/giây      ← CHẬM LẠI
  → Vào > Ra → buffer TĂNG DẦN

  Giống bồn rửa: nước chảy vào nhanh hơn thoát ra → nước dâng

DB down hoàn toàn:
  Events vào Redis:  20,000/giây     ← vẫn không dừng
  Events ra DB:      0/giây          ← không thoát được
  → 5 phút → buffer tích 6M events → ~3GB
  → Redis default maxmemory 1-2GB → TRÀN → mất events
```

### Giải pháp: Backpressure + Multi-layer Defense

**Backpressure là gì?** Khi hệ thống phía sau (DB) chậm, áp lực đẩy ngược lên phía trước để giảm tốc, thay vì để hệ thống sập vì quá tải:

```
DB slow → flush chậm → buffer đầy → reject event → SDK gửi chậm lại
          ↑                                          ↓
          └────── áp lực đẩy ngược từ DB ← SDK ──────┘
                  đây là "back-pressure"
```

**Multi-layer defense là gì?** Không dựa vào 1 cơ chế duy nhất. Xếp nhiều lớp bảo vệ — layer trước fail thì layer sau bắt:

```
Layer 1 fail → Layer 2 bắt
Layer 2 fail → Layer 3 bắt
→ Phải vượt qua CẢ 3 LAYER mới mất event
```

### Layer 1 (code, tự động): Adaptive flush interval — flush nhanh hơn khi buffer đầy

```
Buffer ổn định (< 50% capacity):
  Flush interval = 60 giây            ← bình thường, không vội

Buffer bắt đầu đầy (50-80%):
  Flush interval = 30 giây            ← flush nhanh hơn, giải phóng buffer

Buffer gần tràn (> 80%):
  Flush interval = 10 giây            ← flush gấp, tránh tràn

DB recover, buffer giảm (< 50%):
  Flush interval = 60 giây            ← quay về bình thường

Giống van xả nước: bể sắp đầy → mở van to hơn. Bể vơi → mở nhỏ lại.

Tại sao cần?
  Nếu cố định 60 giây:
    DB slow 2 phút → buffer tích 2.4M events → TRÀN → mất event
  Nếu adaptive:
    DB slow → buffer tăng → interval giảm → flush nhiều batch nhỏ
    → buffer không tràn
```

### Layer 2 (code, tự động): Reject 429 — ngăn buffer tràn khi đã đầy

```
Mỗi lần nhận event:
  1. LLEN "events:buffer"              ← check buffer đang bao nhiêu
  2. Nếu > 100,000 → trả 429 Too Many Requests (KHÔNG LPUSH)
  3. Nếu ≤ 100,000 → LPUSH bình thường

JS SDK nhận 429 → không vứt event đi → giữ trong local buffer (array trong browser):

  const pendingEvents = [];  // "local buffer" = array trong browser memory

  function sendEvent(event) {
      fetch('/api/v1/events', { body: JSON.stringify(event) })
          .then(res => {
              if (res.status === 429) {
                  pendingEvents.push(event);  // giữ tạm
              }
          });
  }

  // Mỗi 5 giây retry gửi lại
  setInterval(() => {
      if (pendingEvents.length > 0) {
          const batch = pendingEvents.splice(0, 50);
          fetch('/api/v1/events/batch', { body: JSON.stringify(batch) });
      }
  }, 5000);

  Đóng tab = mất events trong local buffer
  → Chấp nhận được vì 429 là tình huống hiếm (buffer overflow)
```

**Tại sao reject 429 chứ không drop silently?**

```
Drop silently:
  ✗ Client không biết → event mất → evidence mất
  ✗ Đối tác hỏi "tại sao không detect gian lận?" → không có data

429 + retry:
  ✓ Client biết → giữ tạm, retry sau
  ✓ Server log reject count → monitoring alert → ops biết ngay
  ✓ Phần lớn events sẽ được gửi lại thành công khi buffer giảm
```

### Layer 3 (con người, manual): Redis memory alarm

```
Redis server có giới hạn memory (maxmemory), ví dụ 2GB.

Monitor liên tục:
  Redis used_memory: 400MB / 2GB (20%)     ← OK, không cần làm gì
  Redis used_memory: 1.4GB / 2GB (70%)     ← ⚠️ ALERT gửi cho ops team

Tại sao alert ở 70% chứ không phải 90%?
  → Peak traffic có thể đẩy từ 70% → 100% trong vài phút
  → Alert sớm → ops có thời gian xử lý TRƯỚC KHI tràn

Ops nhận alert → 2 lựa chọn:
  1. Tăng maxmemory (nếu server còn RAM):  CONFIG SET maxmemory 4gb
  2. Scale Redis (nếu hết RAM):            thêm Redis instance

Nếu KHÔNG có layer này:
  Redis đạt maxmemory → reject writes → mất events
  → Không ai biết → đến khi user report → đã muộn
```

### 3 layers hoạt động cùng nhau

```
Tình huống: DB slow → buffer tích nhanh

Layer 1 (tự động):  Flush interval giảm 60s → 30s → 10s
                    → cố gắng giải phóng buffer nhanh hơn

Vẫn không đủ? Buffer > 100K:
Layer 2 (tự động):  Reject 429 → JS SDK giữ tạm → retry sau
                    → ngăn buffer tràn thêm

Đồng thời:
Layer 3 (manual):   Alert ops → tăng Redis memory hoặc fix root cause (DB)
```

| Layer | Tự động? | Xử lý gì | Khi nào kick in |
|-------|---------|----------|----------------|
| 1. Adaptive flush | Tự động | Flush nhanh hơn khi buffer đang đầy dần | Buffer > 50% |
| 2. Reject 429 | Tự động | Ngăn buffer tràn, SDK giữ tạm retry sau | Buffer > 100K |
| 3. Memory alarm | Manual (ops) | Tăng RAM hoặc fix root cause | Redis memory > 70% |

---

## 3. Thử thách 3: Redis Down — Mất buffer, mất real-time check

### Bài toán

```
Redis down → 2 thứ mất cùng lúc:
  1. Event buffer (LPUSH fail)
  2. Real-time counter (INCR fail)
  → Không buffer được events → mất data
  → Không check được tab_switch count → bỏ sót gian lận
```

### Giải pháp: Circuit Breaker + Graceful Degradation

```
Redis available (normal mode):
  ┌──────────────────────────────────────────────┐
  │ Event → Real-time check (Redis) → Buffer (Redis) → DB │
  └──────────────────────────────────────────────┘

Redis down (degraded mode):
  ┌──────────────────────────────────────────────┐
  │ Event → Skip real-time check → INSERT trực tiếp DB    │
  │         (log warning)          (chậm hơn nhưng không   │
  │                                 mất data)              │
  └──────────────────────────────────────────────┘
```

**Chi tiết degraded mode:**

| Thành phần | Normal | Degraded |
|-----------|--------|----------|
| Event ingestion | LPUSH Redis (0.1ms) | INSERT DB trực tiếp (5-50ms) |
| Real-time check | Redis INCR + Lua (< 1ms) | Skip (log warning) |
| Batch analysis | Từ TimescaleDB (không đổi) | Từ TimescaleDB (không đổi) |
| Latency cho client | < 1ms | 5-50ms |
| Detection coverage | Lớp 1 + Lớp 2 | Chỉ Lớp 2 |

**Trade-off chấp nhận được:**
- Mất Lớp 1 real-time check → devtools_open, automation_signal bắt trễ hơn (60s thay vì < 1s)
- Client chậm hơn (5-50ms thay vì 0.1ms) → nhưng exam vẫn hoạt động
- Data không mất → Lớp 2 batch analysis vẫn bắt pattern

**Recovery:**
- Redis recover → circuit breaker close → resume normal mode
- Events đã INSERT trực tiếp DB → Lớp 2 phân tích bình thường
- Real-time counters reset → start fresh (chấp nhận, window ngắn)

---

## 4. Thử thách 4: False Positive — Void nhầm thì sao?

### Bài toán

Đây là **thử thách nguy hiểm nhất** trong hệ thống B2B:

```
False positive: void exam oan cho thí sinh hợp lệ
  → Thí sinh khiếu nại
  → Đối tác (trường học) mất niềm tin
  → Đối tác nói: "hệ thống của bạn không đáng tin"
  → Hủy hợp đồng B2B
  → Mất cả batch khách hàng (trường + tất cả học sinh)

False negative: bỏ sót gian lận
  → Kết quả không phản ánh năng lực thật
  → Đối tác phát hiện → cũng mất niềm tin
  → Nhưng ít gây phản ứng mạnh hơn void nhầm

→ Trong B2B: false positive NGUY HIỂM hơn false negative
```

### Giải pháp: Graduated Response + Human-in-the-Loop

**Nguyên tắc: Auto-void chỉ cho hành vi chắc chắn 100%. Còn lại đều cần admin review.**

```
auto-void (không cần review):
  ✓ automation_signal (WebDriver/headless → chắc chắn bot)
  ✓ paste_ratio > 90% trong writing (97% paste = không thể là typing)

auto-warn (cảnh báo, không void):
  ✓ tab_switch > 5 lần (có thể do notification OS)
  ✓ focus_lost > 10 lần (có thể do multitask app)

flag + admin review:
  ✓ tab-then-correct pattern (cần human judgment)
  ✓ answer_speed anomaly (có thể thí sinh giỏi thật)
  ✓ cross-exam similarity (cần verify thêm context)
```

### Evidence Package cho Admin

Khi tạo detection severity=high, kèm evidence đầy đủ:

```json
{
  "rule_id": "batch_tab_then_correct",
  "severity": "high",
  "evidence": {
    "tab_then_answer_count": 8,
    "correct_after_tab": 8,
    "correct_rate_after_tab": 1.0,
    "normal_correct_rate": 0.42,
    "timeline": [
      {"time": "14:12:31", "action": "tab_switch", "question": "reading_q12"},
      {"time": "14:12:45", "action": "answer_submit", "question": "reading_q12", "correct": true},
      ...
    ]
  }
}
```

Admin nhìn evidence → quyết định: void, giảm severity, hoặc dismiss.

### Appeal Process (cho đối tác B2B)

```
Thí sinh bị flag → Admin review → Void
  → Thí sinh/đối tác khiếu nại
  → Xuất evidence (audit trail từ cheat_detections + exam_events)
  → Review lại: nếu sai → revert, xin lỗi, adjust rule threshold
  → Mỗi appeal = feedback loop để cải thiện rules
```

---

## 5. Thử thách 5: Flush Fail — DB down giữa chừng

### Bài toán

```
Buffer có 1.2M events → bắt đầu flush
  → COPY batch: 800K events inserted OK
  → DB connection drop giữa chừng
  → 400K events chưa insert → mất?
```

### Giải pháp: Reliable Queue Pattern

```
Không dùng LRANGE + LTRIM (unsafe):
  1. LRANGE buffer → đọc events
  2. INSERT DB → crash ở đây
  3. LTRIM buffer → KHÔNG CHẠY → events còn trong buffer → OK cho case này
  NHƯNG nếu INSERT OK, LTRIM fail → events insert 2 lần → duplicate

Dùng RPOPLPUSH (reliable queue):
  1. RPOPLPUSH từ "events:buffer" → "events:processing"
     (atomic move — event chỉ ở 1 trong 2 lists)
  2. INSERT DB từ "events:processing"
  3. Thành công → DEL "events:processing"
  4. Thất bại → events ở lại "events:processing" → retry lần sau

  Crash ở bước 2 → events ở "events:processing" → không mất
  Crash ở bước 3 → events đã INSERT DB + vẫn ở processing
    → Flush lần sau: check processing list trước
    → INSERT lại (idempotent: event có unique ID, DB reject duplicate)
```

### Tại sao cần idempotent insert?

```
Crash giữa INSERT và DEL processing list
  → Events đã insert DB nhưng processing list chưa xóa
  → Flush lần sau: đọc processing list → INSERT lại
  → Nếu không idempotent → duplicate events

Giải pháp:
  - exam_events có PRIMARY KEY (id, created_at)
  - ON CONFLICT DO NOTHING
  - Hoặc: check processing list, nếu có events → query DB xem đã insert chưa
```

---

## 6. Thử thách 6: Event Order — Events đến không đúng thứ tự

### Bài toán

```
Thí sinh thao tác:
  14:12:31 - tab_switch
  14:12:43 - focus_gained
  14:12:45 - answer_submit

Nhưng do network:
  focus_gained đến trước (14:12:43 → arrived 14:12:44)
  tab_switch đến sau (14:12:31 → arrived 14:12:45, network delay)

→ Real-time counter nhận focus_gained trước tab_switch
→ Count sai? Pattern analysis sai?
```

### Giải pháp

```
Real-time (Lớp 1): KHÔNG quan tâm order
  - Counter chỉ đếm: tab_switch count += 1
  - Không quan tâm "focus_gained đến trước hay sau"
  - Đơn giản, robust

Batch (Lớp 2): Dùng event timestamp, KHÔNG dùng arrival time
  - JS SDK gắn timestamp chính xác (Performance.now())
  - SQL ORDER BY timestamp (event time), không phải created_at (arrival time)
  - Tab-then-correct query: dùng event.timestamp cho time window
  - created_at chỉ dùng cho hypertable partitioning

  Vẫn có risk: client clock sai hoặc bị tamper
  → Mitigation: server gắn thêm received_at, so sánh drift
  → Drift > 5 giây → flag event, nhưng vẫn process
```

---

## 7. Thử thách 7: Anti-Anti-Cheat — Thí sinh bypass detection

### Các cách bypass và đối phó

| Bypass | Cách thí sinh làm | Đối phó |
|--------|-------------------|---------|
| **Disable JS SDK** | Dev console: override event sender | SDK integrity check (hash). Không nhận event = suspicious (0 events cho session) |
| **Screen sharing** | Share màn hình qua Discord/Zoom | Detect screen capture API (experimental). Hoặc: proctoring camera |
| **2 device** | Laptop thi + phone tra cứu | Không detect được bằng browser event. Cần: proctoring camera |
| **Slow paste** | Paste 10 chars mỗi lần (dưới threshold) | Batch analysis: tổng paste > total typed → vẫn bắt paste_ratio |
| **Random tab switch** | Chuyển tab ngẫu nhiên để mask pattern | Tab-then-correct: chỉ flag khi tab switch + correct answer. Random tab switch → correct rate bình thường → không flag |
| **VPN/Proxy** | Thay đổi IP để tránh IP tracking | Device fingerprint vẫn detect. IP change giữa session → flag |

### Giới hạn của browser-based detection

```
Có thể detect:
  ✓ Tab switch, focus lost, paste, devtools, automation
  ✓ Typing pattern, answer pattern, timing
  ✓ Device/IP/screen changes

KHÔNG thể detect (chỉ bằng browser):
  ✗ Điện thoại để bên cạnh tra cứu
  ✗ Người khác đọc đáp án cho
  ✗ Screen sharing qua hardware (HDMI splitter)
  ✗ Virtual machine + normal browser inside

→ Muốn chống triệt để hơn → cần:
  - Camera proctoring (AI detect người lạ, nhìn sang)
  - Lockdown browser (chặn mọi app khác)
  - Cả 2 đều ngoài scope browser-based anti-cheat
```

### Chiến lược: Defense in Depth (không phải silver bullet)

```
Browser-based anti-cheat = Lớp 1 (làm được ngay, chi phí thấp)
  → Bắt 70-80% gian lận phổ biến
  → Đủ cho "luyện thi" và "thi thử"

Camera proctoring = Lớp 2 (cần thêm infra + AI)
  → Bắt thêm 15-20% gian lận
  → Cần cho "thi chính thức" có chứng chỉ

Lockdown browser = Lớp 3 (cần install phần mềm)
  → Chặn mọi thao tác ngoài exam
  → UX tệ, cần install → tỷ lệ conversion giảm
```

---

## 8. Thử thách 8: Niềm tin B2B — Tài sản quan trọng nhất

### Tại sao B2B khác B2C?

```
B2C (bán cho end user):
  1 user gian lận → ảnh hưởng 1 user
  Void nhầm → 1 user khiếu nại → xử lý cá nhân

B2B (bán cho tổ chức):
  1 user gian lận → đối tác nghi ngờ TOÀN BỘ hệ thống
  Void nhầm → đối tác khiếu nại → ảnh hưởng HỢP ĐỒNG
  → Mất 1 đối tác = mất hàng trăm/ngàn users

Hệ quả:
  B2C: optimize cho detection rate (bắt nhiều nhất có thể)
  B2B: optimize cho TRUST (bắt đúng, không bắt nhầm, transparent)
```

### 5 trụ cột xây dựng niềm tin

#### Trụ 1: Transparency — Đối tác nhìn thấy mọi thứ

```
Admin Dashboard cho đối tác:
  - Xem real-time: bao nhiêu thí sinh đang thi, bao nhiêu warning
  - Xem chi tiết: mỗi detection kèm evidence (timeline, data)
  - Xuất report: PDF báo cáo sau kỳ thi (tổng hợp detection stats)

Tại sao?
  Đối tác nhìn thấy hệ thống hoạt động → tin tưởng
  "Black box" detect → đối tác nghi ngờ → không tin
```

#### Trụ 2: Configurable — Đối tác tùy chỉnh được

```
Mỗi đối tác có thể điều chỉnh threshold:
  - tab_switch warning: 5 lần (mặc định) → 10 lần (nới lỏng)
  - paste flag: 100 chars (mặc định) → 50 chars (nghiêm khắc hơn)
  - auto-void: bật/tắt (mặc định tắt, đối tác quyết định)

Tại sao?
  Trung tâm luyện thi → nới lỏng (mục đích học tập)
  Tổ chức thi chính thức → nghiêm khắc (chứng chỉ có giá trị)
  Đối tác quyết định = đối tác chịu trách nhiệm = trust
```

#### Trụ 3: Audit Trail — Mọi thứ có bằng chứng

```
Audit trail = bằng chứng pháp lý:
  exam_events:      mọi hành vi thí sinh (giữ 1 năm)
  cheat_detections: mọi detection + evidence (giữ vĩnh viễn)
  admin_actions:    ai review, quyết định gì, khi nào

Tại sao 1 năm?
  - Đối tác cần report cuối kỳ, cuối năm
  - Tranh chấp/khiếu nại có thể đến sau vài tháng
  - Compliance: chứng minh hệ thống hoạt động đúng

TimescaleDB compression + retention → giữ 1 năm mà chi phí storage thấp
```

#### Trụ 4: SLA — Cam kết chất lượng

```
SLA cho đối tác:
  - Anti-cheat availability: 99.9% (downtime < 8.7 giờ/năm)
  - Event processing latency: < 5ms (P99)
  - Real-time detection: < 1 giây
  - Batch detection: < 5 phút sau khi thi xong
  - Data retention: 365 ngày

Nếu anti-cheat down → exam vẫn chạy (degraded mode)
→ Không bao giờ block thí sinh thi vì anti-cheat có vấn đề
→ Thà mất detection tạm thời còn hơn thí sinh không thi được
```

#### Trụ 5: Feedback Loop — Cải thiện liên tục

```
Mỗi detection được admin review:
  - Confirm → rule đúng → maintain threshold
  - Dismiss → false positive → xem xét adjust threshold
  - Escalate → cần thêm evidence → xem xét thêm rule mới

Track metrics:
  - Precision: % detection đúng / tổng detection
  - Recall: % gian lận bắt được / tổng gian lận (ước tính)
  - False positive rate per rule → tune threshold

Report cho đối tác mỗi tháng:
  - Tổng bài thi, tổng detection, false positive rate
  - So sánh tháng trước → tháng này → cải thiện
```

---

## 9. Thử thách 9: Graceful Shutdown — Không mất event khi deploy

### Bài toán

```
Deploy phiên bản mới:
  1. Service đang chạy, buffer có 500K events
  2. SIGTERM → service shutdown
  3. Nếu shutdown ngay → 500K events trong Redis buffer chưa flush → mất?
```

### Giải pháp

```
Khi nhận SIGTERM:
  1. Stop accepting new HTTP requests (server.Shutdown)
  2. Cancel context → buffer flusher nhận signal
  3. Flusher: thực hiện FINAL FLUSH
     → Move events từ buffer → processing list
     → COPY batch vào TimescaleDB
     → Delete processing list
  4. Wait tối đa 30 giây cho final flush
  5. Close Redis connection
  6. Close DB connection pool
  7. Exit

Events an toàn vì:
  - Đang ở Redis buffer → final flush xử lý
  - Đang ở processing list → flush tiếp hoặc instance mới xử lý
  - Đã ở DB → an toàn
```

### Zero-downtime deploy

```
Rolling deployment:
  Instance A đang serve traffic
  Instance B start → healthy check → bắt đầu nhận traffic
  Instance A nhận SIGTERM → drain + final flush → shutdown
  Instance B xử lý buffer (cả events cũ nếu processing list còn)

Tại sao rolling deploy quan trọng cho anti-cheat?
  - Thí sinh đang thi → events phải được nhận liên tục
  - Gap 5 giây giữa 2 instances = 100K events mất
  - Rolling deploy = zero gap
```

---

## 10. Thử thách 10: Monitoring & Observability

### Key Metrics

| Metric | Mục đích | Alert khi |
|--------|---------|----------|
| `events_received_total` | Throughput ingestion | Drop > 20% vs baseline |
| `buffer_size` | Số events trong Redis buffer | > 80% max size |
| `flush_duration_seconds` | Thời gian mỗi lần flush | > 30 giây |
| `flush_events_count` | Số events per flush | Spike bất thường |
| `detection_count` by severity | Số detection per severity level | Critical > 0 → immediate alert |
| `false_positive_rate` by rule | % detection bị dismiss | > 20% → review rule |
| `realtime_check_latency` | Latency Lớp 1 check | P99 > 5ms |
| `redis_memory_used` | Redis memory | > 70% maxmemory |
| `db_connection_pool_used` | DB connections đang dùng | > 80% max pool |

### Health Check

```
GET /api/v1/health

Response:
{
  "status": "ok",           // hoặc "degraded" nếu Redis down
  "redis": "connected",     // hoặc "disconnected"
  "db": "connected",
  "buffer_size": 45230,
  "last_flush_at": "2026-04-06T14:30:00Z",
  "uptime_seconds": 86400
}
```

### Dashboard cho Ops

```
Real-time:
  - Events/second (current vs baseline)
  - Buffer fill rate (% capacity)
  - Active exams count
  - Detection count by severity (last 1 hour)

Historical:
  - Events per day (trend)
  - Detection rate by rule (which rules fire most?)
  - False positive rate (improving over time?)
  - Flush performance (degrading?)
```

---

## 11. Interview Quick Reference

### Elevator Pitch (thử thách)

> "Thử thách lớn nhất không phải scale hay detection — mà là **niềm tin B2B**. Void nhầm 1 thí sinh = đối tác nghi ngờ toàn bộ hệ thống. Nên thiết kế graduated severity: chỉ auto-void cho automation và paste >90%, còn lại cần admin review. Kèm theo: audit trail 1 năm, admin dashboard cho đối tác, configurable thresholds, và feedback loop cải thiện liên tục."

### "Scale 10,000 thí sinh đồng thời thế nào?"

> "20K events/s. Redis LPUSH handle 100K+/s — bottleneck không ở ingestion. COPY batch protocol 100K+ rows/s — bottleneck không ở DB write. Bottleneck tiềm năng: rule engine query → giải quyết bằng continuous aggregates (materialized view tính sẵn mỗi phút). Cross-exam similarity O(n²) → chỉ chạy cuối bài."

### "Redis down thì sao?"

> "Circuit breaker → degraded mode: skip real-time check (Lớp 1), INSERT trực tiếp DB (chậm hơn 5-50ms nhưng không mất data). Lớp 2 batch analysis vẫn chạy từ DB. Exam không bao giờ bị block bởi anti-cheat down — SLA cam kết với đối tác."

### "Buffer overflow?"

> "Backpressure: LLEN check trước LPUSH, reject 429 nếu buffer > max. Adaptive flush interval: buffer đầy → flush nhanh hơn. JS SDK nhận 429 → local buffer + retry. Monitoring alert khi buffer > 70%."

### "False positive — void nhầm?"

> "Auto-void chỉ cho automation_signal và paste_ratio >90% (chắc chắn gian lận). Tab-then-correct, answer speed, cross-exam similarity → severity=high nhưng cần admin review. Trong B2B, false positive nguy hiểm hơn false negative: mất 1 đối tác = mất hàng trăm users."

### Bảng quyết định tổng hợp

| Quyết định | Chọn | Tại sao | Nếu bị challenge |
|-----------|------|---------|------------------|
| Redis buffer | LPUSH + periodic flush | Decouple ingestion khỏi DB, chịu peak | "Sao không Kafka?" → Overkill cho 10K, Redis đủ, ops đơn giản |
| TimescaleDB | PostgreSQL extension | Team quen, time-series native, continuous aggregate | "Sao không ClickHouse?" → Cần SQL join cho cross-exam, PostgreSQL đủ |
| Reliable queue | RPOPLPUSH pattern | Không mất event khi crash | "Sao không Lua script?" → Lua cho move, RPOPLPUSH cho reliability |
| Graduated severity | 4 levels | False positive ở critical = risk mất đối tác B2B | "Sao không binary?" → Binary → void nhầm → mất trust |
| Admin review | Human-in-the-loop cho severity=high | Giảm false positive, đối tác thấy công bằng | "Sao không auto tất cả?" → Mỗi auto-void sai = risk mất hợp đồng |
| Configurable thresholds | Per-tenant config | Mỗi đối tác requirements khác nhau | "Sao phải phức tạp?" → B2B = customizable hoặc mất deal |
| Audit trail 1 năm | TimescaleDB compression + retention | Compliance, tranh chấp, report | "Sao 1 năm?" → Hợp đồng B2B thường 1 năm, cần data suốt kỳ |
| Degraded mode | Skip Redis, INSERT DB trực tiếp | Anti-cheat down ≠ exam down | "Mất real-time check?" → Chấp nhận, exam phải chạy luôn |
