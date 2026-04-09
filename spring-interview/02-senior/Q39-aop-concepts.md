# Q39: What are Aspect, Advice, Pointcut, and JoinPoint in AOP?
> **Dịch:** Aspect, Advice, Pointcut và JoinPoint trong AOP là gì?

## Tra loi ngan gon
> 4 khai niem cot loi cua AOP:
> - **Aspect** = Module chua logic cat ngang (class danh dau @Aspect)
> - **Advice** = Hanh dong cu the (Before, After, Around...)
> - **Pointcut** = Bieu thuc xac dinh AP DUNG O DAU
> - **JoinPoint** = Diem cu the trong code (method dang chay)

## Cach nho - Vi du camera an ninh

```
Aspect   = He thong camera an ninh (toan bo he thong)
Advice   = Camera LAM GI (chup hinh, quay video, bao dong)
Pointcut = Camera DAT O DAU (cua chinh, kho hang, phong server)
JoinPoint = THOI DIEM cu the camera chup (khi ai do di qua)
```

## Vi du ket hop tat ca

```java
@Aspect                          // <-- ASPECT: module chua logic
@Component
public class SecurityAspect {

    // POINTCUT: dinh nghia ap dung o dau
    @Pointcut("@annotation(com.example.Secured)")
    public void securedMethods() {}

    // ADVICE: dinh nghia lam gi + khi nao
    @Before("securedMethods()")
    public void checkPermission(JoinPoint joinPoint) {  // <-- JOINPOINT: diem cu the
        String method = joinPoint.getSignature().getName();
        Object[] args = joinPoint.getArgs();
        
        // Kiem tra quyen
        if (!hasPermission()) {
            throw new AccessDeniedException("No permission for: " + method);
        }
    }
}

// Su dung:
@Service
public class AdminService {
    @Secured                     // Method nay se bi kiem tra quyen
    public void deleteUser(Long id) { }
}
```

## Pointcut Expressions pho bien

```java
// 1. Theo method pattern
@Pointcut("execution(* com.example.service.*.*(..))")
// = Tat ca method, trong tat ca class, package service

// 2. Theo annotation
@Pointcut("@annotation(org.springframework.transaction.annotation.Transactional)")
// = Tat ca method co @Transactional

// 3. Theo class co annotation
@Pointcut("@within(org.springframework.stereotype.Service)")
// = Tat ca method trong class co @Service

// 4. Ket hop (AND, OR, NOT)
@Pointcut("execution(* com.example.service.*.*(..)) && !execution(* *.get*(..))")
// = Tat ca method trong service TRU getXxx methods
```

## Bang tom tat

| Khai niem | La gi | Vi du |
|-----------|-------|-------|
| **Aspect** | Class chua cross-cutting logic | `@Aspect class LoggingAspect` |
| **Advice** | Hanh dong (Before/After/Around) | `@Before("pointcut")` |
| **Pointcut** | Bieu thuc xac dinh noi ap dung | `execution(* service.*.*(..))` |
| **JoinPoint** | Diem thuc thi cu the | Method dang chay, co ten + args |

## Diem quan trong nho phong van
1. **Aspect** = Pointcut + Advice (O DAU + LAM GI)
2. **JoinPoint** cho truy cap **ten method, tham so, target object**
3. **Pointcut** dung bieu thuc `execution()`, `@annotation()`, `within()`
4. Co the **tai su dung** Pointcut bang cach dinh nghia rieng
