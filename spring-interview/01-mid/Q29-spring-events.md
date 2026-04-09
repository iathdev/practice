# Q29: Describe some of the standard Spring events
> **Dịch:** Mô tả một số sự kiện (event) tiêu chuẩn của Spring

## Tra loi ngan gon
> Spring cung cap cac **built-in event** de thong bao ve trang thai cua ApplicationContext: **ContextRefreshedEvent, ContextStartedEvent, ContextStoppedEvent, ContextClosedEvent, RequestHandledEvent**.

## Cach nho
```
Spring Events giong nhu Thong bao cua nha truong:
- Refreshed  = "Truong da san sang!" (khoi dong xong)
- Started    = "Bat dau hoc!"
- Stopped    = "Nghi giai lao!"
- Closed     = "Het gio, ve nha!"
```

## Cac Standard Events

| Event | Khi nao | Mo ta |
|-------|---------|-------|
| **ContextRefreshedEvent** | Context khoi tao hoac refresh xong | Tat ca bean da san sang |
| **ContextStartedEvent** | Goi `context.start()` | Context duoc start |
| **ContextStoppedEvent** | Goi `context.stop()` | Context bi stop |
| **ContextClosedEvent** | Goi `context.close()` | Context bi dong (destroy bean) |
| **RequestHandledEvent** | HTTP request xu ly xong | Chi trong web app |
| **ServletRequestHandledEvent** | Servlet request xu ly xong | Them thong tin servlet |

## Vi du: Lang nghe Standard Events

```java
@Component
public class AppEventListener {

    @EventListener
    public void onContextRefreshed(ContextRefreshedEvent event) {
        System.out.println("Context da khoi tao xong!");
        System.out.println("Thoi diem: " + event.getTimestamp());
        // Dung de: load cache, warm up, kiem tra config
    }

    @EventListener
    public void onContextClosed(ContextClosedEvent event) {
        System.out.println("Context dang dong...");
        // Dung de: giai phong resource, luu state
    }
}
```

## Custom Event (tu tao)

```java
// 1. Dinh nghia Event
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

// 4. Async listener (chay tren thread khac)
@Component
public class AnalyticsListener {
    @Async
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        analyticsService.track("order_created", event.getOrder());
    }
}
```

## Diem quan trong nho phong van
1. **ContextRefreshedEvent** = Context da san sang (thuong dung nhat)
2. Dung `@EventListener` de lang nghe (thay cho implement ApplicationListener)
3. Custom event **khong can** extend ApplicationEvent (tu Spring 4.2)
4. `@Async` + `@EventListener` = xu ly **bat dong bo**
