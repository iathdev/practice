# Project Structure — Tickets Platform

## Architecture: Hexagonal + DDD + Clean Architecture

### Tech Stack

| Layer       | Tech                                          |
|-------------|-----------------------------------------------|
| Framework   | Spring Boot 3.x (Java 21+)                   |
| Build       | Maven (multi-module)                          |
| Persistence | Spring Data JPA + Hibernate                   |
| Messaging   | Spring Events (internal) + RabbitMQ (external)|
| API         | Spring MVC (REST)                             |
| DB          | PostgreSQL                                    |
| Migration   | Flyway                                        |

---

## Bounded Contexts / Modules

```
                    ┌──────────┐
         ┌────────>│ Catalog  │<────────┐
         │         └──────────┘         │
         │      (công viên, loại vé)    │
         │                              │
    ┌────┴─────┐                  ┌─────┴────┐
    │ Pricing  │                  │Inventory │
    │(tính giá)│                  │(sức chứa)│
    └────┬─────┘                  └─────┬────┘
         │                              │
         │         ┌──────────┐         │
         └────────>│ Booking  │<────────┘
                   │ (đặt vé) │
                   └────┬─────┘
                        │
                   ┌────▼─────┐
                   │ Payment  │
                   │(thanh toán)
                   └────┬─────┘
                        │
                   ┌────▼─────┐
                   │Ticketing │
                   │(xuất vé) │
                   └──────────┘

              ┌──────────┐
              │ Identity │  (xuyên suốt)
              └──────────┘
```

### 1. Catalog — "Bán cái gì?"

- Quản lý thông tin công viên: tên, địa chỉ, mô tả, hình ảnh, lịch hoạt động
- Quản lý loại vé: người lớn, trẻ em, VIP, combo, vé theo zone...
- Lịch mở cửa, ngày lễ đóng cửa

### 2. Inventory — "Còn chỗ không?"

- Mỗi công viên có capacity tối đa / ngày
- Theo dõi số vé đã bán, đã reserve, còn lại
- Có thể giới hạn theo zone hoặc time slot
- **Tại sao tách khỏi Catalog?** Catalog là thông tin tĩnh (ít thay đổi), Inventory là trạng thái động (thay đổi liên tục). Tách ra để scale và xử lý concurrency độc lập.

### 3. Pricing — "Giá bao nhiêu?"

- Giá cơ bản theo loại vé
- Khuyến mãi: early bird, combo, mã giảm giá
- Dynamic pricing: cuối tuần, lễ tết đắt hơn
- Giá theo nhóm: mua 10+ vé giảm 15%
- **Tại sao tách khỏi Catalog?** Logic tính giá phức tạp và thay đổi thường xuyên (campaign, flash sale...).

### 4. Booking — "Đặt vé" (Core Domain)

- Module trung tâm, điều phối luồng đặt vé
- Khách chọn công viên + ngày + loại vé + số lượng
- Kiểm tra inventory, tính giá, tạo booking
- Quản lý vòng đời: PENDING → CONFIRMED → USED / CANCELLED

### 5. Payment — "Thanh toán"

- Lắng nghe BookingCreated → tạo payment session
- Tích hợp cổng thanh toán: VNPay, Momo, Stripe...
- Xử lý callback từ payment gateway
- Phát event PaymentCompleted hoặc PaymentFailed
- **Tại sao tách khỏi Booking?** Booking quan tâm "đặt cái gì", Payment quan tâm "trả bằng gì, trả như nào".

### 6. Ticketing — "Xuất vé"

- Lắng nghe PaymentCompleted → sinh vé
- Tạo QR code cho mỗi vé
- Gửi vé qua email / app
- Xác thực vé tại cổng vào (scan QR)
- **Tại sao tách khỏi Booking?** Vé có vòng đời riêng (sinh ra, gửi, scan, hết hạn). Logic QR, chống giả, xác thực là domain riêng biệt.

### 7. Identity — "Ai đang dùng?"

- Đăng ký, đăng nhập (email, social login)
- Phân quyền: khách, nhân viên công viên, admin
- Profile khách hàng

---

## Maven Multi-Module Layout

```
tickets-platform/
├── pom.xml                               # Parent POM
│
├── shared-kernel/
│   ├── pom.xml
│   └── src/main/java/com/tickets/shared/
│       ├── domain/
│       │   ├── AggregateRoot.java
│       │   ├── Entity.java
│       │   ├── ValueObject.java
│       │   ├── DomainEvent.java
│       │   ├── DomainException.java
│       │   └── Money.java
│       └── event/
│           ├── BookingCreated.java
│           ├── PaymentCompleted.java
│           ├── PaymentFailed.java
│           └── BookingCancelled.java
│
├── booking/
│   ├── pom.xml
│   └── src/main/java/com/tickets/booking/
│       ├── domain/
│       │   ├── model/
│       │   │   ├── Booking.java              # Aggregate Root
│       │   │   ├── BookingId.java            # Value Object
│       │   │   ├── BookingItem.java          # Entity
│       │   │   ├── BookingStatus.java        # Enum VO
│       │   │   ├── VisitDate.java            # Value Object
│       │   │   └── GuestInfo.java            # Value Object
│       │   ├── event/
│       │   │   ├── BookingCreated.java
│       │   │   ├── BookingConfirmed.java
│       │   │   └── BookingCancelled.java
│       │   ├── port/
│       │   │   ├── in/
│       │   │   │   └── BookingUseCase.java
│       │   │   └── out/
│       │   │       ├── BookingRepository.java
│       │   │       ├── InventoryChecker.java
│       │   │       └── PricingService.java
│       │   ├── service/
│       │   │   └── BookingDomainService.java
│       │   └── exception/
│       │       ├── InsufficientCapacityException.java
│       │       └── InvalidBookingStateException.java
│       │
│       ├── application/
│       │   ├── command/
│       │   │   ├── CreateBookingCommand.java
│       │   │   ├── CreateBookingCommandHandler.java
│       │   │   ├── ConfirmBookingCommand.java
│       │   │   ├── ConfirmBookingCommandHandler.java
│       │   │   ├── CancelBookingCommand.java
│       │   │   └── CancelBookingCommandHandler.java
│       │   ├── query/
│       │   │   ├── GetBookingQuery.java
│       │   │   ├── GetBookingQueryHandler.java
│       │   │   ├── ListUserBookingsQuery.java
│       │   │   ├── ListUserBookingsQueryHandler.java
│       │   │   └── BookingReadModel.java
│       │   ├── dto/
│       │   │   ├── CreateBookingRequest.java
│       │   │   └── BookingResponse.java
│       │   └── eventhandler/
│       │       └── PaymentCompletedHandler.java
│       │
│       └── infrastructure/
│           ├── persistence/
│           │   ├── JpaBookingRepository.java
│           │   ├── BookingJpaEntity.java
│           │   ├── BookingItemJpaEntity.java
│           │   └── BookingMapper.java
│           ├── web/
│           │   ├── BookingController.java
│           │   └── BookingExceptionHandler.java
│           ├── messaging/
│           │   └── SpringBookingEventPublisher.java
│           ├── external/
│           │   ├── HttpInventoryChecker.java
│           │   └── HttpPricingService.java
│           └── config/
│               └── BookingModuleConfig.java
│
├── catalog/
│   ├── pom.xml
│   └── src/main/java/com/tickets/catalog/
│       ├── domain/
│       │   ├── model/
│       │   │   ├── Park.java
│       │   │   ├── ParkId.java
│       │   │   ├── TicketType.java
│       │   │   ├── TicketTypeId.java
│       │   │   ├── PriceTag.java
│       │   │   └── OperatingSchedule.java
│       │   └── port/
│       │       ├── in/
│       │       │   └── CatalogQueryUseCase.java
│       │       └── out/
│       │           └── ParkRepository.java
│       ├── application/
│       └── infrastructure/
│
├── payment/
│   ├── pom.xml
│   └── src/main/java/com/tickets/payment/
│       ├── domain/
│       │   ├── model/
│       │   │   ├── Payment.java
│       │   │   ├── PaymentId.java
│       │   │   ├── Money.java
│       │   │   ├── PaymentMethod.java
│       │   │   └── PaymentStatus.java
│       │   ├── port/out/
│       │   │   ├── PaymentRepository.java
│       │   │   └── PaymentProvider.java
│       │   └── event/
│       │       ├── PaymentCompleted.java
│       │       └── PaymentFailed.java
│       ├── application/
│       └── infrastructure/
│           └── provider/
│               ├── StripePaymentProvider.java
│               └── VnPayPaymentProvider.java
│
├── ticketing/
│   ├── pom.xml
│   └── src/main/java/com/tickets/ticketing/
│       ├── domain/
│       │   ├── model/
│       │   │   ├── Ticket.java
│       │   │   ├── QrCode.java
│       │   │   └── TicketValidation.java
│       │   └── port/out/
│       │       └── QrCodeGenerator.java
│       ├── application/
│       └── infrastructure/
│
├── inventory/
│   ├── pom.xml
│   └── src/main/java/com/tickets/inventory/
│       ├── domain/
│       │   ├── model/
│       │   │   ├── DailyCapacity.java
│       │   │   ├── Slot.java
│       │   │   └── Reservation.java
│       │   └── event/
│       │       └── CapacityExhausted.java
│       ├── application/
│       └── infrastructure/
│
├── pricing/
│   ├── pom.xml
│   └── src/main/java/com/tickets/pricing/
│       ├── domain/
│       │   ├── model/
│       │   │   ├── PricingRule.java
│       │   │   ├── Promotion.java
│       │   │   └── Discount.java
│       │   └── service/
│       │       └── PriceCalculator.java
│       ├── application/
│       └── infrastructure/
│
├── identity/
│   ├── pom.xml
│   └── src/main/java/com/tickets/identity/
│       ├── domain/
│       ├── application/
│       └── infrastructure/
│
└── bootstrap/
    ├── pom.xml
    └── src/main/
        ├── java/com/tickets/TicketsApplication.java
        └── resources/
            ├── application.yml
            ├── application-dev.yml
            └── db/migration/
                ├── V1__create_catalog.sql
                ├── V2__create_booking.sql
                ├── V3__create_payment.sql
                └── V4__create_inventory.sql
```

---

## Dependency Rule

```
Infrastructure  ──depends on──>  Application  ──depends on──>  Domain
    (outer)                        (middle)                    (inner, pure Java)
```

- **Domain**: Pure Java, không import Spring, không import JPA. Chỉ phụ thuộc shared-kernel.
- **Application**: Phụ thuộc Domain. Dùng Spring @Service, @Transactional.
- **Infrastructure**: Triển khai các Port. Phụ thuộc Spring Data, JPA, HTTP client...

---

## Luồng tổng thể khi khách đặt vé

```
Khách chọn vé
    │
    ▼
[Booking] ──check──> [Inventory] ✓ còn chỗ
    │
    │ ──calc───> [Pricing] → 240.000đ
    │
    │ tạo Booking(PENDING)
    │ phát BookingCreated
    ▼
[Payment] nhận event → tạo payment → redirect VNPay
    │
    │ VNPay callback → PaymentCompleted
    ▼
[Ticketing] nhận event → sinh QR → gửi email
    │
[Inventory] nhận event → trừ capacity chính thức
    │
[Booking] nhận event → status = CONFIRMED
```
