# Q7: What is Aspect?
> **Dịch:** Aspect (khía cạnh) là gì?

## Tra loi ngan gon
> **Aspect** la mot module chua code xu ly cac **cross-cutting concern** (van de cat ngang) nhu logging, security, transaction. No gom **Advice** (lam gi) + **Pointcut** (lam o dau).

## Cach nho
```
Aspect = Camera an ninh trong toa nha
- Pointcut = Dat camera O DAU (cua chinh, tang 2, phong server)
- Advice   = Camera LAM GI (chup hinh, quay video, bao dong)
```

## Vi du don gian

```java
@Aspect        // Danh dau day la 1 Aspect
@Component     // De Spring quan ly
public class LoggingAspect {

    // Pointcut: ap dung cho tat ca method trong package service
    // Advice (@Before): chay TRUOC khi method duoc goi
    @Before("execution(* com.example.service.*.*(..))")
    public void logBefore(JoinPoint joinPoint) {
        System.out.println("Calling: " + joinPoint.getSignature().getName());
    }
}
```

## Cau truc cua Aspect

```
@Aspect
public class MyAspect {

    // POINTCUT - dinh nghia O DAU
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceMethods() {}

    // ADVICE - dinh nghia LAM GI + KHI NAO
    @Before("serviceMethods()")
    public void before() { /* chay truoc */ }

    @After("serviceMethods()")
    public void after() { /* chay sau (luon chay) */ }

    @AfterReturning("serviceMethods()")
    public void afterSuccess() { /* chay khi thanh cong */ }

    @AfterThrowing("serviceMethods()")
    public void afterError() { /* chay khi co exception */ }

    @Around("serviceMethods()")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        // chay truoc
        Object result = pjp.proceed(); // goi method goc
        // chay sau
        return result;
    }
}
```

## Vi du thuc te: Do thoi gian method

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

// Su dung:
@Service
public class UserService {
    @TrackTime  // Chi can them annotation nay
    public List<User> findAll() {
        return userRepo.findAll();
    }
}
```

## Diem quan trong nho phong van
1. Aspect = **Pointcut + Advice** (o dau + lam gi)
2. Dung `@Aspect` + `@Component` de khai bao
3. Can `@EnableAspectJAutoProxy` (Spring Boot tu bat)
4. Cac use case: **Logging, Security, Caching, Transaction, Performance monitoring**
