# Ticket Booking System — Keywords cần nhớ cho phỏng vấn

> Tổng hợp từ 3 file research: booking-inventory-core, booking-payment-integration, payment-challenges.

---

## 1. Inventory & Concurrency

| Keyword | Giải thích ngắn | Khi nào nhắc đến |
|---------|-----------------|-------------------|
| **total_capacity / reserved / booked** | Tách 3 field, available = total - reserved - booked (computed) | Hỏi về inventory model |
| **CHECK constraint** | `reserved + booked <= total_capacity` — safety net ở DB level | Hỏi chống oversell |
| **Optimistic Locking** | Version column, retry khi conflict. Throughput cao hơn pessimistic | Hỏi concurrency control |
| **Pessimistic Locking** | SELECT FOR UPDATE — serialize request. Đơn giản nhưng throughput thấp | Hỏi trade-off concurrency |
| **Atomic SQL** | 1 câu UPDATE với WHERE condition — DB đảm bảo atomic | Hỏi cách đơn giản nhất |
| **Redis Pre-check + DB Confirm** | Redis Lua reject sớm (100K ops/s), DB là source of truth | Hỏi high-traffic solution |
| **Lua Script** | Atomic check + decrement trên Redis, không bị race condition | Hỏi tại sao dùng Lua |
| **Reconciliation** | Sync Redis ↔ DB mỗi 30s, sửa drift | Hỏi Redis ↔ DB consistency |
| **Circuit Breaker** | Redis down → fallback thẳng DB (pessimistic lock) | Hỏi Redis failure handling |

## 2. Reservation & State Machine

| Keyword | Giải thích ngắn | Khi nào nhắc đến |
|---------|-----------------|-------------------|
| **Two-Phase Reservation** | Phase 1: Reserve (hold tạm) → Phase 2: Confirm hoặc Release | Hỏi booking flow |
| **TTL (Time-To-Live)** | Booking PENDING có expires_at (10-15 phút). Hết → release inventory | Hỏi giữ chỗ tạm |
| **Scheduled Job + Delay Queue** | Job scan expired mỗi 15-30s (safety net) + delay queue (precision) | Hỏi cách expire booking |
| **State Machine** | PENDING → CONFIRMED → USED / CANCELLED / EXPIRED | Hỏi booking lifecycle |
| **Terminal State** | EXPIRED, USED — không revert | Hỏi state transition rules |
| **Price Snapshot** | Lưu giá vào booking tại thời điểm đặt, không query giá mới khi thanh toán | Hỏi price consistency |
| **Price Version** | Client gửi expectedPrice + priceVersion, server reject nếu không khớp | Hỏi giá thay đổi giữa chừng |

## 3. Idempotency & Duplicate Prevention

| Keyword | Giải thích ngắn | Khi nào nhắc đến |
|---------|-----------------|-------------------|
| **Defense in Depth (3 Layer)** | Client disable button + API idempotency key + DB UNIQUE constraint | Hỏi chống duplicate |
| **Idempotency Key** | Client generate UUID, gửi header. Server trả response cũ nếu key trùng | Hỏi API design |
| **UNIQUE Constraint** | DB level — chặn mọi race condition dù code có bug | Hỏi tại sao cần cả 3 layer |

## 4. Payment Flow

| Keyword | Giải thích ngắn | Khi nào nhắc đến |
|---------|-----------------|-------------------|
| **Token-based Payment** | Client gửi card trực tiếp cho gateway → nhận token → server charge bằng token | Hỏi security / PCI DSS |
| **PCI DSS** | Server không thấy card number → giảm compliance scope | Hỏi tại sao dùng token |
| **Payment State Machine** | CREATED → PENDING → SUCCESS / FAILED / EXPIRED → REFUNDED | Hỏi payment lifecycle |
| **1 Booking → N Payment Attempts** | Fail lần 1 → tạo attempt 2 (method khác). Max 3 attempts trong TTL | Hỏi retry strategy |
| **Gateway Adapter Pattern** | Abstract gateway qua interface → thêm method mới chỉ implement adapter | Hỏi extensibility |
| **ResultURL vs NotifyURL** | ResultURL = client redirect (không tin). NotifyURL = server webhook (tin) | Hỏi callback handling |

## 5. Dual Confirmation — Case nguy hiểm nhất

| Keyword | Giải thích ngắn | Khi nào nhắc đến |
|---------|-----------------|-------------------|
| **"Tiền trừ nhưng vé không có"** | Callback miss → payment PENDING → expire → user mất tiền | Hỏi edge case nguy hiểm |
| **Dual Confirmation** | Webhook (primary) + Polling job mỗi 2 phút (fallback) | Hỏi giải pháp |
| **Last-chance Query** | Query gateway LẦN CUỐI trước khi expire payment | Hỏi tại sao cần polling |
| **Late Callback → Auto-refund** | Callback đến sau khi booking EXPIRED → refund, KHÔNG confirm lại | Hỏi callback đến trễ |
| **Immediate Poll on ResultURL** | User redirect về → trigger poll ngay (không chờ interval) | Hỏi optimize latency |

## 6. Saga & Event-Driven

| Keyword | Giải thích ngắn | Khi nào nhắc đến |
|---------|-----------------|-------------------|
| **Saga Pattern** | Multi-step transaction, mỗi step có compensation | Hỏi distributed transaction |
| **Compensation Chain** | Fail step N → undo step N-1, N-2... (reverse order) | Hỏi rollback strategy |
| **Choreography vs Orchestration** | Choreography: mỗi service tự quyết (≤4 steps). Orchestration: central coordinator (>4 steps) | Hỏi saga implementation |
| **Outbox Pattern** | Lưu event vào DB cùng transaction → background job publish → at-least-once delivery | Hỏi event reliability |
| **Transactional Outbox** | Business logic + event insert trong cùng 1 COMMIT → không mất event | Hỏi tại sao không publish trực tiếp |
| **Domain Events** | BookingCreated, PaymentCompleted, PaymentFailed, BookingCancelled | Hỏi module communication |
| **Port + Adapter (ACL)** | Booking không import Inventory domain model, chỉ biết interface | Hỏi module boundary |

## 7. Refund & Reconciliation

| Keyword | Giải thích ngắn | Khi nào nhắc đến |
|---------|-----------------|-------------------|
| **Async Refund** | Tạo record PENDING → event → background handler gọi gateway → retry 5 lần | Hỏi refund flow |
| **Partial Refund** | Cancel 1 vé trong booking nhiều vé. SUM(refunded) + new ≤ original | Hỏi refund edge case |
| **3-Level Reconciliation** | Real-time (mỗi callback) + Near-real-time (polling 2 phút) + Daily batch (03:00 AM) | Hỏi đối soát |
| **Ghost Payment** | Internal có, gateway không → manual review | Hỏi discrepancy types |
| **Missed Payment** | Gateway có, internal không → auto-confirm hoặc auto-refund | Hỏi discrepancy handling |

## 8. Data & Architecture

| Keyword | Giải thích ngắn | Khi nào nhắc đến |
|---------|-----------------|-------------------|
| **BigDecimal / BIGINT** | Tiền dùng integer (JPY). Thuế dùng BigDecimal. KHÔNG BAO GIỜ float | Hỏi money handling |
| **DDD (Domain-Driven Design)** | Tách bounded context: Booking, Payment, Inventory, Ticketing, Catalog | Hỏi architecture |
| **Hexagonal Architecture** | Domain không phụ thuộc infrastructure. Đổi gateway → chỉ đổi adapter | Hỏi code structure |
| **Event-Driven Architecture** | Modules giao tiếp qua event (loosely coupled), outbox đảm bảo delivery | Hỏi communication pattern |

---

## Quick Cheat Sheet — 10 câu phải thuộc

1. **Inventory model?** → `total_capacity / reserved / booked` + CHECK constraint. Available là computed.

2. **Chống oversell?** → Redis pre-check (reject sớm) + DB optimistic lock (source of truth) + CHECK constraint (safety net). Reconciliation 30s.

3. **TTL reservation?** → PENDING + expires_at 15 phút. Scheduled job + delay queue. Query gateway trước khi expire.

4. **Booking ↔ Payment?** → Tách hoàn toàn qua domain event. Payment chỉ báo success/fail. Booking quyết định confirm/retry/expire.

5. **Chống duplicate?** → Defense in depth 3 layer: client button + idempotency key + DB UNIQUE.

6. **Tiền trừ vé không có?** → Dual confirmation: webhook + polling 2 phút. Last-chance query trước expire.

7. **Callback đến trễ?** → Auto-refund. KHÔNG confirm lại vì inventory có thể đã bán.

8. **Event không mất?** → Transactional Outbox: event + business logic cùng COMMIT. Consumer phải idempotent.

9. **Payment fail?** → Cho retry (max 3, trong TTL). Hết retry/TTL → compensation: void payment → cancel booking → release inventory.

10. **Reconciliation?** → 3 levels: real-time + near-real-time (polling) + daily batch. Safety net cuối cùng.
