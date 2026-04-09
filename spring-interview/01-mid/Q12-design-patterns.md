# Q12: Name some of the Design Patterns used in the Spring Framework?
> **Dịch:** Kể tên một số Design Pattern được sử dụng trong Spring Framework?

## Trả lời ngắn gọn
> Spring sử dụng nhiều design pattern: **Singleton, Factory, Proxy, Template Method, Observer, MVC, Front Controller, Dependency Injection**.

## Cách nhớ
```
Spring = "Bộ sưu tập" Design Pattern
Mỗi tính năng của Spring đều dựa trên 1 pattern nào đó
```

## Các Design Pattern trong Spring

### 1. Singleton Pattern
```java
// Mặc định, mỗi bean chỉ có 1 instance
@Service  // Mặc định là Singleton
public class UserService { }

// Container chỉ tạo 1 UserService duy nhất
// Mỗi chỗ inject đều dùng CÙNG 1 instance
```

### 2. Factory Pattern
```java
// BeanFactory / ApplicationContext = Factory
// Tạo bean dựa trên config, không cần biết cách tạo

ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
UserService service = ctx.getBean(UserService.class); // Factory tạo
```

### 3. Proxy Pattern (AOP)
```java
// Spring AOP tạo proxy bao quanh bean
@Service
public class UserService {
    @Transactional  // Spring tạo PROXY để quản lý transaction
    public void save(User user) {
        userRepo.save(user);
    }
}

// Thực tế: UserService$$Proxy (proxy) -> UserService (thật)
//          Proxy mở transaction -> gọi save() -> commit/rollback
```

### 4. Template Method Pattern
```java
// JdbcTemplate, RestTemplate, JmsTemplate...
// Định nghĩa "khung" xử lý, bạn chỉ điền logic

jdbcTemplate.query(
    "SELECT * FROM users",           // Bạn cung cấp SQL
    (rs, rowNum) -> new User(        // Bạn cung cấp mapping
        rs.getLong("id"),
        rs.getString("name")
    )
);
// JdbcTemplate lo phần còn lại: mở connection, đóng connection, xử lý lỗi
```

### 5. Observer Pattern (Event)
```java
// Spring Event System
// Publisher
@Service
public class OrderService {
    @Autowired
    private ApplicationEventPublisher publisher;

    public void createOrder(Order order) {
        orderRepo.save(order);
        publisher.publishEvent(new OrderCreatedEvent(order)); // Thông báo
    }
}

// Observer (Listener)
@Component
public class EmailListener {
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        emailService.sendConfirmation(event.getOrder()); // Lắng nghe & xử lý
    }
}
```

### 6. Front Controller Pattern (DispatcherServlet)
```
Mỗi request -> DispatcherServlet (1 nơi duy nhất) -> Route đến Controller phù hợp
```

### 7. MVC Pattern
```
Model      = Data (Entity, DTO)
View       = Hiển thị (Thymeleaf, JSP, JSON)
Controller = Xử lý request, điều phối
```

### 8. Dependency Injection (DI)
```java
@Service
public class OrderService {
    // Spring inject dependency, không phải tự new
    private final UserRepository userRepo;

    public OrderService(UserRepository userRepo) {
        this.userRepo = userRepo; // Injected by Spring
    }
}
```

## Bảng tóm tắt

| Pattern | Dùng ở đâu trong Spring |
|---------|------------------------|
| Singleton | Bean scope mặc định |
| Factory | BeanFactory, ApplicationContext |
| Proxy | AOP, @Transactional, @Async |
| Template Method | JdbcTemplate, RestTemplate |
| Observer | ApplicationEvent, @EventListener |
| Front Controller | DispatcherServlet |
| MVC | Spring MVC |
| DI | @Autowired, Constructor Injection |

## Điểm quan trọng nhớ phỏng vấn
1. **Singleton**: Mặc định, 1 instance / container
2. **Proxy**: AOP, Transaction - tạo proxy tự động
3. **Template Method**: XxxTemplate classes
4. Hiểu pattern giúp hiểu **tại sao** Spring thiết kế như vậy
