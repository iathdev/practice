# Q41: What are inner beans in Spring?
> **Dịch:** Inner Bean trong Spring là gì?

# Q42: What is Join point?
> **Dịch:** Join Point là gì?

# Q43: What is Aspect-Oriented Programming?
> **Dịch:** Lập trình hướng khía cạnh (AOP) là gì?

---

## Q41: Inner Beans

### Trả lời ngắn gọn
> **Inner bean** là bean được định nghĩa **bên trong** một bean khác (trong XML config). Nó không có id, không thể truy cập từ bên ngoài, chỉ dùng cho bean cha.

```xml
<bean id="orderService" class="com.example.OrderService">
    <!-- Inner bean - chỉ dùng trong orderService -->
    <property name="emailService">
        <bean class="com.example.EmailService">
            <property name="host" value="smtp.gmail.com"/>
        </bean>
    </property>
</bean>
```

### Tương đương trong annotation (hiện đại):
```java
@Configuration
public class AppConfig {
    @Bean
    public OrderService orderService() {
        // emailService() giống như inner bean
        return new OrderService(emailService());
    }

    // Hoặc bean riêng - KHUYÊN DÙNG hơn inner bean
    @Bean
    public EmailService emailService() {
        return new EmailService("smtp.gmail.com");
    }
}
```

### Điểm nhớ: Inner bean hiếm dùng trong thực tế. Annotation-based config thay thế.

---

## Q42: Join Point

### Trả lời ngắn gọn
> **Join Point** là một **điểm cụ thể** trong quá trình thực thi chương trình nơi mà Aspect có thể **chèn vào**. Trong Spring AOP, Join Point luôn là **method execution** (không hỗ trợ field access).

```java
@Before("execution(* com.example.service.*.*(..))")
public void logBefore(JoinPoint joinPoint) {
    // JoinPoint cho phép truy cập:
    String methodName = joinPoint.getSignature().getName();     // Tên method
    Object[] args = joinPoint.getArgs();                        // Tham số
    Object target = joinPoint.getTarget();                      // Object gốc
    String className = joinPoint.getTarget().getClass().getName();
    
    System.out.println(className + "." + methodName + "(" + Arrays.toString(args) + ")");
}
```

### Các thông tin từ JoinPoint

| Method | Trả về |
|--------|--------|
| `getSignature()` | Tên method, return type |
| `getArgs()` | Mảng tham số |
| `getTarget()` | Object gốc (không phải proxy) |
| `getThis()` | Proxy object |

---

## Q43: Aspect-Oriented Programming (AOP)

### Trả lời ngắn gọn
> AOP là **mô hình lập trình** bổ sung cho OOP, cho phép tách **cross-cutting concerns** (logging, security, transaction) ra khỏi business logic. Spring AOP dùng **proxy** để "chèn" logic vào trước/sau method.

### Tại sao cần AOP?
```
OOP tốt cho tổ chức code theo OBJECT
Nhưng một số logic KHÔNG thuộc riêng object nào:
  - Logging -> cần ở MỌI service
  - Security -> cần ở MỌI controller
  - Transaction -> cần ở MỌI write operation

AOP = Giải pháp cho vấn đề này
```

### Nguyên tắc hoạt động
```
KHÔNG AOP:  Client --> Service.method()
CÓ AOP:     Client --> Proxy --> [Advice] --> Service.method() --> [Advice] --> Response
```

(Chi tiết xem Q03, Q07, Q31, Q35, Q39)

## Điểm quan trọng nhớ phỏng vấn
1. Inner bean = bean con định nghĩa trong bean cha (hiếm dùng)
2. JoinPoint = điểm cụ thể trong code (Spring chỉ hỗ trợ **method execution**)
3. AOP = tách cross-cutting concern, bổ sung cho OOP
4. Spring AOP = **proxy-based** (runtime), AspectJ = **bytecode-based** (compile/load time)
