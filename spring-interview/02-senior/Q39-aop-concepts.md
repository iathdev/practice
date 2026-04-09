# Q39: What are Aspect, Advice, Pointcut, and JoinPoint in AOP?
> **Dịch:** Aspect, Advice, Pointcut và JoinPoint trong AOP là gì?

## Trả lời ngắn gọn
> 4 khái niệm cốt lõi của AOP:
> - **Aspect** = Module chứa logic cắt ngang (class đánh dấu @Aspect)
> - **Advice** = Hành động cụ thể (Before, After, Around...)
> - **Pointcut** = Biểu thức xác định ÁP DỤNG Ở ĐÂU
> - **JoinPoint** = Điểm cụ thể trong code (method đang chạy)

## Cách nhớ - Ví dụ camera an ninh

```
Aspect   = Hệ thống camera an ninh (toàn bộ hệ thống)
Advice   = Camera LÀM GÌ (chụp hình, quay video, báo động)
Pointcut = Camera ĐẶT Ở ĐÂU (cửa chính, kho hàng, phòng server)
JoinPoint = THỜI ĐIỂM cụ thể camera chụp (khi ai đó đi qua)
```

## Ví dụ kết hợp tất cả

```java
@Aspect                          // <-- ASPECT: module chứa logic
@Component
public class SecurityAspect {

    // POINTCUT: định nghĩa áp dụng ở đâu
    @Pointcut("@annotation(com.example.Secured)")
    public void securedMethods() {}

    // ADVICE: định nghĩa làm gì + khi nào
    @Before("securedMethods()")
    public void checkPermission(JoinPoint joinPoint) {  // <-- JOINPOINT: điểm cụ thể
        String method = joinPoint.getSignature().getName();
        Object[] args = joinPoint.getArgs();
        
        // Kiểm tra quyền
        if (!hasPermission()) {
            throw new AccessDeniedException("No permission for: " + method);
        }
    }
}

// Sử dụng:
@Service
public class AdminService {
    @Secured                     // Method này sẽ bị kiểm tra quyền
    public void deleteUser(Long id) { }
}
```

## Pointcut Expressions phổ biến

```java
// 1. Theo method pattern
@Pointcut("execution(* com.example.service.*.*(..))")
// = Tất cả method, trong tất cả class, package service

// 2. Theo annotation
@Pointcut("@annotation(org.springframework.transaction.annotation.Transactional)")
// = Tất cả method có @Transactional

// 3. Theo class có annotation
@Pointcut("@within(org.springframework.stereotype.Service)")
// = Tất cả method trong class có @Service

// 4. Kết hợp (AND, OR, NOT)
@Pointcut("execution(* com.example.service.*.*(..)) && !execution(* *.get*(..))")
// = Tất cả method trong service TRỪ getXxx methods
```

## Bảng tóm tắt

| Khái niệm | Là gì | Ví dụ |
|-----------|-------|-------|
| **Aspect** | Class chứa cross-cutting logic | `@Aspect class LoggingAspect` |
| **Advice** | Hành động (Before/After/Around) | `@Before("pointcut")` |
| **Pointcut** | Biểu thức xác định nơi áp dụng | `execution(* service.*.*(..))` |
| **JoinPoint** | Điểm thực thi cụ thể | Method đang chạy, có tên + args |

## Điểm quan trọng nhớ phỏng vấn
1. **Aspect** = Pointcut + Advice (Ở ĐÂU + LÀM GÌ)
2. **JoinPoint** cho truy cập **tên method, tham số, target object**
3. **Pointcut** dùng biểu thức `execution()`, `@annotation()`, `within()`
4. Có thể **tái sử dụng** Pointcut bằng cách định nghĩa riêng
