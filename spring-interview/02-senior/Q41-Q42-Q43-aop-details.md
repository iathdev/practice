# Q41: What are inner beans in Spring?
> **Dịch:** Inner Bean trong Spring là gì?

# Q42: What is Join point?
> **Dịch:** Join Point là gì?

# Q43: What is Aspect-Oriented Programming?
> **Dịch:** Lập trình hướng khía cạnh (AOP) là gì?

---

## Q41: Inner Beans

### Tra loi ngan gon
> **Inner bean** la bean duoc dinh nghia **ben trong** mot bean khac (trong XML config). No khong co id, khong the truy cap tu ben ngoai, chi dung cho bean cha.

```xml
<bean id="orderService" class="com.example.OrderService">
    <!-- Inner bean - chi dung trong orderService -->
    <property name="emailService">
        <bean class="com.example.EmailService">
            <property name="host" value="smtp.gmail.com"/>
        </bean>
    </property>
</bean>
```

### Tuong duong trong annotation (hien dai):
```java
@Configuration
public class AppConfig {
    @Bean
    public OrderService orderService() {
        // emailService() giong nhu inner bean
        return new OrderService(emailService());
    }

    // Hoac bean rieng - KHUYEN DUNG hon inner bean
    @Bean
    public EmailService emailService() {
        return new EmailService("smtp.gmail.com");
    }
}
```

### Diem nho: Inner bean hiem dung trong thuc te. Annotation-based config thay the.

---

## Q42: Join Point

### Tra loi ngan gon
> **Join Point** la mot **diem cu the** trong qua trinh thuc thi chuong trinh noi ma Aspect co the **chen vao**. Trong Spring AOP, Join Point luon la **method execution** (khong ho tro field access).

```java
@Before("execution(* com.example.service.*.*(..))")
public void logBefore(JoinPoint joinPoint) {
    // JoinPoint cho phep truy cap:
    String methodName = joinPoint.getSignature().getName();     // Ten method
    Object[] args = joinPoint.getArgs();                        // Tham so
    Object target = joinPoint.getTarget();                      // Object goc
    String className = joinPoint.getTarget().getClass().getName();
    
    System.out.println(className + "." + methodName + "(" + Arrays.toString(args) + ")");
}
```

### Cac thong tin tu JoinPoint

| Method | Tra ve |
|--------|--------|
| `getSignature()` | Ten method, return type |
| `getArgs()` | Mang tham so |
| `getTarget()` | Object goc (khong phai proxy) |
| `getThis()` | Proxy object |

---

## Q43: Aspect-Oriented Programming (AOP)

### Tra loi ngan gon
> AOP la **mo hinh lap trinh** bo sung cho OOP, cho phep tach **cross-cutting concerns** (logging, security, transaction) ra khoi business logic. Spring AOP dung **proxy** de "chen" logic vao truoc/sau method.

### Tai sao can AOP?
```
OOP tot cho to chuc code theo OBJECT
Nhung mot so logic KHONG thuoc rieng object nao:
  - Logging -> can o MOI service
  - Security -> can o MOI controller
  - Transaction -> can o MOI write operation

AOP = Giai phap cho van de nay
```

### Nguyen tac hoat dong
```
KHONG AOP:  Client --> Service.method()
CO AOP:     Client --> Proxy --> [Advice] --> Service.method() --> [Advice] --> Response
```

(Chi tiet xem Q03, Q07, Q31, Q35, Q39)

## Diem quan trong nho phong van
1. Inner bean = bean con dinh nghia trong bean cha (hiem dung)
2. JoinPoint = diem cu the trong code (Spring chi ho tro **method execution**)
3. AOP = tach cross-cutting concern, bo sung cho OOP
4. Spring AOP = **proxy-based** (runtime), AspectJ = **bytecode-based** (compile/load time)
