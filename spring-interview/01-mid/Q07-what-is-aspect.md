# Q7: What is Aspect?
> **Dịch:** Aspect (khía cạnh) là gì?

## Trả lời ngắn gọn
> **Aspect** là một module chứa code xử lý các **cross-cutting concern** (vấn đề cắt ngang) như logging, security, transaction. Nó gồm **Advice** (làm gì) + **Pointcut** (làm ở đâu).

## Cách nhớ
```
Aspect = Camera an ninh trong tòa nhà
- Pointcut = Đặt camera Ở ĐÂU (cửa chính, tầng 2, phòng server)
- Advice   = Camera LÀM GÌ (chụp hình, quay video, báo động)
```

## Ví dụ đơn giản

```java
@Aspect        // Đánh dấu đây là 1 Aspect
@Component     // Để Spring quản lý
public class LoggingAspect {

    // Pointcut: áp dụng cho tất cả method trong package service
    // Advice (@Before): chạy TRƯỚC khi method được gọi
    @Before("execution(* com.example.service.*.*(..))")
    public void logBefore(JoinPoint joinPoint) {
        System.out.println("Calling: " + joinPoint.getSignature().getName());
    }
}
```

## Cấu trúc của Aspect

```
@Aspect
public class MyAspect {

    // POINTCUT - định nghĩa Ở ĐÂU
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceMethods() {}

    // ADVICE - định nghĩa LÀM GÌ + KHI NÀO
    @Before("serviceMethods()")
    public void before() { /* chạy trước */ }

    @After("serviceMethods()")
    public void after() { /* chạy sau (luôn chạy) */ }

    @AfterReturning("serviceMethods()")
    public void afterSuccess() { /* chạy khi thành công */ }

    @AfterThrowing("serviceMethods()")
    public void afterError() { /* chạy khi có exception */ }

    @Around("serviceMethods()")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        // chạy trước
        Object result = pjp.proceed(); // gọi method gốc
        // chạy sau
        return result;
    }
}
```

## Ví dụ thực tế: Đo thời gian method

```java
@Aspect
@Component
public class PerformanceAspect {

    @Around("@annotation(com.example.annotation.TrackTime)")
    public Object trackTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        
        Object result = pjp.proceed();
        
        long duration = System.currentTimeMillis() - start;
        System.out.println(pjp.getSignature() + " took " + duration + "ms");
        
        return result;
    }
}

// Sử dụng:
@Service
public class UserService {
    @TrackTime  // Chỉ cần thêm annotation này
    public List<User> findAll() {
        return userRepo.findAll();
    }
}
```

## Điểm quan trọng nhớ phỏng vấn
1. Aspect = **Pointcut + Advice** (ở đâu + làm gì)
2. Dùng `@Aspect` + `@Component` để khai báo
3. Cần `@EnableAspectJAutoProxy` (Spring Boot tự bật)
4. Các use case: **Logging, Security, Caching, Transaction, Performance monitoring**
