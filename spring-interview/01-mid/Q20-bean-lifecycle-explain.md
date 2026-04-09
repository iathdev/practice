# Q20: Explain Bean lifecycle in Spring framework
> **Dịch:** Giải thích vòng đời (lifecycle) của Bean trong Spring Framework

## Tra loi ngan gon
> Bean lifecycle gom 4 giai doan chinh: **Instantiation -> Dependency Injection -> Initialization -> Destruction**. Developer can thiep duoc qua `@PostConstruct`, `@PreDestroy`, va `BeanPostProcessor`.

(Xem chi tiet Q06 - cau nay la ban mo rong)

## Cach nho - So do nhanh

```
+------------------+
| 1. TAO (new)     |  Container goi constructor
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
| 4. SU DUNG       |  Bean san sang phuc vu
+------------------+
        |
+------------------+
| 5. DESTROY       |  @PreDestroy -> destroy() -> destroy-method
+------------------+
```

## Vi du thuc hanh

```java
@Component
public class DatabaseConnection implements InitializingBean, DisposableBean {

    @Value("${db.url}")
    private String dbUrl;

    private Connection connection;

    // 1. Constructor - Tao bean
    public DatabaseConnection() {
        System.out.println("STEP 1: Constructor called");
    }

    // 2. Dependencies da duoc inject (dbUrl co gia tri)

    // 3a. @PostConstruct - chay dau tien
    @PostConstruct
    public void postConstruct() {
        System.out.println("STEP 3a: @PostConstruct - dbUrl = " + dbUrl);
        // Khoi tao connection
    }

    // 3b. InitializingBean
    @Override
    public void afterPropertiesSet() {
        System.out.println("STEP 3b: afterPropertiesSet");
    }

    // === Bean dang duoc su dung ===

    // 5a. @PreDestroy
    @PreDestroy
    public void preDestroy() {
        System.out.println("STEP 5a: @PreDestroy - Dong connection");
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
// --- App chay ---
// STEP 5a: @PreDestroy - Dong connection
// STEP 5b: destroy
```

## BeanPostProcessor - can thiep vao TAT CA bean

```java
@Component
public class CustomBeanPostProcessor implements BeanPostProcessor {

    // Chay SAU inject, TRUOC @PostConstruct
    @Override
    public Object postProcessBeforeInitialization(Object bean, String name) {
        System.out.println("Before init: " + name);
        return bean;
    }

    // Chay SAU @PostConstruct
    @Override
    public Object postProcessAfterInitialization(Object bean, String name) {
        System.out.println("After init: " + name);
        return bean; // Co the return proxy thay vi bean goc
    }
}
```

## Use case thuc te

| Giai doan | Lam gi |
|-----------|--------|
| @PostConstruct | Load cache, validate config, khoi tao resource |
| @PreDestroy | Dong connection, flush buffer, giai phong resource |
| BeanPostProcessor | Custom annotation processing, tao proxy |

## Diem quan trong nho phong van
1. Thu tu Init: **@PostConstruct** -> **afterPropertiesSet** -> **init-method**
2. Thu tu Destroy: **@PreDestroy** -> **destroy()** -> **destroy-method**
3. **@PostConstruct/@PreDestroy** la cach don gian nhat (best practice)
4. **Prototype** bean: Spring **KHONG** goi destroy method
