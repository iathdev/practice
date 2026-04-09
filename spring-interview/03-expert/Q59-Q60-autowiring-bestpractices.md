# Q59: How does autowiring work in Spring?
> **Dịch:** Autowiring hoạt động như thế nào trong Spring?

# Q60: What are some of the best practices for Spring Framework?
> **Dịch:** Một số best practice cho Spring Framework là gì?

---

## Q59: Autowiring hoạt động như thế nào?

### Trả lời ngắn gọn
> Khi container khởi động, nó: (1) scan tìm bean, (2) phân tích dependency, (3) tìm bean phù hợp theo **type**, (4) inject qua constructor/setter/field. Nếu có nhiều bean cùng type thì dùng **@Qualifier/@Primary** để phân giải.

### Quá trình nội bộ

```
1. Component Scanning
   - Quét các package tìm @Component, @Service, @Repository, @Controller
   - Đăng ký bean definitions

2. Bean Creation (theo thứ tự dependency)
   - A phụ thuộc B -> Tạo B trước, rồi tạo A

3. Dependency Resolution
   - Tìm bean theo TYPE (primary strategy)
   - Nếu nhiều bean cùng type:
     a. Tìm @Primary
     b. Tìm @Qualifier
     c. Tìm theo tên biến (fallback)
     d. Ném NoUniqueBeanDefinitionException

4. Injection
   - Constructor: inject khi tạo bean
   - Setter: gọi setter sau khi tạo
   - Field: dùng reflection sau khi tạo
```

### Ví dụ quá trình resolution
```java
// Có 2 bean DataSource
@Bean @Primary
public DataSource mysqlDS() { ... }

@Bean
public DataSource postgresDS() { ... }

// Case 1: Inject -> chọn @Primary (mysqlDS)
@Autowired
private DataSource dataSource; // -> mysqlDS

// Case 2: Dùng @Qualifier -> chọn cụ thể
@Autowired
@Qualifier("postgresDS")
private DataSource dataSource; // -> postgresDS

// Case 3: Tên biến trùng tên bean -> match theo tên
@Autowired
private DataSource postgresDS; // -> postgresDS (match by name)
```

---

## Q60: Best Practices for Spring Framework

### 1. Dependency Injection
```java
// ĐÚNG: Constructor injection
@Service
public class UserService {
    private final UserRepository repo; // final = immutable
    public UserService(UserRepository repo) { this.repo = repo; }
}

// TRÁNH: Field injection
@Service
public class UserService {
    @Autowired // Khó test, không final
    private UserRepository repo;
}
```

### 2. Layered Architecture
```
@Controller  -> Chỉ xử lý HTTP request/response
@Service     -> Business logic
@Repository  -> Data access
// KHÔNG gọi Repository từ Controller trực tiếp
```

### 3. Exception Handling
```java
// ĐÚNG: Global handler
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(NotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handle(NotFoundException ex) { }
}

// TRÁNH: try-catch khắp nơi trong controller
```

### 4. Configuration
```java
// ĐÚNG: Type-safe config
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private String name;
    private int timeout;
}

// TRÁNH: @Value khắp nơi
@Value("${app.name}") private String name;
@Value("${app.timeout}") private int timeout;
```

### 5. Transaction
```java
// ĐÚNG: Service level, readOnly cho read
@Service
@Transactional(readOnly = true)
public class UserService {
    @Transactional
    public User create(UserDto dto) { }
}
```

### 6. Các best practices khác
```
- Dùng Spring Boot profiles cho môi trường (dev, staging, prod)
- Dùng @Validated cho input validation
- Dùng Spring Actuator cho monitoring
- Dùng constructor injection (testable, immutable)
- Dùng interface cho services (dễ mock khi test)
- Tránh circular dependencies
- Giữ bean STATELESS (singleton)
- Log ở mức phù hợp (DEBUG cho dev, INFO cho prod)
```

## Điểm quan trọng nhớ phỏng vấn
1. Autowiring: **byType** -> @Primary -> @Qualifier -> **byName**
2. Best practice #1: **Constructor injection** (final, testable)
3. Best practice #2: **Layered architecture** (Controller -> Service -> Repository)
4. Best practice #3: **@Transactional** ở Service, `readOnly=true` cho read
