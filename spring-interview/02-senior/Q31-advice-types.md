# Q31: What are the different types of Advices?
> **Dịch:** Các loại Advice khác nhau là gì?

## Tra loi ngan gon
> Advice la **hanh dong** thuc hien tai mot **Join Point**. Co 5 loai: **@Before, @After, @AfterReturning, @AfterThrowing, @Around**.

## Cach nho
```
Method goc: [===THUC THI===]

@Before          --> [===]
@After           [===] -->          (luon chay, ca khi exception)
@AfterReturning  [===] -->          (chi khi THANH CONG)
@AfterThrowing   [===] --> X        (chi khi CO LOI)
@Around      --> [===] -->          (bao quanh, MANH NHAT)
```

## Vi du tung loai

```java
@Aspect
@Component
public class LoggingAspect {

    // 1. @Before - Chay TRUOC method
    @Before("execution(* com.example.service.*.*(..))")
    public void before(JoinPoint jp) {
        System.out.println("BEFORE: " + jp.getSignature().getName());
    }

    // 2. @After - Chay SAU method (LUON CHAY, ke ca exception)
    @After("execution(* com.example.service.*.*(..))")
    public void after(JoinPoint jp) {
        System.out.println("AFTER: " + jp.getSignature().getName());
    }

    // 3. @AfterReturning - Chi chay khi THANH CONG
    @AfterReturning(
        pointcut = "execution(* com.example.service.*.*(..))",
        returning = "result")
    public void afterReturning(JoinPoint jp, Object result) {
        System.out.println("SUCCESS: " + jp.getSignature() + " -> " + result);
    }

    // 4. @AfterThrowing - Chi chay khi CO EXCEPTION
    @AfterThrowing(
        pointcut = "execution(* com.example.service.*.*(..))",
        throwing = "ex")
    public void afterThrowing(JoinPoint jp, Exception ex) {
        System.out.println("ERROR: " + jp.getSignature() + " -> " + ex.getMessage());
    }

    // 5. @Around - BAO QUANH (manh nhat, lam duoc tat ca)
    @Around("execution(* com.example.service.*.*(..))")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        System.out.println("BEFORE (around)");
        long start = System.currentTimeMillis();

        Object result = pjp.proceed(); // Goi method goc

        long time = System.currentTimeMillis() - start;
        System.out.println("AFTER (around) - " + time + "ms");
        return result;
    }
}
```

## Thu tu chay khi ket hop

```
@Around (before)
  @Before
    [Method thuc thi]
  @AfterReturning (neu thanh cong)
  @AfterThrowing  (neu co loi)
  @After          (luon chay)
@Around (after)
```

## Bang tom tat

| Advice | Khi nao chay | Truy cap result | Truy cap exception | Thay doi ket qua |
|--------|-------------|:---:|:---:|:---:|
| @Before | Truoc method | Khong | Khong | Khong |
| @After | Sau method (luon) | Khong | Khong | Khong |
| @AfterReturning | Sau khi thanh cong | Co | Khong | Khong |
| @AfterThrowing | Khi co exception | Khong | Co | Khong |
| @Around | Bao quanh | Co | Co | **Co** |

## Diem quan trong nho phong van
1. **@Around** manh nhat - co the **thay doi** input/output, **bo qua** method goc
2. **@After** luon chay (nhu `finally`), **@AfterReturning** chi khi thanh cong
3. `ProceedingJoinPoint.proceed()` chi co trong **@Around**
4. Thu tu chay: Around -> Before -> Method -> AfterReturning/AfterThrowing -> After -> Around
