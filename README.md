# Practice — System Design & Interview Preparation

Project phục vụ mục đích **học tập** và **chuẩn bị phỏng vấn** thông qua việc thiết kế và implement các hệ thống thực tế, tập trung vào những bài toán phổ biến trong phỏng vấn Backend/System Design.

## Bài toán chính

### 1. Tickets Platform — Hệ thống đặt vé công viên (Java / Spring Boot)

Hệ thống đặt vé công viên giải trí tham khảo theo hướng Disney Parks, sử dụng kiến trúc **Hexagonal + DDD + Clean Architecture**.

**Các bài toán được giải quyết:**

| Bài toán | Giải pháp | Keyword phỏng vấn |
|----------|-----------|-------------------|
| Race condition & Overselling | Redis Pre-check + DB Optimistic Lock + CHECK constraint | Defense in Depth |
| Cache Invalidation (Redis drift) | Write-through + Reconciliation 30s | Eventual Consistency |
| Reservation TTL (hold hết hạn) | Scheduled Job + `SKIP LOCKED` | Two-Phase Reservation |
| Hot Slot (10K người tranh 1 ngày) | Inventory Sharding | Reduce Contention |
| Thundering Herd (cache miss đồng loạt) | Singleflight pattern | Local Dedup |
| Tiền trừ nhưng vé không có | Dual Confirmation (webhook + polling) | Belt and Suspenders |
| Double Payment | 3-layer idempotency (client + API + DB) | Defense in Depth |
| Event mất giữa chừng | Transactional Outbox + polling publisher | At-least-once Delivery |
| Payment fail | Saga Pattern + compensation chain | Choreography |
| Giá thay đổi giữa chừng | Price Snapshot + version check | Optimistic Concurrency |
| Redis down | Circuit breaker + fallback pessimistic lock | Graceful Degradation |

**Tech Stack:** Spring Boot 3.x, Java 21+, PostgreSQL, Redis (Lua script), RabbitMQ, Flyway

### 2. Anti-Cheat Service — Phát hiện gian lận thi online (Golang)

Service backend phục vụ nền tảng thi tiếng Anh online (IELTS/TOEIC), phát hiện gian lận real-time và hậu kỳ. Đáp ứng 5,000-10,000 thí sinh thi đồng thời.

**Các bài toán được giải quyết:**

| Bài toán | Giải pháp | Keyword phỏng vấn |
|----------|-----------|-------------------|
| High-throughput event ingestion | Redis Event Buffer + batch flush 60s | Write Buffering |
| Real-time detection | Redis counters + threshold rules | Stream Processing |
| Complex pattern analysis | TimescaleDB + continuous aggregates | Time-series Analysis |
| Lưu trữ & audit trail | TimescaleDB compression + retention policy | Data Lifecycle |
| Cross-exam cheating | Answer similarity analysis (batch) | Batch Processing |
| Scale 5K-10K concurrent | Buffer decouple write, batch INSERT | Backpressure |

**Tech Stack:** Golang, Redis, PostgreSQL + TimescaleDB

## Cấu trúc tài liệu

```
docs/
├── 01-project-structure.md          # Kiến trúc tổng thể Tickets Platform
├── 02-anti-cheating-service.md       # Thiết kế Anti-Cheat Service
├── interview/
│   └── 06-interview-disney-approach.md  # Cheat sheet phỏng vấn
└── research/
    ├── anti-cheat/
    │   └── prompt.md                 # Yêu cầu Anti-Cheat Service
    └── ticket/
        ├── 01-booking-inventory-core.md       # Inventory model, race condition, concurrency
        ├── 02-booking-payment-integration.md  # Saga, Outbox, end-to-end flow
        └── 03-payment-challenges.md           # 10 thử thách thiết kế payment
```

## Mục tiêu học tập

- Hiểu sâu các **design pattern** trong distributed system: Saga, Outbox, Circuit Breaker, Singleflight
- Nắm vững **concurrency control**: Optimistic/Pessimistic Locking, Atomic operations, Lua script
- Thực hành **DDD + Hexagonal Architecture** với bounded contexts thực tế
- Chuẩn bị trả lời **system design interview** với các trade-off rõ ràng
- Xây dựng tư duy **defense in depth** — không tin bất kỳ single layer nào
