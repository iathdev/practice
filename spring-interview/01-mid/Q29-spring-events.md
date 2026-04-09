# Q29: Describe some of the standard Spring events
> **Dịch:** Mô tả một số sự kiện (event) tiêu chuẩn của Spring

## Trả lời ngắn gọn
> Spring cung cấp các **built-in event** để thông báo về trạng thái của ApplicationContext: **ContextRefreshedEvent, ContextStartedEvent, ContextStoppedEvent, ContextClosedEvent, RequestHandledEvent**.

## Cách nhớ
```
Spring Events giống như Thông báo của nhà trường:
- Refreshed  = "Trường đã sẵn sàng!" (khởi động xong)
- Started    = "Bắt đầu học!"
- Stopped    = "Nghỉ giải lao!"
- Closed     = "Hết giờ, về nhà!"
```

## Các Standard Events

| Event | Khi nào | Mô tả |
|-------|---------|-------|
| **ContextRefreshedEvent** | Context khởi tạo hoặc refresh xong | Tất cả bean đã sẵn sàng |
| **ContextStartedEvent** | Gọi `context.start()` | Context được start |
| **ContextStoppedEvent** | Gọi `context.stop()` | Context bị stop |
| **ContextClosedEvent** | Gọi `context.close()` | Context bị đóng (destroy bean) |
| **RequestHandledEvent** | HTTP request xử lý xong | Chỉ trong web app |
| **ServletRequestHandledEvent** | Servlet request xử lý xong | Thêm thông tin servlet |

## Ví dụ: Lắng nghe Standard Events

```java
@Component
public class AppEventListener {

    @EventListener
    public void onContextRefreshed(ContextRefreshedEvent event) {
        System.out.println("Context đã khởi tạo xong!");
        System.out.println("Thời điểm: " + event.getTimestamp());
        // Dùng để: load cache, warm up, kiểm tra config
    }

    @EventListener
    public void onContextClosed(ContextClosedEvent event) {
        System.out.println("Context đang đóng...");
        // Dùng để: giải phóng resource, lưu state
    }
}
```

## Custom Event (tự tạo)

```java
// 1. Định nghĩa Event
public class OrderCreatedEvent {
    private final Order order;
    public OrderCreatedEvent(Order order) { this.order = order; }
    public Order getOrder() { return order; }
}

// 2. Publish event
@Service
public class OrderService {
    @Autowired
    private ApplicationEventPublisher publisher;

    public void createOrder(Order order) {
        orderRepo.save(order);
        publisher.publishEvent(new OrderCreatedEvent(order));
    }
}

// 3. Listen event
@Component
public class NotificationListener {
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        emailService.sendOrderConfirmation(event.getOrder());
    }
}

// 4. Async listener (chạy trên thread khác)
@Component
public class AnalyticsListener {
    @Async
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        analyticsService.track("order_created", event.getOrder());
    }
}
```

## Điểm quan trọng nhớ phỏng vấn
1. **ContextRefreshedEvent** = Context đã sẵn sàng (thường dùng nhất)
2. Dùng `@EventListener` để lắng nghe (thay cho implement ApplicationListener)
3. Custom event **không cần** extend ApplicationEvent (từ Spring 4.2)
4. `@Async` + `@EventListener` = xử lý **bất đồng bộ**
