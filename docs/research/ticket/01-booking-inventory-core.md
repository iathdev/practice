# Booking & Inventory — Core Problems & Solutions

> Từ lúc user chọn vé đến khi giữ chỗ thành công — mọi thứ có thể sai, và cách thiết kế để không bao giờ oversell.

---

## Mục lục

1. [Inventory Model — Lưu gì trong database?](#1-inventory-model--lưu-gì-trong-database)
2. [Race Condition & Overselling](#2-race-condition--overselling)
3. [Kiến trúc recommend: Redis Pre-check + DB Optimistic Lock](#3-kiến-trúc-recommend-redis-pre-check--db-optimistic-lock)
4. [Reservation TTL — Giữ chỗ tạm](#4-reservation-ttl--giữ-chỗ-tạm)
5. [Booking State Machine](#5-booking-state-machine)
6. [Price Consistency — Giá thay đổi giữa chừng](#6-price-consistency--giá-thay-đổi-giữa-chừng)
7. [Idempotency — Chống duplicate booking](#7-idempotency--chống-duplicate-booking)
8. [Interview Quick Reference](#8-interview-quick-reference)

---

## 1. Inventory Model — Lưu gì trong database?

### Model recommend: Tách `total_capacity` / `reserved` / `booked`

```sql
CREATE TABLE daily_inventory (
    park_id         UUID NOT NULL,
    ticket_type_id  UUID NOT NULL,
    visit_date      DATE NOT NULL,
    total_capacity  INT NOT NULL,           -- 500 (admin set, ít thay đổi)
    reserved        INT NOT NULL DEFAULT 0, -- đang giữ chỗ tạm (chờ payment)
    booked          INT NOT NULL DEFAULT 0, -- đã thanh toán xong
    version         INT NOT NULL DEFAULT 0, -- optimistic locking
    PRIMARY KEY (park_id, ticket_type_id, visit_date),

    -- Safety net: KHÔNG BAO GIỜ oversell, dù code có bug
    CONSTRAINT chk_no_oversell CHECK (reserved + booked <= total_capacity),
    CONSTRAINT chk_non_negative CHECK (reserved >= 0 AND booked >= 0)
);

-- available = total_capacity - reserved - booked (tính toán, KHÔNG lưu riêng)
```

### Tại sao tách 3 field?

```
Luồng hoạt động:

  Khách chọn 5 vé → Reserve:
      reserved += 5           (available giảm 5)

  Payment thành công → Confirm:
      reserved -= 5, booked += 5  (available không đổi, chuyển từ reserved → booked)

  Payment fail / TTL hết → Release:
      reserved -= 5           (available tăng lại 5)

  Cancel booking đã confirmed → Release:
      booked -= 5             (available tăng lại 5)
```

| Trạng thái | Ý nghĩa | Khi nào thay đổi |
|-----------|---------|------------------|
| `total_capacity` | Sức chứa tối đa (admin set) | Hiếm khi đổi |
| `reserved` | Đang giữ chỗ tạm, chờ thanh toán | +N khi reserve, -N khi confirm hoặc expire |
| `booked` | Đã thanh toán thành công | +N khi payment success |
| `available` (computed) | = total - reserved - booked | Không lưu, tính từ 3 field trên |

**Tại sao không dùng single `remaining`?** → `remaining` là giá trị derived, không biết bao nhiêu đang giữ tạm vs đã confirm. Nếu cancel booking mà quên cộng lại → drift. Không có audit trail.

### So sánh các Inventory Model

| Model | Khi nào dùng | Nhược điểm chính |
|-------|-------------|------------------|
| Single `remaining` | POC, prototype | Drift risk, không phân biệt reserved/booked |
| **Tách reserved/booked** | **Production (recommend)** | Cần sync khi reserve/confirm/release |
| COUNT từ bookings | Reporting, low traffic | Chậm, không atomic |
| Individual tickets (mỗi vé 1 row) | Assigned-seat (concert, cinema) | Overkill cho General Admission |

---

## 2. Race Condition & Overselling

### Bài toán

Công viên còn 6 chỗ. Cùng lúc:
- Khách A đặt **5 vé**, Khách B đặt **3 vé**

```
Thread A: READ available = 6  →  check 5 ≤ 6 ✓  →  UPDATE reserved += 5
Thread B: READ available = 6  →  check 3 ≤ 6 ✓  →  UPDATE reserved += 3
                    ↑ đọc cùng giá trị cũ vì A chưa commit
Kết quả: bán 8 vé cho 6 chỗ → OVERSELL
```

**Nguyên nhân gốc**: thao tác **read-check-write** không phải atomic.

### Concurrency Solutions

#### A. Pessimistic Locking (SELECT FOR UPDATE)

```sql
BEGIN;
SELECT total_capacity, reserved, booked
FROM daily_inventory
WHERE park_id = ? AND ticket_type_id = ? AND visit_date = ?
FOR UPDATE;  -- lock row, thread khác phải chờ

-- application check
-- available = total_capacity - reserved - booked
-- IF available < quantity THEN ROLLBACK;

UPDATE daily_inventory
SET reserved = reserved + :quantity
WHERE park_id = ? AND ticket_type_id = ? AND visit_date = ?;
COMMIT;
```

| Ưu | Nhược |
|---|---|
| Chắc chắn không oversell | Serialize mọi request trên cùng row → throughput thấp |
| Dễ hiểu, dễ debug | Deadlock risk nếu lock nhiều row |

#### B. Optimistic Locking (Version column)

```sql
SELECT total_capacity, reserved, booked, version
FROM daily_inventory WHERE ...;

-- App check: available >= quantity

UPDATE daily_inventory
SET reserved = reserved + :quantity, version = version + 1
WHERE ... AND version = :readVersion;
-- affected rows = 0 → conflict → retry
```

| Ưu | Nhược |
|---|---|
| Không lock → throughput cao hơn | Retry storm khi contention cao |
| JPA `@Version` hỗ trợ sẵn | Cần retry logic + max attempts |

#### C. Atomic SQL (Conditional UPDATE)

```sql
UPDATE daily_inventory
SET reserved = reserved + :quantity
WHERE park_id = ? AND ticket_type_id = ? AND visit_date = ?
  AND (total_capacity - reserved - booked) >= :quantity;
-- 1 câu SQL, database đảm bảo atomic
-- affected rows = 1 → OK, 0 → hết chỗ
```

| Ưu | Nhược |
|---|---|
| Đơn giản nhất, 1 query | Khó thêm business logic phức tạp |
| DB đảm bảo atomicity | Chỉ phù hợp check đơn giản |

#### D. Redis Pre-Check + Database Confirm (Recommend cho high traffic)

```
1. Redis Lua: DECRBY capacity:{key} :quantity
   → result < 0 → INCRBY hoàn lại → reject ngay (không đụng DB)
   → result >= 0 → tiếp tục

2. Database: Optimistic Lock UPDATE (source of truth)
   → DB fail → INCRBY Redis hoàn lại (compensation)

3. Reconciliation: sync Redis = DB mỗi 30s-1min
```

| Ưu | Nhược |
|---|---|
| Reject sớm, giảm DB load đáng kể | Thêm Redis, cần sync 2 layer |
| Redis: 100K+ ops/sec | Compensation khi fail |

### Recommend

**Inventory**: Model 2 — `total_capacity` / `reserved` / `booked` + CHECK constraint

**Concurrency**: Bắt đầu với Pessimistic Lock hoặc Atomic SQL. Chuyển sang Redis pre-check khi đo được bottleneck thật sự.

**Safety**: CHECK constraint + reconciliation job. Defense in depth.

---

## 3. Kiến trúc recommend: Redis Pre-check + DB Optimistic Lock

Tham khảo từ Disney Parks (Genie+, Lightning Lane, Park Reservation).

### Tổng quan

```
┌──────────┐     ┌──────────────────────────────────────────────────────┐
│  Client   │     │              Booking Service                         │
│           │     │                                                      │
│ GET avail ├────▶│  Redis Cache ──────── read path (< 1ms)             │
│           │     │  ┌──────────────┐                                    │
│ POST book ├────▶│  │ Lua: atomic  │──── pre-check (reject sớm)        │
│           │     │  │ check+decr   │                                    │
│           │     │  └──────┬───────┘                                    │
│           │     │         │ passed                                     │
│           │     │         ▼                                            │
│           │     │  ┌──────────────┐                                    │
│           │     │  │ DB Optimistic│──── source of truth                │
│           │     │  │ Lock UPDATE  │                                    │
│           │     │  └──────┬───────┘                                    │
│           │     │         │ success                                    │
│           │     │         ▼                                            │
│           │     │  Create Booking (PENDING, TTL 15min)                 │
└──────────┘     └──────────────────────────────────────────────────────┘
```

### Redis Layer

```
Key pattern: inventory:{park_id}:{ticket_type_id}:{visit_date}
Value: Hash { total: 500, reserved: 120, booked: 350 }
TTL: end of visit_date + 1h
```

**Lua script cho atomic pre-reserve:**

```lua
local key = KEYS[1]
local qty = tonumber(ARGV[1])
local total = tonumber(redis.call('HGET', key, 'total') or 0)
local reserved = tonumber(redis.call('HGET', key, 'reserved') or 0)
local booked = tonumber(redis.call('HGET', key, 'booked') or 0)
local available = total - reserved - booked

if available < qty then
    return -1  -- không đủ chỗ
end

redis.call('HINCRBY', key, 'reserved', qty)
return available - qty  -- remaining after reserve
```

### DB Optimistic Lock

```sql
UPDATE daily_inventory
SET reserved = reserved + :quantity,
    version = version + 1
WHERE park_id = :parkId
  AND ticket_type_id = :ticketTypeId
  AND visit_date = :visitDate
  AND version = :expectedVersion
  AND (total_capacity - reserved - booked) >= :quantity;
```

### Khi DB fail → Rollback Redis

```
Redis reserve OK → DB conflict (version mismatch) →
  Redis: HINCRBY key reserved -quantity (hoàn lại)
  Retry: đọc DB version mới → thử lại (max 3 lần)
```

### Reconciliation (sửa drift Redis ↔ DB)

```
Job chạy mỗi 30 giây:

1. SELECT reserved, booked FROM daily_inventory WHERE visit_date = TODAY
2. Với mỗi row: HSET inventory:{key} reserved {db_reserved} booked {db_booked}
3. Log nếu phát hiện drift > threshold

Tại sao cần?
- Redis là cache, DB là source of truth
- App crash giữa Redis update và DB update → drift
- Network issue → compensation không đến Redis
- Reconciliation = safety net cuối cùng
```

### Redis Down → Fallback

```
Redis unreachable?
  → Bypass Redis, gọi thẳng DB (Pessimistic Lock)
  → Chậm hơn nhưng correct
  → Circuit breaker: sau N failures → switch to DB-only mode
  → Khi Redis recover → reconciliation sync lại
```

---

## 4. Reservation TTL — Giữ chỗ tạm

### Bài toán

Khách chọn vé → reserve → nhưng không thanh toán → chỗ bị "treo" mãi.

### Two-Phase Reservation

```
Phase 1: RESERVE (temporary hold)
    User chọn vé → System reserve + set TTL
    Booking status: PENDING, expires_at = now + 15min

Phase 2: CONFIRM hoặc RELEASE
    Payment success → CONFIRM (reserved → booked)
    Payment fail / TTL hết → RELEASE (reserved -= N)
```

### Implementation: DB expires_at + Scheduled Job

```sql
-- Booking có expires_at
INSERT INTO booking (id, status, expires_at)
VALUES (?, 'PENDING', NOW() + INTERVAL '15 minutes');

-- Scheduled job mỗi 15-30 giây
UPDATE booking SET status = 'EXPIRED'
WHERE status = 'PENDING' AND expires_at < NOW()
RETURNING id, park_id, ticket_type_id, visit_date, quantity;
-- → Release inventory cho mỗi expired booking
```

**Bổ sung Delay Queue (precision):**
```
Khi reserve → publish delayed message (delay = TTL)
    Message: { bookingId, action: "CHECK_EXPIRY" }

Khi message arrive:
    if booking.status == PENDING → expire + release
    if booking.status == CONFIRMED → ignore (đã thanh toán)
```

DB scheduled job là safety net, delay queue cho precision. Cả 2 chạy song song.

### TTL nên bao lâu?

| Loại | TTL | Lý do |
|------|-----|-------|
| Concert ticket (hot) | 5-8 phút | Demand cực cao, cần release nhanh |
| Park ticket (normal) | 10-15 phút | Đủ thời gian thanh toán |
| Hotel room | 20-30 phút | Quy trình phức tạp hơn |

**TTL phải > thời gian payment trung bình, < mức chấp nhận business.**

### Best Practice

- Hiển thị **countdown** cho user → tạo urgency + giảm abandon
- Cho phép **extend 1 lần** nếu user đang active
- **Graceful expiry**: warning 2 phút trước khi hết hạn
- **Luôn query gateway trước khi expire** (xem payment-challenges.md — "Tiền trừ nhưng vé không có")

---

## 5. Booking State Machine

### States & Transitions

```
              User chọn vé + reserve
                      │
                      ▼
                ┌──────────┐
                │  PENDING  │──── Đang chờ thanh toán
                └─────┬─────┘
                      │
          ┌───────────┼───────────┐
          │           │           │
          ▼           ▼           ▼
    ┌──────────┐ ┌─────────┐ ┌──────────┐
    │CONFIRMED │ │ EXPIRED │ │CANCELLED │
    │(đã trả)  │ │(hết TTL)│ │(user hủy)│
    └─────┬────┘ └─────────┘ └──────────┘
          │
          ▼
    ┌──────────┐
    │   USED   │──── Đã scan vé vào cổng
    └──────────┘
```

### Transitions & Triggers

| From | To | Trigger | Side Effect |
|------|-----|---------|-------------|
| — | PENDING | User đặt vé | Reserve inventory, set TTL |
| PENDING | CONFIRMED | Payment SUCCESS | reserved → booked, issue ticket |
| PENDING | EXPIRED | TTL hết | Release reserved inventory |
| PENDING | CANCELLED | User cancel trước khi pay | Release reserved inventory |
| CONFIRMED | CANCELLED | User cancel sau khi pay | Release booked, trigger refund |
| CONFIRMED | USED | Scan QR vào cổng | Mark ticket used |

### Quy tắc quan trọng

- **EXPIRED và USED là terminal states** — không revert
- **CONFIRMED → CANCELLED** phải trigger refund (async)
- **Chỉ PENDING mới có TTL** — CONFIRMED không expire
- Payment FAILED → booking **vẫn PENDING** (cho retry), chỉ expire khi hết TTL

---

## 6. Price Consistency — Giá thay đổi giữa chừng

### Bài toán

```
10:00 - User xem giá: 2,000円/vé
10:05 - Admin đổi giá: 3,000円/vé
10:06 - User nhấn "Đặt vé" → Tính giá nào? 2,000 hay 3,000?
```

### Giải pháp: Price Snapshot

Khi tạo booking, **snapshot giá tại thời điểm đó** vào booking record:

```sql
CREATE TABLE booking_items (
    booking_id      UUID NOT NULL,
    ticket_type_id  UUID NOT NULL,
    quantity        INT NOT NULL,
    unit_price      BIGINT NOT NULL,    -- giá snapshot tại thời điểm đặt (JPY)
    total_price     BIGINT NOT NULL,    -- unit_price × quantity
    price_version   INT NOT NULL        -- version giá đã áp dụng
);
```

**Flow:**
1. User xem giá → GET price (có version)
2. User đặt vé → POST booking { expectedPrice, priceVersion }
3. Server check: `priceVersion == currentVersion`?
   - Khớp → tạo booking với giá đó
   - Không khớp → reject, trả giá mới cho user confirm lại

**Payment dùng giá trong booking record, KHÔNG query giá mới.** Tránh "admin đổi giá giữa chừng".

### Kiểu dữ liệu cho tiền

```
JPY không có decimal → dùng BIGINT (2000円 = 2000, không × 100)
Tính thuế → BigDecimal, KHÔNG BAO GIỜ float/double
  消費税 10%: 1999 × 1.10 = 2198.9 → BigDecimal xử lý rounding chính xác
Với tiền, sai 1円 cũng không chấp nhận được
```

---

## 7. Idempotency — Chống duplicate booking

### Bài toán

User nhấn "Đặt vé" 2 lần nhanh → 2 booking, 2 lần reserve → oversell.

### Defense in Depth — 3 Layer

```
Layer 1: Client     → Disable button sau click đầu tiên
Layer 2: API        → Idempotency key header
Layer 3: Database   → UNIQUE constraint
```

| Layer | Chặn gì | Nếu thiếu |
|-------|---------|-----------|
| Client disable button | 99% double-click | Script/curl bypass |
| Idempotency key | Duplicate request cùng key | 2 request khác key cùng booking |
| DB UNIQUE constraint | Chặn ở tầng data | Race condition giữa check và insert |

**Flow:**
1. Client generate UUID → gửi header `Idempotency-Key`
2. Server check: đã có booking cho key này? → Có → trả response cũ
3. Không có → tạo booking mới
4. DB UNIQUE constraint trên `idempotency_key` → bắt mọi race condition

---

## 8. Interview Quick Reference

### "Inventory model nên thiết kế thế nào?"

> "Tách 3 field: total_capacity, reserved, booked. Available = total - reserved - booked, tính toán không lưu riêng. CHECK constraint `reserved + booked <= total_capacity` là safety net cuối cùng. Pattern này cho phép: reserve tạm khi chờ payment, confirm khi thành công, release khi fail/expire."

### "Chống oversell thế nào?"

> "Defense in depth 3 lớp: (1) Redis pre-check reject sớm khi hết chỗ (100K ops/s). (2) DB optimistic lock là source of truth — version column đảm bảo atomic. (3) CHECK constraint ở DB level — dù code bug cũng không vượt capacity. Reconciliation job sync Redis ↔ DB mỗi 30 giây."

### "TTL reservation hoạt động thế nào?"

> "Booking PENDING có expires_at. Scheduled job scan mỗi 15-30 giây + delay queue cho precision. Hết TTL → expire booking + release inventory. Quan trọng: trước khi expire, luôn query payment gateway lần cuối — tránh case 'tiền trừ nhưng vé không có'."

### "Tại sao Redis + DB mà không chỉ DB?"

> "Read path: Redis trả available trong < 1ms (user xem còn chỗ không). Write path: Redis reject ngay khi hết chỗ, chỉ request hợp lệ mới đụng DB → giảm DB load 80-90%. DB vẫn là source of truth. Redis down → fallback thẳng DB. Reconciliation sửa drift."
