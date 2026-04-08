# Booking ↔ Payment Integration — Saga, Outbox, End-to-End Flow

> Booking và Payment là 2 domain riêng biệt. File này giải thích cách chúng phối hợp: ai gọi ai, event nào trigger gì, và xử lý failure thế nào.

---

## Mục lục

1. [Module Communication — Ai nói chuyện với ai?](#1-module-communication--ai-nói-chuyện-với-ai)
2. [Saga Pattern — Compensation khi fail](#2-saga-pattern--compensation-khi-fail)
3. [Outbox Pattern — Event không bao giờ mất](#3-outbox-pattern--event-không-bao-giờ-mất)
4. [End-to-End Flow — Từ chọn vé đến nhận vé](#4-end-to-end-flow--từ-chọn-vé-đến-nhận-vé)
5. [Failure Matrix — Mỗi bước có thể fail thế nào](#5-failure-matrix--mỗi-bước-có-thể-fail-thế-nào)
6. [Interview Quick Reference](#6-interview-quick-reference)

---

## 1. Module Communication — Ai nói chuyện với ai?

### 2 kiểu giao tiếp

```
Sync Call (──>):  Cần kết quả NGAY để tiếp tục xử lý
Domain Event (══>):  Thông báo "xong rồi", không chờ phản hồi
```

### Bản đồ giao tiếp

```
                  ──> Sync Call
                  ══> Domain Event

  ┌─────────┐        ┌──────────┐       ┌───────────┐
  │ Catalog  │◀──────│ Booking  │══════▶│  Payment  │
  └─────────┘  sync  │  (core)  │       └─────┬─────┘
                      └────┬─────┘             ║
              sync ┌───────┤                   ║ event
                   ▼       ▼ sync              ▼
            ┌──────────┐ ┌─────────┐    ┌───────────┐
            │ Inventory│ │ Pricing │    │ Ticketing │
            └──────────┘ └─────────┘    └───────────┘
```

### Sync Calls (cần answer ngay)

| Caller | Callee | Mục đích |
|--------|--------|----------|
| Booking → Inventory | Kiểm tra còn chỗ + reserve | |
| Booking → Pricing | Tính giá, lấy price snapshot | |
| Booking → Catalog | Validate loại vé tồn tại | |

### Domain Events (fire & forget)

```
BookingCreated
    ├══▶ Payment:    Tạo payment session
    └══▶ Inventory:  Reserve (giữ chỗ tạm)

PaymentCompleted
    ├══▶ Booking:    Chuyển status → CONFIRMED
    ├══▶ Ticketing:  Sinh vé + QR
    └══▶ Inventory:  Confirm reservation (reserved → booked)

PaymentFailed
    ├══▶ Booking:    Quyết định retry hay cancel
    └══▶ Inventory:  Release reservation (nếu booking cancel)

BookingCancelled
    ├══▶ Payment:    Refund nếu đã trả
    ├══▶ Ticketing:  Vô hiệu hóa vé
    └══▶ Inventory:  Release capacity
```

### Port + Adapter (Anti-Corruption Layer)

Booking **không import** domain model của Inventory. Chỉ biết interface:

```
┌─────────────────┐         ┌─────────────────┐
│     BOOKING      │         │    INVENTORY     │
│                  │         │                  │
│  Domain:         │         │  Application:    │
│   InventoryChecker (interface)    InventoryQueryHandler
│        ▲         │         │        ▲         │
│  Infrastructure: │         │                  │
│   InventoryAdapter ────────▶ (gọi trực tiếp)  │
└─────────────────┘         └─────────────────┘
```

Nếu sau tách microservices → thay adapter thành HTTP/gRPC client. Domain layer **KHÔNG ĐỔI**.

---

## 2. Saga Pattern — Compensation khi fail

### Booking Saga

Một lần đặt vé chạm nhiều module. Nếu bước 5 fail → phải undo bước 1, 3, 4.

```
Step 1: Reserve Inventory       ← Compensation: Release Inventory
Step 2: Validate & Snapshot Price  ← (read-only, không cần compensation)
Step 3: Create Booking (PENDING)   ← Compensation: Cancel Booking
Step 4: Create Payment             ← Compensation: Void/Expire Payment
Step 5: Process Payment (gateway)  ← Compensation: Refund Payment
Step 6: Confirm Booking            ← (final step)
Step 7: Issue Ticket               ← Compensation: Void Ticket
```

### Payment fail → Compensation chain

**Nếu còn retry (< 3 attempts) VÀ TTL chưa hết:**
```
Payment FAILED → Booking vẫn PENDING, inventory vẫn reserved
User chọn phương thức thanh toán khác → tạo payment attempt mới
UX: "Thanh toán thất bại. Thử lại?"
```

**Nếu hết retry HOẶC TTL hết:**
```
1. Void payment
2. Cancel booking → EXPIRED
3. Release inventory (reserved -= N)
4. Publish BookingCancelled event
```

### Tại sao cho retry mà không cancel ngay?

| Cancel ngay | Cho retry |
|-------------|-----------|
| User phải đặt lại từ đầu | User chỉ chọn method thanh toán khác |
| Inventory release → người khác mua mất | Inventory vẫn reserved cho user |
| UX tệ | UX tốt |

**Giới hạn**: max 3 attempts. TTL vẫn chạy — hết TTL thì expire bất kể.

### Ai quyết định gì?

```
Payment Service:  chỉ thông báo "fail" hoặc "success"
Booking Service:  quyết định retry, cancel, hay expire
Inventory Service: thực hiện release hoặc confirm
```

Tách trách nhiệm → dễ test, dễ thay đổi logic từng service.

### Choreography vs Orchestration

| Tiêu chí | Choreography | Orchestration |
|----------|--------------|---------------|
| Cách hoạt động | Mỗi service lắng nghe event, tự quyết | Central coordinator điều phối |
| Số bước ít (2-4) | **Tốt** | Overkill |
| Số bước nhiều (5+) | Khó quản lý | **Tốt** |
| Trace/debug | Khó (event đi lung tung) | Dễ (1 nơi nhìn toàn bộ) |

**Monolith**: Choreography + Spring Events + Outbox. Đủ đơn giản.
**Microservices**: Orchestration khi saga > 4 bước.

### Quy tắc Saga

- **Mọi step phải idempotent** — compensation có thể chạy nhiều lần
- **Compensation phải luôn thành công** — nếu fail → human intervention + alert
- **Set timeout cho mỗi step** — tránh saga treo
- **Tách read-only steps** — Pricing không cần compensation

---

## 3. Outbox Pattern — Event không bao giờ mất

### Bài toán

```
bookingService.confirmBooking()  // 1. Update booking → CONFIRMED
eventPublisher.publish(event)    // 2. Publish PaymentCompleted event

Nếu app crash giữa bước 1 và 2:
  → Booking đã CONFIRMED nhưng event KHÔNG publish
  → Ticketing không biết → không sinh vé
  → User: "Đã thanh toán nhưng không nhận vé"
```

### Giải pháp: Transactional Outbox

```
Thay vì publish event trực tiếp, LƯU event vào DB cùng transaction:

BEGIN;
  UPDATE booking SET status = 'CONFIRMED';
  INSERT INTO outbox_events (type, payload) VALUES ('BookingConfirmed', {...});
COMMIT;

→ Cả 2 thành công hoặc cả 2 rollback — KHÔNG bao giờ mất event
```

### Outbox publisher (background job)

```
Job chạy mỗi vài giây:

1. SELECT * FROM outbox_events WHERE published = FALSE ORDER BY created_at LIMIT 100
2. Publish mỗi event ra message broker (RabbitMQ / Spring Events)
3. UPDATE outbox_events SET published = TRUE WHERE id IN (...)
4. Nếu publish fail → skip, retry lần sau

Consumer phải idempotent (event có thể publish 2 lần nếu job crash giữa bước 2 và 3)
```

### Schema

```sql
CREATE TABLE outbox_events (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type  TEXT NOT NULL,          -- 'BookingConfirmed', 'PaymentCompleted'
    payload     JSONB NOT NULL,         -- event data
    published   BOOLEAN DEFAULT FALSE,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_outbox_unpublished ON outbox_events (created_at)
    WHERE published = FALSE;
```

### Tại sao không dùng Change Data Capture (CDC)?

| Outbox + polling | CDC (Debezium) |
|-----------------|----------------|
| Đơn giản, code-level | Infrastructure-level (Debezium + Kafka Connect) |
| Không cần thêm infra | Cần Kafka + Debezium |
| Delay = polling interval (vài giây) | Near real-time |
| **Phù hợp monolith** | Phù hợp microservices |

---

## 4. End-to-End Flow — Từ chọn vé đến nhận vé

```
Phase 1: BROWSE (Read path — Redis)
════════════════════════════════════
1. User mở app, chọn ngày 1/5
2. GET /availability → Redis → "còn 135 chỗ, giá 2,000円/vé" (< 1ms)

Phase 2: RESERVE (Write path — Redis + DB)
════════════════════════════════════
3. User chọn 5 vé, nhấn "Đặt vé"
4. POST /bookings {quantity: 5, expectedPrice: 2000, idempotencyKey: "uuid"}
5. Validate price (expectedPrice == currentPrice? → snapshot)
6. Redis Lua: pre-reserve (available -= 5)
7. DB optimistic lock: reserved += 5
8. Create booking (PENDING, expires_at = now + 15min)
9. Publish BookingCreated (outbox)
10. Response: {bookingId, status: PENDING, expires_at}

Phase 3: PAYMENT
════════════════════════════════════
11. User nhấn "Thanh toán" → chọn payment method
12. POST /bookings/{id}/pay {method, idempotencyKey, token}
13. Idempotency check → create payment record (attempt 1)
14. Gateway: charge bằng token (server không thấy card number)
15. Payment → PENDING, lưu gateway transaction reference

Phase 4: CONFIRMATION
════════════════════════════════════
16. Gateway callback (webhook) → payment SUCCESS
17. Verify: query gateway API ✓, verify amount ✓
18. Payment → SUCCESS
19. DB: reserved -= 5, booked += 5 (optimistic lock)
20. Redis Lua: confirm (reserved -= 5, booked += 5)
21. Booking → CONFIRMED
22. Publish BookingConfirmed (outbox)

Phase 5: TICKETING (Event-driven)
════════════════════════════════════
23. Outbox publisher → publish BookingConfirmed
24. Ticketing service → sinh QR code / e-ticket
25. Notification service → gửi email/SMS confirm + vé

Phase 6: USAGE
════════════════════════════════════
26. User đến công viên, scan QR
27. Ticket → USED, Booking → USED
```

---

## 5. Failure Matrix — Mỗi bước có thể fail thế nào

### Bảng tổng hợp

| Phase | Failure | Xử lý | Kết quả |
|-------|---------|-------|---------|
| **Reserve** | Redis reject (hết chỗ) | Trả "Sold out" ngay | User chọn ngày khác |
| **Reserve** | DB optimistic lock conflict | Rollback Redis, retry 3 lần | Retry trong hoặc trả lỗi |
| **Payment** | Gateway down | Circuit breaker, "thử lại sau" | Booking PENDING, TTL chạy |
| **Payment** | User không thanh toán | TTL 15min → expire | Release inventory |
| **Payment** | Payment fail (insufficient funds) | Booking vẫn PENDING, cho retry | Max 3 attempts |
| **Confirm** | Callback miss | Polling job query gateway mỗi 2 phút | Detect trong ≤ 2 phút |
| **Confirm** | Callback SAU khi expire | **Auto-refund** | Booking đã EXPIRED, thông báo đặt lại |
| **Confirm** | Redis confirm fail | Reconciliation 30s fix | DB vẫn correct |
| **Ticketing** | Service down | Event retry (outbox) | Booking đã CONFIRMED, vé sinh sau |
| **Mọi phase** | Data không khớp | Reconciliation daily 03:00 AM | Safety net cuối cùng |

### Case nguy hiểm nhất: "Tiền trừ nhưng vé không có"

```
1. User thanh toán → gateway charge OK
2. Callback miss (network issue)
3. Hệ thống không biết → payment vẫn PENDING
4. TTL hết → booking EXPIRED, inventory released

Giải pháp: Dual Confirmation (xem chi tiết tại 03-payment-challenges.md)
  - Webhook callback (primary)
  - Gateway polling mỗi 2 phút (fallback)
  - Query gateway LẦN CUỐI trước khi expire
```

### Case: Callback đến SAU khi expire

```
10:00 - Payment created (expires_at = 10:15)
10:15 - Timeout job: payment EXPIRED, booking EXPIRED, inventory released
10:16 - Callback đến: payment SUCCESS

Quyết định: AUTO-REFUND. KHÔNG cố confirm lại.
  → Inventory có thể đã bán cho người khác
  → Booking EXPIRED là terminal state
  → Thông báo: "Thanh toán trễ, đã hoàn tiền, vui lòng đặt lại"
```

---

## 6. Interview Quick Reference

### "Booking và Payment phối hợp thế nào?"

> "Tách hoàn toàn qua domain events. Payment chỉ thông báo 'success/fail'. Booking quyết định confirm/retry/expire. Inventory thực hiện reserve/release/confirm. 3 state machines chạy độc lập, liên kết qua event. Outbox pattern đảm bảo event không mất."

### "Saga compensation khi payment fail?"

> "Nếu còn retry (< 3) và TTL chưa hết → booking vẫn PENDING, user retry method khác. Hết retry hoặc TTL → void payment, expire booking, release inventory, publish BookingCancelled. Retry trước cancel vì UX tốt hơn — user không phải đặt lại từ đầu."

### "Sao đảm bảo event không mất?"

> "Transactional Outbox: lưu event vào DB cùng transaction với business logic (cùng COMMIT). Background job poll outbox → publish. Nếu publish fail → retry. Consumer phải idempotent vì event có thể publish 2 lần. Đây là at-least-once delivery."

### "End-to-end flow?"

> "6 phase: Browse (Redis read) → Reserve (Redis pre-check + DB lock) → Payment (gateway charge, token-based) → Confirmation (dual: webhook + polling) → Ticketing (event-driven, outbox) → Usage (scan QR). Mỗi phase có failure handling riêng, reconciliation là safety net cuối cùng."
