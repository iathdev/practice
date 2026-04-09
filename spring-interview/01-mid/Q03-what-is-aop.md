# Q3: What is AOP?
> **Dịch:** AOP (Lập trình hướng khía cạnh) là gì?

## Tra loi ngan gon
> AOP (Aspect-Oriented Programming) la ky thuat lap trinh huong **khi canh**, cho phep tach cac **cross-cutting concerns** (logging, security, transaction) ra khoi business logic chinh.

## Cach nho
```
OOP  = To chuc code theo OBJECT (doi tuong)
AOP  = To chuc code theo ASPECT (khi canh / moi quan tam chung)

Vi du thuc te:
- Business logic = Lam bai thi
- AOP = Camera giam sat (chay doc lap, khong anh huong bai thi)
```

## Van de AOP giai quyet
**KHONG** co AOP - code logging tran ngap khap noi:
```java
public class OrderService {
    public void createOrder(Order order) {
        log.info("START createOrder");        // lap lai
        long start = System.currentTimeMillis(); // lap lai
        
        // === Business logic ===
        orderRepo.save(order);
        
        log.info("END createOrder, time: " +   // lap lai
            (System.currentTimeMillis() - start));
    }
    
    public void cancelOrder(Long id) {
        log.info("START cancelOrder");         // LAP LAI!
        long start = System.currentTimeMillis();
        
        // === Business logic ===
        orderRepo.deleteById(id);
        
        log.info("END cancelOrder, time: " +
            (System.currentTimeMillis() - start));
    }
}
```

**CO** AOP - code sach se:
```java
// Business logic - SACH, chi co logic thoi
@Service
public class OrderService {
    public void createOrder(Order order) {
        orderRepo.save(order);
    }
    public void cancelOrder(Long id) {
        orderRepo.deleteById(id);
    }
}

// Logging tach rieng thanh 1 Aspect
@Aspect
@Component
public class LoggingAspect {
    @Around("execution(* com.example.service.*.*(..))")
    public Object logMethod(ProceedingJoinPoint joinPoint) throws Throwable {
        String method = joinPoint.getSignature().getName();
        log.info("START " + method);
        long start = System.currentTimeMillis();
        
        Object result = joinPoint.proceed(); // goi method goc
        
        log.info("END " + method + ", time: " + 
            (System.currentTimeMillis() - start));
        return result;
    }
}
```

## Cac khai niem chinh trong AOP

| Khai niem | Giai thich | Vi du |
|-----------|-----------|-------|
| **Aspect** | Module chua logic cat ngang | LoggingAspect, SecurityAspect |
| **Join Point** | Diem ma Aspect co the chen vao | Khi method duoc goi |
| **Advice** | Hanh dong thuc hien | @Before, @After, @Around |
| **Pointcut** | Bieu thuc xac dinh Join Point nao | `execution(* com.example.*.*(..))` |
| **Weaving** | Qua trinh ket hop Aspect vao code | Compile-time, Runtime |

## Cac loai Advice
```
@Before  ──> [METHOD] ──> @AfterReturning (thanh cong)
                      ──> @AfterThrowing  (co exception)
                      ──> @After          (luon chay)
@Around  ──> BAO QUANH method (manh nhat)
```

## Diem quan trong nho phong van
1. AOP giup **tach** cross-cutting concerns ra khoi business logic
2. Spring AOP dung **Proxy pattern** (Runtime weaving)
3. Cac use case pho bien: **Logging, Security, Transaction, Caching**
4. `@EnableAspectJAutoProxy` de bat AOP trong Spring
