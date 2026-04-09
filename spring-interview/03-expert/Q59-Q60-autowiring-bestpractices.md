# Q59: How does autowiring work in Spring?
> **Dịch:** Autowiring hoạt động như thế nào trong Spring?

# Q60: What are some of the best practices for Spring Framework?
> **Dịch:** Một số best practice cho Spring Framework là gì?

---

## Q59: Autowiring hoat dong nhu the nao?

### Tra loi ngan gon
> Khi container khoi dong, no: (1) scan tim bean, (2) phan tich dependency, (3) tim bean phu hop theo **type**, (4) inject qua constructor/setter/field. Neu co nhieu bean cung type thi dung **@Qualifier/@Primary** de phan giai.

### Qua trinh noi bo

```
1. Component Scanning
   - Quet cac package tim @Component, @Service, @Repository, @Controller
   - Dang ky bean definitions

2. Bean Creation (theo thu tu dependency)
   - A phu thuoc B -> Tao B truoc, roi tao A

3. Dependency Resolution
   - Tim bean theo TYPE (primary strategy)
   - Neu nhieu bean cung type:
     a. Tim @Primary
     b. Tim @Qualifier
     c. Tim theo ten bien (fallback)
     d. Nem NoUniqueBeanDefinitionException

4. Injection
   - Constructor: inject khi tao bean
   - Setter: goi setter sau khi tao
   - Field: dung reflection sau khi tao
```

### Vi du qua trinh resolution
```java
// Co 2 bean DataSource
@Bean @Primary
public DataSource mysqlDS() { ... }

@Bean
public DataSource postgresDS() { ... }

// Case 1: Inject -> chon @Primary (mysqlDS)
@Autowired
private DataSource dataSource; // -> mysqlDS

// Case 2: Dung @Qualifier -> chon cu the
@Autowired
@Qualifier("postgresDS")
private DataSource dataSource; // -> postgresDS

// Case 3: Ten bien trung ten bean -> match theo ten
@Autowired
private DataSource postgresDS; // -> postgresDS (match by name)
```

---

## Q60: Best Practices for Spring Framework

### 1. Dependency Injection
```java
// DUNG: Constructor injection
@Service
public class UserService {
    private final UserRepository repo; // final = immutable
    public UserService(UserRepository repo) { this.repo = repo; }
}

// TRANH: Field injection
@Service
public class UserService {
    @Autowired // Kho test, khong final
    private UserRepository repo;
}
```

### 2. Layered Architecture
```
@Controller  -> Chi xu ly HTTP request/response
@Service     -> Business logic
@Repository  -> Data access
// KHONG goi Repository tu Controller truc tiep
```

### 3. Exception Handling
```java
// DUNG: Global handler
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(NotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handle(NotFoundException ex) { }
}

// TRANH: try-catch khap noi trong controller
```

### 4. Configuration
```java
// DUNG: Type-safe config
@ConfigurationProperties(prefix = "app")
public class AppProperties {
    private String name;
    private int timeout;
}

// TRANH: @Value khap noi
@Value("${app.name}") private String name;
@Value("${app.timeout}") private int timeout;
```

### 5. Transaction
```java
// DUNG: Service level, readOnly cho read
@Service
@Transactional(readOnly = true)
public class UserService {
    @Transactional
    public User create(UserDto dto) { }
}
```

### 6. Cac best practices khac
```
- Dung Spring Boot profiles cho moi truong (dev, staging, prod)
- Dung @Validated cho input validation
- Dung Spring Actuator cho monitoring
- Dung constructor injection (testable, immutable)
- Dung interface cho services (de mock khi test)
- Avoid circular dependencies
- Giu bean STATELESS (singleton)
- Log o muc phu hop (DEBUG cho dev, INFO cho prod)
```

## Diem quan trong nho phong van
1. Autowiring: **byType** -> @Primary -> @Qualifier -> **byName**
2. Best practice #1: **Constructor injection** (final, testable)
3. Best practice #2: **Layered architecture** (Controller -> Service -> Repository)
4. Best practice #3: **@Transactional** o Service, `readOnly=true` cho read
