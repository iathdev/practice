# Q35: What is the difference between concern and cross-cutting concern in Spring AOP?
> **Dịch:** Sự khác nhau giữa Concern và Cross-cutting Concern trong Spring AOP là gì?

## Tra loi ngan gon
> **Concern** = chuc nang cu the cua 1 module (vd: xu ly order). **Cross-cutting concern** = chuc nang **cat ngang** nhieu module (vd: logging, security, transaction). AOP xu ly cross-cutting concern.

## Cach nho
```
Concern = Mon hoc (Toan, Van, Anh) - moi mon rieng biet
Cross-cutting = Diem danh - ap dung cho TAT CA mon hoc
```

## Minh hoa

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
               (cat ngang TAT CA services)
```

## Vi du cu the

### Concern (business logic rieng)
```java
@Service
public class OrderService {
    // Concern: chi xu ly ORDER
    public Order createOrder(OrderDto dto) { /* ... */ }
    public void cancelOrder(Long id) { /* ... */ }
}

@Service
public class UserService {
    // Concern: chi xu ly USER
    public User register(UserDto dto) { /* ... */ }
    public User findById(Long id) { /* ... */ }
}
```

### Cross-cutting concern (cat ngang nhieu module)
```java
// Logging - can o MOI service
// Security - can o MOI service
// Transaction - can o MOI service

// KHONG dung AOP: phai lap lai code o moi noi
@Service
public class OrderService {
    public Order createOrder(OrderDto dto) {
        log.info("Creating order...");       // Logging - LAP LAI
        checkPermission();                   // Security - LAP LAI
        beginTransaction();                  // Transaction - LAP LAI
        // business logic
        commitTransaction();
    }
}

// DUNG AOP: tach rieng, ap dung tu dong
@Aspect
@Component
public class LoggingAspect {
    @Before("execution(* com.example.service.*.*(..))")
    public void log(JoinPoint jp) {
        log.info("Calling: " + jp.getSignature());
    }
}
// -> Logging tu dong ap dung cho TAT CA service
```

## Cac cross-cutting concern pho bien

| Cross-cutting concern | Spring implementation |
|---|---|
| **Logging** | AOP (@Aspect) |
| **Transaction** | @Transactional |
| **Security** | Spring Security (Filter chain) |
| **Caching** | @Cacheable |
| **Error handling** | @ControllerAdvice |
| **Validation** | @Valid, Bean Validation |
| **Monitoring** | Spring Actuator, Micrometer |

## Diem quan trong nho phong van
1. **Concern** = logic rieng cua 1 module
2. **Cross-cutting** = logic **chung** cat ngang nhieu module
3. AOP giup **tach** cross-cutting concern ra khoi business logic
4. Ket qua: code **sach**, **de bao tri**, **khong lap lai**
