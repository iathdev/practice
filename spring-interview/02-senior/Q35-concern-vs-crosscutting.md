# Q35: What is the difference between concern and cross-cutting concern in Spring AOP?
> **Dịch:** Sự khác nhau giữa Concern và Cross-cutting Concern trong Spring AOP là gì?

## Trả lời ngắn gọn
> **Concern** = chức năng cụ thể của 1 module (vd: xử lý order). **Cross-cutting concern** = chức năng **cắt ngang** nhiều module (vd: logging, security, transaction). AOP xử lý cross-cutting concern.

## Cách nhớ
```
Concern = Môn học (Toán, Văn, Anh) - mỗi môn riêng biệt
Cross-cutting = Điểm danh - áp dụng cho TẤT CẢ môn học
```

## Minh họa

```
                  UserService    OrderService    PaymentService
                  +-----------+  +------------+  +-----------+
CONCERN           | findUser  |  | createOrder|  | charge    |
(business logic)  | updateUser|  | cancelOrder|  | refund    |
                  +-----------+  +------------+  +-----------+
                       |              |               |
CROSS-CUTTING  ========|==============|===============|========  Logging
CONCERN        ========|==============|===============|========  Security
               ========|==============|===============|========  Transaction
               (cắt ngang TẤT CẢ services)
```

## Ví dụ cụ thể

### Concern (business logic riêng)
```java
@Service
public class OrderService {
    // Concern: chỉ xử lý ORDER
    public Order createOrder(OrderDto dto) { /* ... */ }
    public void cancelOrder(Long id) { /* ... */ }
}

@Service
public class UserService {
    // Concern: chỉ xử lý USER
    public User register(UserDto dto) { /* ... */ }
    public User findById(Long id) { /* ... */ }
}
```

### Cross-cutting concern (cắt ngang nhiều module)
```java
// Logging - cần ở MỌI service
// Security - cần ở MỌI service
// Transaction - cần ở MỌI service

// KHÔNG dùng AOP: phải lặp lại code ở mọi nơi
@Service
public class OrderService {
    public Order createOrder(OrderDto dto) {
        log.info("Creating order...");       // Logging - LẶP LẠI
        checkPermission();                   // Security - LẶP LẠI
        beginTransaction();                  // Transaction - LẶP LẠI
        // business logic
        commitTransaction();
    }
}

// DÙNG AOP: tách riêng, áp dụng tự động
@Aspect
@Component
public class LoggingAspect {
    @Before("execution(* com.example.service.*.*(..))")
    public void log(JoinPoint jp) {
        log.info("Calling: " + jp.getSignature());
    }
}
// -> Logging tự động áp dụng cho TẤT CẢ service
```

## Các cross-cutting concern phổ biến

| Cross-cutting concern | Spring implementation |
|---|---|
| **Logging** | AOP (@Aspect) |
| **Transaction** | @Transactional |
| **Security** | Spring Security (Filter chain) |
| **Caching** | @Cacheable |
| **Error handling** | @ControllerAdvice |
| **Validation** | @Valid, Bean Validation |
| **Monitoring** | Spring Actuator, Micrometer |

## Điểm quan trọng nhớ phỏng vấn
1. **Concern** = logic riêng của 1 module
2. **Cross-cutting** = logic **chung** cắt ngang nhiều module
3. AOP giúp **tách** cross-cutting concern ra khỏi business logic
4. Kết quả: code **sạch**, **dễ bảo trì**, **không lặp lại**
