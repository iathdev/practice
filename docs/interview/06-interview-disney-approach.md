# Interview Cheat Sheet: Hệ thống đặt vé công viên — Disney Approach

> Dùng để trả lời phỏng vấn khi được hỏi "Kể về dự án bạn đã làm" hoặc deep dive technical.

---

## 1. Elevator Pitch (30 giây)

> "Em xây dựng hệ thống đặt vé công viên giải trí, xử lý hàng ngàn booking đồng thời mỗi ngày. Kiến trúc tham khảo theo hướng Disney Parks — dùng **Redis làm availability cache** cho read path (< 1ms), **PostgreSQL với optimistic locking** làm source of truth cho write path, và **Lua script** để đảm bảo atomic operations trên Redis. Hệ thống thiết kế theo nguyên tắc **defense in depth**: Redis filter → DB optimistic lock → CHECK constraint → reconciliation job, đảm bảo **không bao giờ oversell** dù có failure ở bất kỳ layer nào."

---

## 2. Kiến trúc tổng quan (vẽ khi được yêu cầu)

```
Client → Redis (read availability, < 1ms)
       → Booking Service → Redis pre-reserve (Lua script)
                         → DB optimistic lock (PostgreSQL)
                         → Redis confirm/rollback

Background Jobs:
  - Reconciliation (Redis ↔ DB mỗi 30s)
  - Expire reservations (mỗi 15s)
  - Outbox publisher (mỗi 1s)
```

**Nguyên tắc cốt lõi:**
- DB là source of truth, Redis là cache — mất Redis thì rebuild được
- Drift luôn theo hướng conservative — thà reject sớm hơn là oversell
- Graceful degradation — Redis down thì fallback về DB, chậm hơn nhưng vẫn đúng

---

## 3. Database Design

### Inventory Model: Tách 3 trạng thái

```sql
daily_inventory (
    park_id, ticket_type_id, visit_date,
    total_capacity,  -- admin set, hiếm đổi
    reserved,        -- đang giữ tạm (chờ payment)
    booked,          -- đã thanh toán
    version,         -- optimistic locking
    CHECK (reserved + booked <= total_capacity)  -- KHÔNG BAO GIỜ oversell
)
-- available = total_capacity - reserved - booked (computed, không lưu)
```

**Nếu bị hỏi "Sao không dùng 1 field `remaining`?":**
- `remaining` là denormalized, derived value — cancel mà quên cộng lại thì drift
- Tách reserved/booked → biết chính xác bao nhiêu đang hold vs đã confirm
- Hold hết hạn → chỉ trừ `reserved`, không ảnh hưởng `booked` → recoverable
- CHECK constraint chặn oversell ở tầng DB, dù code có bug

### Luồng dữ liệu

```
Reserve:     reserved += N          (available giảm N)
Confirm:     reserved -= N, booked += N  (available không đổi)
Expire:      reserved -= N          (available tăng lại N)
Cancel:      booked -= N            (available tăng lại N)
```

---

## 4. Redis Layer

### Data Structure: Hash (không phải single counter)

```
Key:   availability:{park_id}:{ticket_type_id}:{visit_date}
Value: { total: 500, reserved: 45, booked: 320, available: 135 }
```

**Nếu bị hỏi "Sao dùng Hash, không dùng single key `available=135`?":**
- Hash → biết breakdown chi tiết → debug/monitor dễ → reconcile chính xác
- Single counter → không biết reserved vs booked → khó phát hiện drift ở đâu

### Tại sao dùng Lua Script?

Redis cần **atomic read-check-write** trên nhiều field. Ví dụ pre-reserve:

```
1. HGET available     → check đủ chỗ không
2. HINCRBY reserved   → tăng reserved
3. HINCRBY available  → giảm available
```

Nếu 3 lệnh riêng → 2 thread xen kẽ → race condition.
Lua script chạy **single-threaded, không bị xen ngang**.

**"Sao không dùng MULTI/EXEC?"** — MULTI/EXEC không hỗ trợ đọc giữa chừng rồi quyết định. Không có `if/else`. Lua thì có.

### 4 Lua Scripts

| Script | Khi nào | Làm gì |
|--------|---------|--------|
| Check | User xem availability | HGET available, return đủ/không đủ |
| Pre-reserve | User submit đặt vé | HINCRBY reserved +N, available -N |
| Rollback | DB reject / reservation expire | HINCRBY reserved -N, available +N |
| Confirm | Payment thành công | HINCRBY reserved -N, booked +N |

---

## 5. Luồng đặt vé (Happy Path)

```
1. User check availability → Redis Hash → trả về "còn 135 chỗ" (< 1ms)
2. User submit booking → Lua pre-reserve: available -= 5 trên Redis
3. DB optimistic lock: UPDATE reserved += 5 WHERE version = X
4. Tạo booking (PENDING, expires_at = now + 15 min)
5. User thanh toán → Payment callback
6. DB: reserved -= 5, booked += 5
7. Redis Lua confirm: reserved -= 5, booked += 5
8. Booking status → CONFIRMED
```

---

## 6. Failure Cases — Câu hỏi phỏng vấn hay gặp nhất

### "Redis pre-reserve OK nhưng DB reject thì sao?"

→ **Compensation**: gọi Lua rollback → available += N trên Redis
→ Retry tối đa 3 lần (exponential backoff + jitter)
→ Hết retry → trả 409 Conflict cho client

### "Rollback Redis cũng fail thì sao?"

→ Redis giữ giá trị conservative (available thấp hơn thực tế)
→ **Không oversell** — chỉ mất sale tạm thời
→ Reconciliation job fix trong 30s

**Nguyên tắc: drift LUÔN theo hướng conservative. Thà mất sale 30s còn hơn oversell.**

### "Redis down hoàn toàn thì sao?"

→ Circuit breaker mở → bypass Redis
→ Fallback: query DB trực tiếp + **đổi sang pessimistic lock** tạm thời
→ Tại sao pessimistic? Vì không có Redis filter traffic → optimistic lock sẽ retry storm
→ Redis recover → reconcile full → resume normal flow

### "DB down thì sao?"

→ Không thể đặt vé (DB là source of truth, không compromise)
→ Redis vẫn serve availability reads (user xem được "còn chỗ")
→ POST /bookings trả 503 + circuit breaker reject nhanh

### "2 request cùng lúc, cả 2 pass Redis nhưng chỉ 1 pass DB?"

→ Đúng design. Redis là fast gate (filter phần lớn), DB là final arbiter
→ Request thua → rollback Redis → retry hoặc reject
→ CHECK constraint là safety net cuối cùng

---

## 7. Tại sao Optimistic Lock mà không Pessimistic?

**Câu trả lời ngắn:** Vì Redis đã filter traffic rồi, contention tại DB thấp → optimistic đủ tốt, throughput cao hơn.

**Câu trả lời dài:**

| | Pessimistic | Optimistic (chọn) |
|---|---|---|
| Mechanism | Lock row, thread khác chờ | Update WHERE version = X, fail thì retry |
| Khi conflict | Block (chờ) | Retry (nhanh hơn) |
| DB connection | Giữ suốt transaction | Release sớm hơn |
| Throughput | Thấp (serialize) | Cao hơn |

**3 lý do chọn optimistic:**
1. **Read:Write = 100:1** — 10K check availability (Redis) vs 100 thực sự đặt (DB). Optimistic không block read
2. **Redis đã filter** — chỉ request pass Redis mới xuống DB → contention thấp → retry storm ít xảy ra
3. **Connection pool** — pessimistic giữ connection + row lock suốt transaction → pool cạn nhanh khi nhiều concurrent users

**Khi nào pessimistic tốt hơn?**
- Redis down (fallback mode) → không có filter → cần serialize tại DB
- Business logic phức tạp cần critical section (check nhiều bảng)
- Contention cực cao trên 1 row (flash sale)

---

## 8. Các bài toán đã giải quyết — Quick Reference

### 8.1 Cache Invalidation (Redis drift khỏi DB)

**Vấn đề:** App crash, network timeout, Redis restart → Redis ≠ DB

**Giải pháp:** Write-through + Reconciliation mỗi 30s

**"Sao không cache-aside?"**
- Cache-aside invalidate → giữa delete và rebuild, cache miss → spike DB load
- Write-through → Redis luôn có data, nếu sai thì conservative direction → reconciliation fix

### 8.2 Reservation TTL (Hold hết hạn)

**Vấn đề:** User đặt nhưng không thanh toán → phải release chỗ

**Giải pháp:** Scheduled Job quét DB mỗi 15s (`SELECT ... WHERE status='PENDING' AND expires_at < NOW() FOR UPDATE SKIP LOCKED`)

**"Sao không dùng Redis TTL / Keyspace Notification?"**
- Redis key expire ≠ DB update → nếu notification miss → DB không release → stuck
- Keyspace notification là best-effort, không guarantee delivery
- Scheduled Job: DB là source of truth, idempotent, không thêm infrastructure

**"Sao dùng `SKIP LOCKED`?"**
- Nhiều instance chạy cùng job → mỗi instance lấy batch khác nhau → parallel processing tự nhiên

### 8.3 Hot Slot (10.000 người tranh 1 ngày)

**Vấn đề:** Ngày lễ, cùng `(park_id, visit_date)` → DB row contention

**Giải pháp:** Inventory Sharding — 500 capacity → 10 shard × 50

**"Sao không dùng queue / waiting room?"**
- Queue thêm latency, UX kém ("bạn đang ở vị trí 2847" cho mua vé công viên thì quá)
- Sharding transparent — user không biết, không phải chờ, giảm contention N lần
- Queue chỉ cần khi capacity rất thấp (50 vé VIP, 10K người tranh)

### 8.4 Thundering Herd (Cache miss đồng loạt)

**Vấn đề:** Redis restart → 10K request cùng lúc cache miss → DB overload

**Giải pháp:** Singleflight pattern — 1 thread query DB, N-1 thread chờ cùng CompletableFuture

**"Sao không distributed lock (Redisson)?"**
- Redis đang down → distributed lock cũng down → vô nghĩa
- Singleflight: local, in-process, zero dependency. N instances × 1 query vẫn OK

### 8.5 Reconciliation (Phát hiện drift)

**3 levels:**
- **L1 (30s):** So sánh Redis vs DB → overwrite Redis
- **L2 (hourly):** Verify DB: SUM(booking quantity) == inventory counters
- **L3 (daily 2AM):** Full audit, report, alert

**"Sao không dùng 2PC (distributed transaction)?"**
- Redis không hỗ trợ XA/2PC
- 2PC: latency cao, availability giảm (1 node down → cả transaction block)
- Eventual consistency + reconciliation: đơn giản, 30s drift chấp nhận được

---

## 9. So sánh với các hệ thống khác — "Sao chọn approach này?"

### vs Airbnb (Pure Pessimistic Lock)

> "Airbnb mỗi listing chỉ 1 người đặt → contention gần 0 → pessimistic lock đủ. Công viên hàng ngàn người check availability cùng lúc, nếu mọi read đều hit DB → bottleneck. Em thêm Redis cache layer để tách read/write path."

### vs Shopee (Redis-first + Async DB)

> "Shopee flash sale cần throughput cực đại, chấp nhận eventual consistency. Booking vé công viên cần **confirm ngay** — user trả tiền xong phải biết chắc đã có chỗ. Ngoài ra Redis-first có risk: Redis crash trước khi persist DB → mất booking. Với Shopee (giá rẻ, volume cao) thì OK, với booking gia đình (giá cao) thì không chấp nhận được."

### vs Amazon (DynamoDB Pre-sharded Counters)

> "Over-engineering cho use case này. Data model đơn giản (park × date × ticket_type), vài chục ngàn keys. PostgreSQL + Redis xử lý tốt, chi phí thấp hơn nhiều so với DynamoDB."

---

## 10. Defense in Depth — Keyword hay cho phỏng vấn

```
Layer 1: Redis Lua pre-reserve     → Reject 90% invalid requests (< 1ms)
Layer 2: DB Optimistic Lock        → Chặn race condition ở DB level
Layer 3: CHECK constraint          → DB không cho phép reserved+booked > capacity
Layer 4: Reconciliation job        → Fix drift mỗi 30s
Layer 5: Deep audit                → Cross-check hourly + daily report
```

> "Dù bất kỳ layer nào fail, layer tiếp theo vẫn bắt được. Không có single point of failure cho data correctness."

---

## 11. Câu hỏi follow-up thường gặp

### "Throughput hệ thống bao nhiêu?"

- Read (availability): 100K+/s (Redis)
- Write (booking): Hàng ngàn/s (DB, tùy hardware). Với sharding × 10 → throughput × 10

### "Dùng message queue không?"

- **Outbox pattern** + polling publisher → at-least-once event delivery
- Không dùng queue cho booking flow chính (cần sync confirm)
- Saga pattern cho cross-service flows (inventory → payment → ticketing)

### "Idempotency?"

- Client gửi `Idempotency-Key` header (UUID) mỗi request
- Server check trước khi xử lý → return cached response nếu duplicate
- Event consumers cũng check `processed_events` table → chống duplicate processing

### "State Machine?"

```
PENDING → CONFIRMED → USED (terminal)
PENDING → EXPIRED (terminal)
PENDING → PAYMENT_FAILED → PENDING (retry)
CONFIRMED → CANCELLED → REFUNDED (terminal)
```
- Enforce trong Aggregate Root (DDD), không phải service layer
- Mỗi transition = 1 domain event → audit trail

### "Price thay đổi giữa chừng?"

- **Price snapshot** vào booking tại thời điểm tạo, không bao giờ recalculate
- Client gửi `expectedPrice` → server validate → mismatch thì trả lỗi kèm giá mới

### "Timezone?"

- DB lưu UTC (Instant) cho timestamps, DATE cho visit_date
- Business logic dùng park timezone (mỗi park có ZoneId)
- Inject `Clock` interface (testability)

### "Monitoring gì?"

- Alert khi `reserved` bất thường cao (nhiều abandon → có thể bot)
- Alert khi drift detected (reconciliation)
- Metrics: conversion rate (reserved → booked), abandon rate, P99 latency

### "Scale lên thì sao?"

Theo thứ tự, chỉ thêm khi cần:
1. Inventory sharding (giảm DB contention)
2. Redis Cluster (scale read/write)
3. Read replica PostgreSQL (scale read queries)
4. Virtual waiting room (flash sale events)
5. Nếu vẫn không đủ → chuyển sang Shopee-style Redis-first

---

## 12. Tóm tắt 1 câu cho từng quyết định

| Quyết định | 1 câu | Nếu bị challenge |
|-----------|-------|-------------------|
| Redis + DB (2 layer) | "Tách read/write path — read cần nhanh (Redis), write cần đúng (DB)" | "Airbnb không cần vì contention thấp, công viên traffic cao hơn nhiều" |
| Optimistic lock | "Redis filter traffic rồi → DB contention thấp → không cần serialize" | "Fallback sang pessimistic khi Redis down" |
| Lua script | "Cách duy nhất trong Redis để atomic read-check-write nhiều field" | "MULTI/EXEC không có conditional logic" |
| Write-through cache | "Redis phải luôn có data (triệu user check), drift conservative hơn cache-aside" | "Cache-aside invalidate → cache miss spike → DB overload" |
| Scheduled Job expiry | "DB là source of truth, idempotent, SKIP LOCKED cho parallel" | "Redis TTL notification là best-effort, miss thì DB stuck" |
| Sharding | "Giảm contention N lần, transparent cho user, không thêm latency" | "Queue chỉ khi capacity rất thấp hoặc cần strict FIFO" |
| Singleflight | "Redis down thì distributed lock cũng down, local dedup đủ" | "N instances × 1 query vẫn OK cho DB" |
| Reconciliation | "Redis không hỗ trợ 2PC, eventual consistency 30s là đủ" | "Drift luôn conservative → mất sale tạm, không oversell" |
| CHECK constraint | "Tuyến phòng thủ cuối, bất kể code bug hay Redis desync" | "Defense in depth — không tin tưởng bất kỳ single layer nào" |
