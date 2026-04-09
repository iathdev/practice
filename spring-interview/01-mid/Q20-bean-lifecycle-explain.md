# Q20: Explain Bean lifecycle in Spring framework
> **Dịch:** Giải thích vòng đời (lifecycle) của Bean trong Spring Framework

## Trả lời ngắn gọn
> Bean lifecycle gồm 4 giai đoạn chính: **Instantiation -> Dependency Injection -> Initialization -> Destruction**. Developer can thiệp được qua `@PostConstruct`, `@PreDestroy`, và `BeanPostProcessor`.

(Xem chi tiết Q06 - câu này là bản mở rộng)

## Cách nhớ - Sơ đồ nhanh

```
+------------------+
| 1. TẠO (new)     |  Container gọi constructor
+------------------+
        |
+------------------+
| 2. INJECT (DI)   |  Set dependencies (@Autowired)
+------------------+
        |
+------------------+
| 3. INIT          |  @PostConstruct -> afterPropertiesSet -> init-method
+------------------+
        |
+------------------+
| 4. SỬ DỤNG       |  Bean sẵn sàng phục vụ
+------------------+
        |
+------------------+
| 5. DESTROY       |  @PreDestroy -> destroy() -> destroy-method
+------------------+
```

## Ví dụ thực hành

```java
@Component
public class DatabaseConnection implements InitializingBean, DisposableBean {

    @Value("${db.url}")
    private String dbUrl;

    private Connection connection;

    // 1. Constructor - Tạo bean
    public DatabaseConnection() {
        System.out.println("STEP 1: Constructor called");
    }

    // 2. Dependencies đã được inject (dbUrl có giá trị)

    // 3a. @PostConstruct - chạy đầu tiên
    @PostConstruct
    public void postConstruct() {
        System.out.println("STEP 3a: @PostConstruct - dbUrl = " + dbUrl);
        // Khởi tạo connection
    }

    // 3b. InitializingBean
    @Override
    public void afterPropertiesSet() {
        System.out.println("STEP 3b: afterPropertiesSet");
    }

    // === Bean đang được sử dụng ===

    // 5a. @PreDestroy
    @PreDestroy
    public void preDestroy() {
        System.out.println("STEP 5a: @PreDestroy - Đóng connection");
    }

    // 5b. DisposableBean
    @Override
    public void destroy() {
        System.out.println("STEP 5b: destroy");
    }
}

// Output:
// STEP 1: Constructor called
// STEP 3a: @PostConstruct - dbUrl = jdbc:mysql://localhost/mydb
// STEP 3b: afterPropertiesSet
// --- App chạy ---
// STEP 5a: @PreDestroy - Đóng connection
// STEP 5b: destroy
```

## BeanPostProcessor - can thiệp vào TẤT CẢ bean

```java
@Component
public class CustomBeanPostProcessor implements BeanPostProcessor {

    // Chạy SAU inject, TRƯỚC @PostConstruct
    @Override
    public Object postProcessBeforeInitialization(Object bean, String name) {
        System.out.println("Before init: " + name);
        return bean;
    }

    // Chạy SAU @PostConstruct
    @Override
    public Object postProcessAfterInitialization(Object bean, String name) {
        System.out.println("After init: " + name);
        return bean; // Có thể return proxy thay vì bean gốc
    }
}
```

## Use case thực tế

| Giai đoạn | Làm gì |
|-----------|--------|
| @PostConstruct | Load cache, validate config, khởi tạo resource |
| @PreDestroy | Đóng connection, flush buffer, giải phóng resource |
| BeanPostProcessor | Custom annotation processing, tạo proxy |

## Điểm quan trọng nhớ phỏng vấn
1. Thứ tự Init: **@PostConstruct** -> **afterPropertiesSet** -> **init-method**
2. Thứ tự Destroy: **@PreDestroy** -> **destroy()** -> **destroy-method**
3. **@PostConstruct/@PreDestroy** là cách đơn giản nhất (best practice)
4. **Prototype** bean: Spring **KHÔNG** gọi destroy method
