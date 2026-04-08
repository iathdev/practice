# Payment Flow — Khó khăn & Thử thách khi thiết kế hệ thống thanh toán

> Từ lúc user nhấn "Đặt vé" đến khi nhận được vé — mọi thứ có thể sai ở giữa, và cách thiết kế để handle.

---

## Mục lục

1. [Tổng quan: Payment nằm ở đâu trong Booking Flow](#1-tổng-quan-payment-nằm-ở-đâu-trong-booking-flow)
2. [Thử thách 1: Tách Payment khỏi Booking — Boundary ở đâu?](#2-thử-thách-1-tách-payment-khỏi-booking--boundary-ở-đâu)
3. [Thử thách 2: Payment State Machine — Thiết kế sao cho đúng?](#3-thử-thách-2-payment-state-machine--thiết-kế-sao-cho-đúng)
4. [Thử thách 3: Double Payment — User nhấn 2 lần](#4-thử-thách-3-double-payment--user-nhấn-2-lần)
5. [Thử thách 4: "Tiền trừ nhưng vé không có" — Bài toán nguy hiểm nhất](#5-thử-thách-4-tiền-trừ-nhưng-vé-không-có--bài-toán-nguy-hiểm-nhất)
6. [Thử thách 5: Webhook không đáng tin — Callback miss, trùng, hoặc đến trễ](#6-thử-thách-5-webhook-không-đáng-tin--callback-miss-trùng-hoặc-đến-trễ)
7. [Thử thách 6: Gateway Down — Thanh toán không khả dụng](#7-thử-thách-6-gateway-down--thanh-toán-không-khả-dụng)
8. [Thử thách 7: Refund — Hoàn tiền không đơn giản như tưởng](#8-thử-thách-7-refund--hoàn-tiền-không-đơn-giản-như-tưởng)
9. [Thử thách 8: Amount Mismatch — Số tiền không khớp](#9-thử-thách-8-amount-mismatch--số-tiền-không-khớp)
10. [Thử thách 9: Reconciliation — Đối soát hàng ngày](#10-thử-thách-9-reconciliation--đối-soát-hàng-ngày)
11. [Thử thách 10: Saga — Compensation chain khi payment fail](#11-thử-thách-10-saga--compensation-chain-khi-payment-fail)
12. [End-to-End: Booking → Payment → Confirm → Ticket](#12-end-to-end-booking--payment--confirm--ticket)
13. [Interview Quick Reference](#13-interview-quick-reference)

---

## 1. Tổng quan: Payment nằm ở đâu trong Booking Flow

### Booking Flow nhìn từ xa

```
User chọn vé → Reserve inventory → Tạo booking (PENDING)
    → Thanh toán → Confirm booking → Sinh vé
```

Payment nằm **giữa** luồng — sau khi inventory đã reserve, trước khi booking confirm. Đây là điểm **nguy hiểm nhất** vì:

- **Trước payment**: inventory đã lock cho user (nếu payment fail → phải release)
- **Sau payment**: tiền đã trừ (nếu confirm fail → phải refund)
- **Trong payment**: có thể timeout, gateway down, callback miss, double charge...

### Mối liên hệ Payment ↔ Booking ↔ Inventory

```
                    ┌─────────────┐
                    │  Inventory   │
                    │  (Redis+DB)  │
                    └──────┬──────┘
                           │
              reserve ◄────┤────► release (nếu fail/expire)
              confirm ◄────┤────► (reserved → booked)
                           │
┌──────────┐        ┌──────┴──────┐        ┌──────────────┐
│  Client  │───────▶│   Booking   │───────▶│   Payment    │
│          │◀───────│   Service   │◀───────│   Service    │
└──────────┘        └─────────────┘        └──────┬───────┘
                                                   │
                                           ┌───────▼───────┐
                                           │    Payment    │
                                           │    Gateway    │
                                           └───────────────┘
```

**Nguyên tắc cốt lõi**: Payment chỉ phụ trách "thu tiền". Booking quyết định "làm gì khi thanh toán xong/fail". Inventory chỉ biết "reserve/release/confirm". Ba thứ này **phải tách** — và việc tách này chính là thử thách đầu tiên.

---

## 2. Thử thách 1: Tách Payment khỏi Booking — Boundary ở đâu?

### Vấn đề

Cách dễ nhất: booking table thêm vài field `paid_at`, `payment_status`, `gateway_txn_id`... xong.

Nhưng sẽ gặp ngay:
- Booking table phình to — nhiều field nullable vì "có payment mới có giá trị"
- User thanh toán fail → retry → cần track attempt history → booking table càng phình
- Đổi gateway (hoặc thêm payment method mới) → sửa booking code
- Refund có lifecycle riêng (PENDING → PROCESSING → SUCCESS/FAILED) → lẫn vào booking logic
- Reconciliation (đối soát) cần query payment data riêng → query booking table vô cùng phức tạp

### Quyết định thiết kế

**Tách 3 table**: `payments`, `payment_events`, `refunds`.

| Table | Trách nhiệm |
|-------|-------------|
| `payments` | Mỗi lần thanh toán: amount, status, gateway info, idempotency |
| `payment_events` | Audit trail — mọi thay đổi trạng thái (ai, khi nào, data gì) |
| `refunds` | Mỗi lần hoàn tiền — lifecycle riêng, có thể partial refund |

**Mối quan hệ: 1 Booking → N Payment Attempts**. User thanh toán lần 1 fail → tạo payment attempt 2 → thành công → booking confirmed.

### Gateway Adapter Pattern

Dù ban đầu chỉ dùng 1 gateway, vẫn nên abstract qua interface. Lý do:
- Gateway hỗ trợ nhiều payment method (credit card, konbini, carrier billing, e-wallet...) — mỗi loại có flow riêng
- Test dễ: mock interface
- Thêm payment method mới → chỉ implement adapter, booking không đổi dòng nào

### Token-based Flow — Tại sao server không nên thấy card number?

```
Client (browser)               Gateway Token Server          Our Server
     │                                │                          │
     │ 1. User nhập card info         │                          │
     │ 2. JS SDK gửi card trực       │                          │
     │    tiếp sang gateway           │                          │
     │───────────────────────────────▶│                          │
     │◀── token (1 lần dùng)         │                          │
     │                                │                          │
     │ 3. Gửi token (KHÔNG phải card)│                          │
     │───────────────────────────────────────────────────────────▶│
     │                                │  4. Charge bằng token    │
     │                                │◀─────────────────────────│
     │                                │─────────────────────────▶│
```

Server KHÔNG BAO GIỜ thấy raw card number → giảm PCI DSS compliance scope đáng kể. Đây không phải optional — mà là **bắt buộc** nếu không muốn audit PCI DSS full scope.

### 2 loại callback từ gateway

| | ResultURL (client redirect) | NotifyURL/Webhook (server-to-server) |
|---|---|---|
| **Kiểu** | Browser redirect | HTTP POST server-to-server |
| **Đáng tin?** | KHÔNG — user đóng browser, mất mạng | CÓ — gateway retry nếu server không respond |
| **Dùng để** | UX — hiển thị "thanh toán thành công" | Business logic — xác nhận thanh toán |

**Nguyên tắc: KHÔNG BAO GIỜ đánh dấu "đã thanh toán" dựa vào client redirect. Chỉ tin server-to-server callback hoặc API response.**

---

## 3. Thử thách 2: Payment State Machine — Thiết kế sao cho đúng?

### Vấn đề

Payment không phải chỉ có "chưa trả" và "đã trả". Có rất nhiều trạng thái trung gian và edge case:
- User đang ở trang gateway → PENDING
- Gateway trả lỗi → FAILED (nhưng có thể retry)
- Quá thời gian → EXPIRED
- Thanh toán thành công nhưng cần hoàn → REFUNDED

### State Machine

```
              create
                │
                ▼
          ┌──────────┐
          │  CREATED  │──── gateway nhận request
          └─────┬─────┘
                │ đang xử lý thanh toán
                ▼
          ┌──────────┐
          │  PENDING  │──── đang chờ kết quả
          └─────┬─────┘
                │
      ┌─────────┼──────────┐
      │         │          │
      ▼         ▼          ▼
┌──────────┐ ┌────────┐ ┌──────────┐
│ SUCCESS  │ │ FAILED │ │ EXPIRED  │
│          │ │        │ │(timeout) │
└─────┬────┘ └────┬───┘ └──────────┘
      │           │
      │           ▼
      │     ┌──────────┐
      │     │  PENDING  │ (retry with new payment attempt)
      │     └──────────┘
      ▼
┌──────────┐
│ REFUNDED │ (partial hoặc full)
└──────────┘
```

### Transitions & Side Effects — Liên kết với Booking

| From | To | Trigger | Side Effect (liên kết Booking) |
|------|-----|---------|-------------------------------|
| CREATED | PENDING | Gateway nhận request | Lưu gateway transaction reference |
| PENDING | SUCCESS | Callback/API response thành công | **Confirm booking** (reserved → booked), publish PaymentCompleted |
| PENDING | FAILED | Callback/API response thất bại | Publish PaymentFailed → booking quyết định retry hay release |
| PENDING | EXPIRED | Timeout (15 min) | **Release inventory**, booking → EXPIRED |
| SUCCESS | REFUNDED | Admin/user cancel | Gọi gateway refund, **release inventory** |

### Tại sao Payment có state machine riêng, tách khỏi Booking?

Booking có state machine riêng: `PENDING → CONFIRMED → USED / CANCELLED / EXPIRED`

Payment có state machine riêng: `CREATED → PENDING → SUCCESS / FAILED / EXPIRED → REFUNDED`

Hai state machine **chạy độc lập nhưng liên kết qua event**:
- Payment SUCCESS → trigger Booking CONFIRMED
- Payment FAILED → Booking quyết định retry (vẫn PENDING) hay cancel
- Payment EXPIRED → Booking EXPIRED, inventory release

Nếu gộp → một entity phải quản lý cả 2 lifecycle → phức tạp, khó maintain, khó test.

---

## 4. Thử thách 3: Double Payment — User nhấn 2 lần

### Vấn đề

User nhấn "Thanh toán" 2 lần nhanh:
- Request 1: tạo payment → charge card
- Request 2: tạo payment MỚI → charge card LẦN NỮA → user trả tiền 2 lần

### Giải pháp: Defense in Depth — 3 Layer

```
Layer 1: Client-side    → Disable button sau click đầu tiên
Layer 2: API            → Idempotency key header
Layer 3: Database       → UNIQUE constraint trên (booking_id, attempt_number)
```

### Tại sao cần cả 3 layer?

| Layer | Chặn gì | Nếu thiếu |
|-------|---------|-----------|
| Client disable button | 99% double-click từ user bình thường | User dùng script/curl → bypass |
| Idempotency key | Duplicate request có cùng key | 2 request khác key nhưng cùng booking → tạo 2 payment |
| DB UNIQUE constraint | Chặn ở tầng data, bất kể bug code | Race condition giữa check và insert |

**Đây là pattern giống hệt chống oversell ở inventory** — defense in depth, không tin bất kỳ single layer nào.

### Idempotency key hoạt động thế nào?

1. Client generate UUID, gửi kèm header `Idempotency-Key`
2. Server check: đã có payment cho key này chưa? → Có → trả response cũ
3. Check: booking đã có payment PENDING chưa? → Có → trả payUrl cũ
4. Không có → tạo payment mới

**Liên kết Booking**: Mỗi booking có tối đa 1 payment PENDING tại một thời điểm. Payment fail → tạo attempt mới (attempt_number tăng, gateway_order_id mới).

---

## 5. Thử thách 4: "Tiền trừ nhưng vé không có" — Bài toán nguy hiểm nhất

### Vấn đề

```
1. User thanh toán bằng credit card
2. Gateway charge thành công
3. Gateway gửi callback → nhưng server đang deploy / network issue → callback fail
4. Gateway retry callback vài lần → vẫn fail → ngừng retry
5. Hệ thống: payment = PENDING, booking = PENDING
6. Timeout job: payment EXPIRED, booking EXPIRED, inventory released
7. User: "Tiền bị trừ nhưng không nhận được vé!"
```

Đây là **bài toán nguy hiểm nhất** trong payment — và là thứ phân biệt hệ thống production với toy project.

### Giải pháp: Dual Confirmation (Belt and Suspenders)

**Không chỉ chờ callback. Chủ động query gateway.**

```
                    Webhook callback ────────┐
                    (primary)               │
                                            ▼
                                     ┌──────────────┐
Payment PENDING ─────────────────────│ handleSuccess │──▶ Booking CONFIRMED
                                     └──────────────┘
                                            ▲
                    Polling job  ───────────┘
                    (fallback, mỗi 2 phút)
```

**2 job chạy song song:**

| Job | Tần suất | Làm gì |
|-----|----------|--------|
| **Status Polling** ("Belt") | Mỗi 2 phút | Query gateway API cho tất cả payment PENDING > 5 phút. Nếu gateway nói SUCCESS → confirm |
| **Timeout Expiry** ("Suspenders") | Mỗi 30 giây | Check payment quá TTL (15 phút). **Query gateway LẦN CUỐI** trước khi expire |

### Tại sao query gateway trước khi expire?

```
Scenario:
  User thanh toán lúc 10:14:50
  Payment expires_at = 10:15:00
  Callback chưa đến (network delay)
  Timeout job chạy lúc 10:15:05

  Nếu expire ngay          → user mất tiền, booking cancelled  ✗
  Nếu query gateway trước  → gateway nói SUCCESS → confirm     ✓
```

Chi phí: 1 API call. Lợi ích: tránh case "tiền trừ nhưng vé không có".

**Stripe cũng recommend pattern này**: listen webhook + poll as fallback.

### Liên kết Booking

- Polling job phát hiện SUCCESS → gọi `bookingService.confirmBooking()` → reserved → booked
- Timeout job expire → gọi `bookingService.expireBooking()` → release inventory
- **Luôn query gateway trước khi kết luận**. Nếu gateway nói thành công → confirm bất kể timeout

---

## 6. Thử thách 5: Webhook không đáng tin — Callback miss, trùng, hoặc đến trễ

### Callback đến trùng (duplicate)

Gateway retry khi không nhận response, hoặc network glitch gửi 2 lần.

**Giải pháp**: Idempotent handler.
1. Check: payment đã SUCCESS rồi? → skip, trả OK
2. Validate state transition: chỉ PENDING → SUCCESS là hợp lệ
3. Optimistic locking: 2 thread xử lý cùng lúc → 1 thành công, 1 conflict → retry

### Callback miss

Đã giải quyết bằng Polling job (Thử thách 4). Thêm 1 trick:

Khi user quay về từ gateway (ResultURL redirect) → **trigger immediate poll**. Vì ResultURL chứng tỏ user đã thao tác xong trên gateway → callback sắp đến hoặc đã miss.

### Callback đến SAU khi đã expire — Bài toán khó nhất

```
Timeline:
  10:00 - Payment created (expires_at = 10:15)
  10:15 - Timeout job expire payment → booking EXPIRED, inventory released
  10:16 - Callback đến: payment SUCCESS
```

User đã trả tiền. Nhưng booking đã expire, inventory đã release (có thể đã bán cho người khác).

**Quyết định**: Auto-refund. KHÔNG cố confirm lại.

| Cố confirm lại | Auto-refund |
|----------------|-------------|
| Inventory có thể đã bán cho người khác → oversell | Tiền về user, không oversell |
| Phải re-reserve → có thể fail (hết chỗ) | Đơn giản, luôn an toàn |
| UX phức tạp ("đã expire rồi lại confirm") | UX rõ ràng: "thanh toán trễ, đã hoàn tiền, vui lòng đặt lại" |

**Liên kết Booking**: Booking EXPIRED là trạng thái terminal. Payment muộn → không revert booking state → chỉ refund tiền.

---

## 7. Thử thách 6: Gateway Down — Thanh toán không khả dụng

### Vấn đề

Payment gateway bảo trì hoặc quá tải vào ngày cao điểm (lễ lớn, event đặc biệt...).

### Giải pháp: Circuit Breaker + Tách Booking khỏi Payment

**Nguyên tắc**: Booking + inventory reserve là independent operation. Payment là bước tiếp theo.

```
1. Booking vẫn tạo được (PENDING), inventory vẫn reserve
2. Payment trả lỗi "Hệ thống thanh toán tạm thời gián đoạn"
3. Reservation TTL vẫn hoạt động → expire nếu quá hạn
4. User retry khi gateway recover
```

**Tại sao không chờ gateway recover rồi mới tạo booking?**
- User đã chọn vé, chọn ngày → reserve chỗ cho họ trước
- Payment fail thì retry được, còn inventory nếu không reserve → mất
- TTL đảm bảo không giữ chỗ mãi nếu user không quay lại

**Circuit breaker** ngăn cascade failure: nếu gateway đang down, đừng spam request → fail fast, báo user ngay.

**Multi-method fallback** (nếu gateway hỗ trợ nhiều payment method): credit card down → offer konbini/e-wallet. Mỗi method có API endpoint riêng, có thể 1 method down trong khi method khác vẫn hoạt động.

---

## 8. Thử thách 7: Refund — Hoàn tiền không đơn giản như tưởng

### Các loại refund

| Loại | Khi nào | Liên kết Booking |
|------|---------|-----------------|
| Full refund | User cancel toàn bộ | Booking → CANCELLED, release all inventory |
| Partial refund | User cancel 1 vé trong booking nhiều vé | Booking update quantity, release partial |
| Auto-refund | Late payment sau khi booking đã expire | Booking đã EXPIRED, inventory đã release |
| Admin refund | CS xử lý khiếu nại | Tùy case |

### Tại sao refund phải async?

Gateway refund thường chậm (credit card refund có thể mất vài ngày để về tài khoản, konbini refund cần xử lý riêng).

| Sync refund | Async refund |
|-------------|-------------|
| User chờ gateway response (2-10s) | User thấy "đang xử lý hoàn tiền" ngay |
| Gateway timeout → user không biết thành công chưa | Retry job handle failure |
| Block thread | Non-blocking |

**Flow**: Tạo refund record (PENDING) → publish event → background handler gọi gateway → update status. Retry 5 lần nếu fail.

### Validate refund amount

Không refund quá số đã trả. Đặc biệt với partial refund: cần SUM tất cả refund đã thành công cho payment đó, đảm bảo `total_refunded + new_refund ≤ original_amount`.

---

## 9. Thử thách 8: Amount Mismatch — Số tiền không khớp

### Vấn đề

| Nguyên nhân | Ví dụ |
|-------------|-------|
| Float arithmetic | `2000.0 * 3 = 5999.999999` thay vì `6000` |
| Gateway format | JPY không có decimal. Gửi sai format → charge sai |
| Price changed | Admin đổi giá giữa lúc user checkout |
| Tax rounding | 消費税 10%: `1999 * 1.10 = 2198.9` → `2199` hay `2198`? |

### Giải pháp: Verify ở 3 điểm

```
Point 1 (tạo payment):     Verify booking amount = SUM(items). Chặn bug tính giá
Point 2 (webhook callback): Verify amount gateway trả = amount trong payment record. Chặn tampering
Point 3 (reconciliation):   Cross-check toàn bộ. Chặn mọi thứ còn lại
```

**Nguyên tắc về kiểu dữ liệu**:
- JPY không có decimal → dùng integer/long (`2000円 = 2000`, không × 100)
- Tính thuế → dùng BigDecimal/Decimal, KHÔNG BAO GIỜ dùng float/double
- Với tiền, sai 1円 cũng không chấp nhận được

### Liên kết Booking

**Price snapshot**: Khi tạo booking, snapshot giá tại thời điểm đó. Payment dùng giá trong booking record, không query giá mới. Tránh "admin đổi giá giữa chừng".

---

## 10. Thử thách 9: Reconciliation — Đối soát hàng ngày

### Vấn đề

Cuối ngày, số tiền trong hệ thống phải khớp với số tiền gateway report. Nếu không khớp:
- Có payment thành công mà hệ thống không ghi nhận (missed callback)
- Có payment hệ thống ghi nhận nhưng gateway không có (ghost payment)
- Số tiền không khớp

### 3 Levels Reconciliation

```
Level 1 — Real-time (mỗi callback):
  Verify amount match giữa callback data và payment record
  → Reject ngay nếu mismatch

Level 2 — Near real-time (mỗi 2 phút):
  Polling job query gateway cho PENDING payments
  → Catch missed callbacks

Level 3 — Daily batch (03:00 AM):
  Download settlement report từ gateway
  Cross-check với internal payment records
  Generate discrepancy report
  Alert nếu có mismatch
```

### 3 loại discrepancy

| Loại | Nguyên nhân | Xử lý |
|------|-------------|-------|
| **Ghost payment** (internal có, gateway không) | Bug ghi nhận sai, hoặc gateway chưa settle | Verify lại với gateway API. Flag cho manual review |
| **Missed payment** (gateway có, internal không) | Callback miss + polling miss | Auto-confirm nếu booking còn PENDING. Nếu booking đã expire → auto-refund |
| **Amount mismatch** | Bug tính tiền, currency/format issue | Flag cho manual review. Không auto-fix |

### Liên kết Booking

- Missed payment phát hiện qua reconciliation → nếu booking vẫn PENDING → confirm, release reserved → booked
- Nếu booking đã EXPIRED → auto-refund (giống case late callback)
- **Reconciliation là safety net cuối cùng** — bắt mọi thứ mà real-time và near-real-time bỏ sót

---

## 11. Thử thách 10: Saga — Compensation chain khi payment fail

### Full Booking Saga

```
Step 1: Reserve Inventory          ← Compensation: Release Inventory
Step 2: Validate & Snapshot Price  ← (read-only, không cần compensation)
Step 3: Create Booking (PENDING)   ← Compensation: Cancel Booking
Step 4: Create Payment             ← Compensation: Void/Expire Payment
Step 5: Process Payment (gateway)  ← Compensation: Refund Payment
Step 6: Confirm Booking            ← (final step)
Step 7: Issue Ticket               ← Compensation: Void Ticket
```

### Payment fail → Compensation chain

Khi payment fail, hệ thống quyết định:

**Nếu còn retry attempts (< 3) VÀ TTL chưa hết:**
- Booking vẫn PENDING, inventory vẫn reserved
- User có thể chọn phương thức thanh toán khác
- UX: "Thanh toán thất bại. Thử lại?"

**Nếu hết retry attempts HOẶC TTL hết:**
- Compensation Step 4: Void payment
- Compensation Step 3: Cancel booking
- Compensation Step 1: Release inventory
- Publish BookingCancelled event

### Tại sao cho retry mà không cancel ngay?

| Cancel ngay | Cho retry |
|-------------|-----------|
| User phải đặt lại từ đầu | User chỉ cần chọn phương thức thanh toán khác |
| Inventory release → có thể bị người khác mua | Inventory vẫn reserved cho user |
| UX tệ | UX tốt |

**Nhưng vẫn có giới hạn**: max 3 attempts, và TTL vẫn chạy. Hết TTL → expire bất kể còn attempt hay không.

### Liên kết Booking: Ai quyết định?

- **Payment Service** chỉ thông báo: "payment fail" hoặc "payment success"
- **Booking Service** quyết định: retry, cancel, hay expire
- **Inventory Service** thực hiện: release hoặc confirm

Tách trách nhiệm rõ ràng → dễ test, dễ thay đổi logic mỗi service độc lập.

---

## 12. End-to-End: Booking → Payment → Confirm → Ticket

### Toàn bộ luồng

```
Phase 1: BROWSE (Read path — Redis)
═══════════════════════════════════════
1. User mở app, chọn ngày 1/5
2. GET /availability → Redis → "còn 135 chỗ, giá 2000円/vé" (< 1ms)

Phase 2: RESERVE (Write path — Redis + DB)
═══════════════════════════════════════
3. User chọn 5 vé, nhấn "Đặt vé"
4. POST /bookings {quantity: 5, expectedPrice: 2000}
5. Validate price (expectedPrice == currentPrice? → snapshot)
6. Redis Lua: pre-reserve (available -= 5)
7. DB optimistic lock: reserved += 5
8. Create booking (PENDING, expires_at = now + 15min)
9. Response: {bookingId, status: PENDING, expires_at}

Phase 3: PAYMENT
═══════════════════════════════════════
10. User nhấn "Thanh toán" → chọn payment method
11. POST /bookings/{id}/pay {method, idempotencyKey, token}
12. Idempotency check → create payment record (attempt 1)
13. Gateway: đăng ký giao dịch → thực hiện thanh toán
14. Payment → PENDING, lưu gateway transaction reference

Phase 4: CONFIRMATION
═══════════════════════════════════════
15. Gateway callback (webhook) / API response → thanh toán thành công
16. Verify bằng gateway query API ✓, verify amount ✓
17. Payment → SUCCESS
18. DB: reserved -= 5, booked += 5 (optimistic lock)
19. Redis Lua: confirm (reserved -= 5, booked += 5)
20. Booking → CONFIRMED
21. Publish BookingConfirmed event (outbox)

Phase 5: TICKETING (Event-driven)
═══════════════════════════════════════
22. Outbox publisher → publish BookingConfirmed
23. Ticketing service nhận event → sinh QR code / e-ticket
24. Notification service → gửi email/SMS confirm + vé

Phase 6: USAGE
═══════════════════════════════════════
25. User đến công viên, scan QR
26. Ticket → USED, Booking → USED
```

### Failure ở mỗi phase

| Phase | Failure | Xử lý | Liên kết |
|-------|---------|-------|----------|
| 2 (Reserve) | Redis reject (hết chỗ) | Trả "Sold out" ngay | Inventory |
| 2 (Reserve) | DB optimistic lock conflict | Rollback Redis, retry 3 lần | Inventory |
| 3 (Payment) | Gateway down | Circuit breaker, fallback method, "thử lại sau" | Payment |
| 3 (Payment) | User không thanh toán | TTL 15min → expire → release inventory | Booking + Inventory |
| 4 (Confirm) | Callback miss | Polling job query gateway mỗi 2 phút | Payment |
| 4 (Confirm) | Callback đến sau expire | Auto-refund, thông báo user đặt lại | Payment + Refund |
| 4 (Confirm) | Redis confirm fail | Reconciliation 30s fix | Inventory |
| 5 (Ticketing) | Ticketing service down | Event retry (outbox). Booking đã CONFIRMED, vé sinh sau | Event-driven |

---

## 13. Interview Quick Reference

### Elevator Pitch

> "Luồng thanh toán dùng **dual confirmation** — webhook callback là primary, gateway polling là fallback — đảm bảo không bao giờ có case 'tiền trừ mà vé không có'. Payment tách riêng khỏi Booking với state machine riêng, support retry, và reconciliation tự động 3 levels."

### Câu hỏi thường gặp

**"Sao tách Payment Service riêng?"**
> Booking chỉ cần biết "đã thanh toán chưa". Payment lo chi tiết gateway, retry, refund. Thêm payment method mới chỉ cần implement adapter, booking không đổi.

**"User nhấn thanh toán 2 lần?"**
> 3 layer: disable button (client), idempotency key (API), UNIQUE constraint (DB). Defense in depth — giống cách chống oversell.

**"Tiền trừ nhưng callback không đến?"**
> Dual confirmation: webhook callback + gateway polling mỗi 2 phút. Trước khi expire payment, luôn query gateway lần cuối. Worst case chậm 2 phút, không bao giờ mất tiền.

**"Callback đến sau khi booking đã expire?"**
> Auto-refund. Không cố confirm vì inventory có thể đã bán cho người khác.

**"Gateway down?"**
> Circuit breaker + multi-method fallback. Booking vẫn tạo (PENDING), inventory vẫn reserve. User retry khi recover. TTL đảm bảo không giữ chỗ mãi.

**"Reconciliation?"**
> 3 levels: real-time (verify mỗi callback), near-real-time (polling mỗi 2 phút), daily batch (03:00 AM cross-check settlement report).

**"Refund?"**
> Async. Tạo refund record → publish event → background handler gọi gateway → retry 5 lần. Tách table riêng vì lifecycle khác payment.

### Bảng quyết định tổng hợp

| Quyết định | Chọn | Tại sao |
|-----------|------|---------|
| Dual confirmation | Webhook + polling | Chỉ callback → miss thì user mất tiền |
| Payment tách Booking | Adapter pattern | Gộp → coupling. Thêm method phải sửa booking |
| Async refund | Event-driven | Sync → block user chờ gateway (chậm) |
| Query trước khi expire | Last-chance gateway query | Không query → có thể expire payment đã thành công |
| 3-layer idempotency | Client + API + DB | Mỗi layer chặn 1 loại duplicate khác nhau |
| Reconciliation 3 levels | Real-time + polling + daily | Mỗi level catch loại lỗi khác nhau |
| integer cho tiền | long (JPY) / BigDecimal (tax) | float có rounding error — sai 1円 cũng không được |
| Token-based | Server không thấy card | PCI DSS compliance — bắt buộc |
| Cho retry khi payment fail | Max 3, trong TTL | Cancel ngay → UX tệ, mất inventory đã reserve |
