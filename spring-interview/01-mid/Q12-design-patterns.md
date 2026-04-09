# Q12: Name some of the Design Patterns used in the Spring Framework?
> **Dịch:** Kể tên một số Design Pattern được sử dụng trong Spring Framework?

## Tra loi ngan gon
> Spring su dung nhieu design pattern: **Singleton, Factory, Proxy, Template Method, Observer, MVC, Front Controller, Dependency Injection**.

## Cach nho
```
Spring = "Bo suu tap" Design Pattern
Moi tinh nang cua Spring deu dua tren 1 pattern nao do
```

## Cac Design Pattern trong Spring

### 1. Singleton Pattern
```java
// Mac dinh, moi bean chi co 1 instance
@Service  // Mac dinh la Singleton
public class UserService { }

// Container chi tao 1 UserService duy nhat
// Moi cho inject deu dung CUNG 1 instance
```

### 2. Factory Pattern
```java
// BeanFactory / ApplicationContext = Factory
// Tao bean dua tren config, khong can biet cach tao

ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
UserService service = ctx.getBean(UserService.class); // Factory tao
```

### 3. Proxy Pattern (AOP)
```java
// Spring AOP tao proxy bao quanh bean
@Service
public class UserService {
    @Transactional  // Spring tao PROXY de quan ly transaction
    public void save(User user) {
        userRepo.save(user);
    }
}

// Thuc te: UserService$$Proxy (proxy) -> UserService (that)
//          Proxy mo transaction -> goi save() -> commit/rollback
```

### 4. Template Method Pattern
```java
// JdbcTemplate, RestTemplate, JmsTemplate...
// Dinh nghia "khung" xu ly, ban chi dien logic

jdbcTemplate.query(
    "SELECT * FROM users",           // Ban cung cap SQL
    (rs, rowNum) -> new User(        // Ban cung cap mapping
        rs.getLong("id"),
        rs.getString("name")
    )
);
// JdbcTemplate lo phan con lai: mo connection, dong connection, xu ly loi
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
        publisher.publishEvent(new OrderCreatedEvent(order)); // Thong bao
    }
}

// Observer (Listener)
@Component
public class EmailListener {
    @EventListener
    public void onOrderCreated(OrderCreatedEvent event) {
        emailService.sendConfirmation(event.getOrder()); // Lang nghe & xu ly
    }
}
```

### 6. Front Controller Pattern (DispatcherServlet)
```
Moi request -> DispatcherServlet (1 noi duy nhat) -> Route den Controller phu hop
```

### 7. MVC Pattern
```
Model      = Data (Entity, DTO)
View       = Hien thi (Thymeleaf, JSP, JSON)
Controller = Xu ly request, dieu phoi
```

### 8. Dependency Injection (DI)
```java
@Service
public class OrderService {
    // Spring inject dependency, khong phai tu new
    private final UserRepository userRepo;

    public OrderService(UserRepository userRepo) {
        this.userRepo = userRepo; // Injected by Spring
    }
}
```

## Bang tom tat

| Pattern | Dung o dau trong Spring |
|---------|------------------------|
| Singleton | Bean scope mac dinh |
| Factory | BeanFactory, ApplicationContext |
| Proxy | AOP, @Transactional, @Async |
| Template Method | JdbcTemplate, RestTemplate |
| Observer | ApplicationEvent, @EventListener |
| Front Controller | DispatcherServlet |
| MVC | Spring MVC |
| DI | @Autowired, Constructor Injection |

## Diem quan trong nho phong van
1. **Singleton**: Mac dinh, 1 instance / container
2. **Proxy**: AOP, Transaction - tao proxy tu dong
3. **Template Method**: XxxTemplate classes
4. Hieu pattern giup hieu **tai sao** Spring thiet ke nhu vay
