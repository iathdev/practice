# Q31: What are the different types of Advices?
> **Dịch:** Các loại Advice khác nhau là gì?

## Trả lời ngắn gọn
> Advice là **hành động** thực hiện tại một **Join Point**. Có 5 loại: **@Before, @After, @AfterReturning, @AfterThrowing, @Around**.

## Cách nhớ
```
Method gốc: [===THỰC THI===]

@Before          --> [===]
@After           [===] -->          (luôn chạy, cả khi exception)
@AfterReturning  [===] -->          (chỉ khi THÀNH CÔNG)
@AfterThrowing   [===] --> X        (chỉ khi CÓ LỖI)
@Around      --> [===] -->          (bao quanh, MẠNH NHẤT)
```

## Ví dụ từng loại

```java
@Aspect
@Component
public class LoggingAspect {

    // 1. @Before - Chạy TRƯỚC method
    @Before("execution(* com.example.service.*.*(..))")
    public void before(JoinPoint jp) {
        System.out.println("BEFORE: " + jp.getSignature().getName());
    }

    // 2. @After - Chạy SAU method (LUÔN CHẠY, kể cả exception)
    @After("execution(* com.example.service.*.*(..))")
    public void after(JoinPoint jp) {
        System.out.println("AFTER: " + jp.getSignature().getName());
    }

    // 3. @AfterReturning - Chỉ chạy khi THÀNH CÔNG
    @AfterReturning(
        pointcut = "execution(* com.example.service.*.*(..))",
        returning = "result")
    public void afterReturning(JoinPoint jp, Object result) {
        System.out.println("SUCCESS: " + jp.getSignature() + " -> " + result);
    }

    // 4. @AfterThrowing - Chỉ chạy khi CÓ EXCEPTION
    @AfterThrowing(
        pointcut = "execution(* com.example.service.*.*(..))",
        throwing = "ex")
    public void afterThrowing(JoinPoint jp, Exception ex) {
        System.out.println("ERROR: " + jp.getSignature() + " -> " + ex.getMessage());
    }

    // 5. @Around - BAO QUANH (mạnh nhất, làm được tất cả)
    @Around("execution(* com.example.service.*.*(..))")
    public Object around(ProceedingJoinPoint pjp) throws Throwable {
        System.out.println("BEFORE (around)");
        long start = System.currentTimeMillis();

        Object result = pjp.proceed(); // Gọi method gốc

        long time = System.currentTimeMillis() - start;
        System.out.println("AFTER (around) - " + time + "ms");
        return result;
    }
}
```

## Thứ tự chạy khi kết hợp

```
@Around (before)
  @Before
    [Method thực thi]
  @AfterReturning (nếu thành công)
  @AfterThrowing  (nếu có lỗi)
  @After          (luôn chạy)
@Around (after)
```

## Bảng tóm tắt

| Advice | Khi nào chạy | Truy cập result | Truy cập exception | Thay đổi kết quả |
|--------|-------------|:---:|:---:|:---:|
| @Before | Trước method | Không | Không | Không |
| @After | Sau method (luôn) | Không | Không | Không |
| @AfterReturning | Sau khi thành công | Có | Không | Không |
| @AfterThrowing | Khi có exception | Không | Có | Không |
| @Around | Bao quanh | Có | Có | **Có** |

## Điểm quan trọng nhớ phỏng vấn
1. **@Around** mạnh nhất - có thể **thay đổi** input/output, **bỏ qua** method gốc
2. **@After** luôn chạy (như `finally`), **@AfterReturning** chỉ khi thành công
3. `ProceedingJoinPoint.proceed()` chỉ có trong **@Around**
4. Thứ tự chạy: Around -> Before -> Method -> AfterReturning/AfterThrowing -> After -> Around
