# Q3: What is AOP?
> **Dịch:** AOP (Lập trình hướng khía cạnh) là gì?

## Trả lời ngắn gọn
> AOP (Aspect-Oriented Programming) là kỹ thuật lập trình hướng **khía cạnh**, cho phép tách các **cross-cutting concerns** (logging, security, transaction) ra khỏi business logic chính.

## Cách nhớ
```
OOP  = Tổ chức code theo OBJECT (đối tượng)
AOP  = Tổ chức code theo ASPECT (khía cạnh / mối quan tâm chung)

Ví dụ thực tế:
- Business logic = Làm bài thi
- AOP = Camera giám sát (chạy độc lập, không ảnh hưởng bài thi)
```

## Vấn đề AOP giải quyết
**KHÔNG** có AOP - code logging tràn ngập khắp nơi:
```java
public class OrderService {
    public void createOrder(Order order) {
        log.info("START createOrder");        // lặp lại
        long start = System.currentTimeMillis(); // lặp lại
        
        // === Business logic ===
        orderRepo.save(order);
        
        log.info("END createOrder, time: " +   // lặp lại
            (System.currentTimeMillis() - start));
    }
    
    public void cancelOrder(Long id) {
        log.info("START cancelOrder");         // LẶP LẠI!
        long start = System.currentTimeMillis();
        
        // === Business logic ===
        orderRepo.deleteById(id);
        
        log.info("END cancelOrder, time: " +
            (System.currentTimeMillis() - start));
    }
}
```

**CÓ** AOP - code sạch sẽ:
```java
// Business logic - SẠCH, chỉ có logic thôi
@Service
public class OrderService {
    public void createOrder(Order order) {
        orderRepo.save(order);
    }
    public void cancelOrder(Long id) {
        orderRepo.deleteById(id);
    }
}

// Logging tách riêng thành 1 Aspect
@Aspect
@Component
public class LoggingAspect {
    @Around("execution(* com.example.service.*.*(..))")
    public Object logMethod(ProceedingJoinPoint joinPoint) throws Throwable {
        String method = joinPoint.getSignature().getName();
        log.info("START " + method);
        long start = System.currentTimeMillis();
        
        Object result = joinPoint.proceed(); // gọi method gốc
        
        log.info("END " + method + ", time: " + 
            (System.currentTimeMillis() - start));
        return result;
    }
}
```

## Các khái niệm chính trong AOP

| Khái niệm | Giải thích | Ví dụ |
|-----------|-----------|-------|
| **Aspect** | Module chứa logic cắt ngang | LoggingAspect, SecurityAspect |
| **Join Point** | Điểm mà Aspect có thể chèn vào | Khi method được gọi |
| **Advice** | Hành động thực hiện | @Before, @After, @Around |
| **Pointcut** | Biểu thức xác định Join Point nào | `execution(* com.example.*.*(..))` |
| **Weaving** | Quá trình kết hợp Aspect vào code | Compile-time, Runtime |

## Các loại Advice
```
@Before  ──> [METHOD] ──> @AfterReturning (thành công)
                      ──> @AfterThrowing  (có exception)
                      ──> @After          (luôn chạy)
@Around  ──> BAO QUANH method (mạnh nhất)
```

## Điểm quan trọng nhớ phỏng vấn
1. AOP giúp **tách** cross-cutting concerns ra khỏi business logic
2. Spring AOP dùng **Proxy pattern** (Runtime weaving)
3. Các use case phổ biến: **Logging, Security, Transaction, Caching**
4. `@EnableAspectJAutoProxy` để bật AOP trong Spring
