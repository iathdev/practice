# 🍃 Spring Framework Interview Q&A — Senior Developer

> 50+ Câu hỏi phỏng vấn Spring cho Senior Developer — Có ví dụ cụ thể

**Mục lục:** Spring Core · IoC & DI · Spring Beans · AOP · Spring MVC · Spring Boot · Data Access · Security · WebFlux · Best Practices

---

## 1. Spring Framework Core `MUST KNOW`

### Q1. Spring Framework là gì?

Spring là **open-source Java EE framework** được sử dụng rộng rãi nhất. Hai concept cốt lõi:
- **Dependency Injection (DI)** — Tiêm phụ thuộc
- **Aspect-Oriented Programming (AOP)** — Lập trình hướng khía cạnh

Spring giúp xây dựng ứng dụng Java với loose coupling giữa các components, dễ test và maintain.

**Các modules chính:**

| Module | Chức năng |
|---|---|
| Spring Core | IoC container, DI |
| Spring AOP | Cross-cutting concerns |
| Spring MVC | Web applications, REST |
| Spring Data | JPA, JDBC, MongoDB |
| Spring Security | Authentication, Authorization |
| Spring Boot | Auto-configuration |

---

### Q2. Các features quan trọng của Spring Framework?

- **Lightweight:** Rất ít overhead khi sử dụng framework
- **Dependency Injection:** Components độc lập, Spring container quản lý wiring
- **IoC Container:** Quản lý Spring Bean lifecycle và configurations
- **MVC Framework:** Tạo web applications và RESTful web services
- **Transaction Management:** Hỗ trợ JDBC, exception handling với ít config
- **Modular:** Chỉ dùng modules cần thiết, không cần load tất cả

---

### Q3. Ưu điểm của việc sử dụng Spring Framework?

| Ưu điểm | Mô tả |
|---|---|
| Giảm dependencies trực tiếp | IoC container quản lý khởi tạo và inject dependencies |
| Dễ viết Unit Test | Business logic không phụ thuộc trực tiếp vào implementation |
| Giảm boilerplate code | JdbcTemplate, RestTemplate loại bỏ code lặp lại |
| Modular design | Chỉ add dependency cần thiết |
| Complete package | Hỗ trợ từ web đến mobile (Android) |

---

### Q4. Các features mới của Spring 5?

- **Java 8+ support:** Lambda expressions, Optional, Stream API
- Java EE 7 & Servlet 4.0 specs
- **NIO 2 streams:** Cải thiện file operations
- **spring-jcl:** Logging abstraction thống nhất
- **Kotlin support**
- **Reactor 3.1 Flux & Mono, RxJava**
- **Spring WebFlux:** Reactive programming
- **JUnit 5 support**
- **Component index:** `META-INF/spring.components` thay vì classpath scanning

---

## 2. Inversion of Control (IoC) & Dependency Injection (DI) `CRITICAL`

### Q5. Dependency Injection là gì?

**Dependency Injection** là design pattern cho phép loại bỏ hard-coded dependencies, làm ứng dụng loose-coupled, extensible và maintainable.

Thay vì object tự tạo dependencies, chúng được **inject từ bên ngoài** (container).

**Lợi ích:** Separation of Concerns · Giảm boilerplate code · Configurable components · Dễ unit testing

```java
// ❌ Hard-coded dependency
public class OrderService {
    private PaymentService payment = new CreditCardPayment(); // Tight coupling!
}

// ✅ Dependency Injection
public class OrderService {
    private final PaymentService payment;

    public OrderService(PaymentService payment) { // Injected!
        this.payment = payment;
    }
}
```

---

### Q6. Spring IoC Container là gì?

**Inversion of Control (IoC)** là cơ chế để đạt được loose-coupling giữa object dependencies. Thay vì object tự quản lý dependencies, container sẽ inject chúng.

**Hai loại IoC Container:**

| BeanFactory | ApplicationContext |
|---|---|
| Basic container | Full-featured container |
| Lazy loading | Eager loading (default) |
| Lightweight | Event publishing, i18n, resource loading |

```java
// Standalone với annotations
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);

// Standalone với XML
ApplicationContext ctx = new ClassPathXmlApplicationContext("spring-config.xml");

// File system XML
ApplicationContext ctx = new FileSystemXmlApplicationContext("/path/to/config.xml");

// Web: AnnotationConfigWebApplicationContext hoặc XmlWebApplicationContext
```

---

### Q7. Làm thế nào để implement DI trong Spring? `HOT`

**1. Constructor Injection (RECOMMENDED)**

```java
@Service
public class UserService {
    private final UserRepository userRepository;
    private final EmailService emailService;

    // @Autowired optional cho single constructor (Spring 4.3+)
    public UserService(UserRepository userRepository, EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }
}
```

**2. Setter Injection**

```java
@Service
public class NotificationService {
    private SMSService smsService;

    @Autowired(required = false) // optional dependency
    public void setSmsService(SMSService smsService) {
        this.smsService = smsService;
    }
}
```

**3. Field Injection (NOT Recommended)**

```java
@Service
public class BadService {
    @Autowired // ❌ Hard to test, hidden dependencies
    private UserRepository userRepository;
}
```

> **Tại sao Constructor Injection tốt hơn?**
> - Dependencies rõ ràng, visible
> - Immutable (final fields)
> - Dễ test (pass mocks via constructor)
> - Fail fast nếu thiếu dependency

---

### Q8. Các loại autowiring trong Spring?

| Type | Description |
|---|---|
| no (default) | Không autowire, phải explicit wire |
| byName | Autowire by property name |
| byType | Autowire by property type |
| constructor | Autowire by constructor argument type |

```java
// Multiple implementations - dùng @Qualifier
public interface PaymentService { }

@Service("creditCard")
public class CreditCardPayment implements PaymentService { }

@Service("paypal")
@Primary  // Default khi không có @Qualifier
public class PayPalPayment implements PaymentService { }

@Service
public class OrderService {
    public OrderService(@Qualifier("creditCard") PaymentService payment) {
        // Gets CreditCardPayment
    }
}
```

---

## 3. Spring Beans `CRITICAL`

### Q9. Spring Bean là gì?

**Spring Bean** là bất kỳ Java class nào được khởi tạo và quản lý bởi Spring IoC container.
- Container quản lý lifecycle của bean
- Container inject dependencies
- Bean được lấy qua `ApplicationContext.getBean()`

---

### Q10. Các cách cấu hình Spring Bean? `HOT`

**1. XML Configuration**

```xml
<!-- spring-config.xml -->
<bean name="userService" class="com.example.UserService">
    <property name="userRepository" ref="userRepository"/>
</bean>
```

**2. Java Configuration (Recommended)**

```java
@Configuration
@ComponentScan("com.example")
public class AppConfig {

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    @Bean
    @Profile("dev")
    public DataSource devDataSource() {
        return new H2DataSource();
    }
}
```

**3. Annotation-based Configuration**

```java
@Component       // Generic component
@Service         // Business logic layer
@Repository      // Data access layer
@Controller      // MVC controller
@RestController  // REST API controller
```

---

### Q11. Các loại Bean Scope trong Spring?

| Scope | Description | Use Case |
|---|---|---|
| singleton (default) | Một instance duy nhất cho mỗi container | Stateless services |
| prototype | Instance mới mỗi lần request | Stateful beans |
| request | Một instance cho mỗi HTTP request | Web: request data |
| session | Một instance cho mỗi HTTP session | Web: user session |
| application | Một instance cho mỗi ServletContext | Web: app-wide |

```java
@Component
@Scope("prototype")  // New instance each time
public class ShoppingCart { }

@Component
@RequestScope  // One per HTTP request
public class RequestContext { }

@Component
@SessionScope  // One per user session
public class UserSession { }
```

---

### Q12. Spring Bean Lifecycle? `HOT`

```
1.  Bean Instantiation (Constructor)
2.  Populate Properties (DI)
3.  BeanNameAware.setBeanName()
4.  BeanFactoryAware.setBeanFactory()
5.  ApplicationContextAware.setApplicationContext()
6.  BeanPostProcessor.postProcessBeforeInitialization()
7.  @PostConstruct method
8.  InitializingBean.afterPropertiesSet()
9.  Custom init-method
10. BeanPostProcessor.postProcessAfterInitialization()
    ═══════════ BEAN READY ═══════════
11. @PreDestroy method
12. DisposableBean.destroy()
13. Custom destroy-method
```

```java
@Component
public class MyBean {

    @PostConstruct
    public void init() {
        System.out.println("Bean initialized");
    }

    @PreDestroy
    public void cleanup() {
        System.out.println("Bean destroyed");
    }
}
```

---

### Q13. Spring Bean có thread-safe không?

**Không!** Default scope là singleton — một instance cho tất cả threads.

```java
// ❌ Thread-safety issue
@Service
public class BadService {
    private User currentUser;  // Shared mutable state!

    public void setUser(User user) {
        this.currentUser = user;  // Race condition!
    }
}
```

**Solutions:**
- Thiết kế beans là **stateless** (không có mutable instance variables)
- Dùng `@Scope("prototype")` hoặc `@RequestScope`
- Dùng `ThreadLocal` cho thread-specific data
- Dùng synchronized hoặc concurrent collections

---

### Q14. @Component vs @Controller vs @Repository vs @Service?

| Annotation | Layer | Special Behavior |
|---|---|---|
| @Component | Generic | Base annotation |
| @Service | Business Logic | Semantic only (no special behavior) |
| @Repository | Data Access | Exception translation (PersistenceExceptionTranslationPostProcessor) |
| @Controller | Presentation | Request handling, view resolution |
| @RestController | REST API | @Controller + @ResponseBody |

```java
@Repository  // Translates SQLException → DataAccessException
public class UserRepository {
    public User findById(Long id) { ... }
}

@Service  // Business logic
public class UserService {
    private final UserRepository userRepository;
    ...
}

@RestController  // REST API
@RequestMapping("/api/users")
public class UserController {
    private final UserService userService;
    ...
}
```

---

## 4. Aspect-Oriented Programming (AOP)

### Q15. AOP là gì? Tại sao cần AOP?

**Aspect-Oriented Programming** là paradigm để xử lý **cross-cutting concerns** — các tính năng cắt ngang nhiều modules:
- Logging
- Transaction management
- Security / Authentication
- Caching
- Performance monitoring

AOP cho phép tách biệt cross-cutting concerns khỏi business logic, tránh code lặp lại.

---

### Q16. Các khái niệm trong AOP? `HOT`

| Concept | Description |
|---|---|
| **Aspect** | Class chứa cross-cutting concerns (annotated với `@Aspect`) |
| **Join Point** | Điểm trong code execution (method call, exception thrown...) |
| **Pointcut** | Expression để match join points |
| **Advice** | Action thực hiện tại join point (`@Before`, `@After`, `@Around`...) |
| **Weaving** | Process link aspects với target objects |

```java
@Aspect
@Component
public class LoggingAspect {

    // Pointcut: match all methods in service package
    @Pointcut("execution(* com.example.service.*.*(..))")
    public void serviceLayer() {}

    // Before
    @Before("serviceLayer()")
    public void logBefore(JoinPoint jp) {
        log.info("Calling: {}", jp.getSignature().getName());
    }

    // After Returning
    @AfterReturning(pointcut = "serviceLayer()", returning = "result")
    public void logAfter(JoinPoint jp, Object result) {
        log.info("Method {} returned: {}", jp.getSignature().getName(), result);
    }

    // Around (most powerful)
    @Around("serviceLayer()")
    public Object measureTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.currentTimeMillis();
        Object result = pjp.proceed();  // Execute method
        long duration = System.currentTimeMillis() - start;
        log.info("Method {} took {} ms", pjp.getSignature().getName(), duration);
        return result;
    }

    // After Throwing
    @AfterThrowing(pointcut = "serviceLayer()", throwing = "ex")
    public void logException(JoinPoint jp, Exception ex) {
        log.error("Exception in {}: {}", jp.getSignature().getName(), ex.getMessage());
    }
}
```

---

### Q17. Spring AOP vs AspectJ AOP?

| Aspect | Spring AOP | AspectJ |
|---|---|---|
| Weaving | Runtime (proxy-based) | Compile-time, Load-time, Runtime |
| Join Points | Method execution only | Method, constructor, field access... |
| Performance | Slightly slower (proxies) | Faster (bytecode modification) |
| Complexity | Simple, no special compiler | Requires AspectJ compiler |
| Scope | Spring beans only | Any Java class |

> **Khi nào dùng gì?**
> - **Spring AOP:** Đủ cho hầu hết cases (logging, transactions, security)
> - **AspectJ:** Khi cần aspect cho non-Spring objects hoặc field access

---

## 5. Spring MVC `CRITICAL`

### Q18. Spring MVC là gì? Các components chính?

Spring MVC là web framework theo **Model-View-Controller** pattern.

**Components:**
- **DispatcherServlet:** Front controller, nhận tất cả requests
- **HandlerMapping:** Map request URL → Controller
- **Controller:** Xử lý request, return ModelAndView
- **ViewResolver:** Resolve view name → actual view
- **View:** Render response (JSP, Thymeleaf, JSON...)

---

### Q19. Request Flow trong Spring MVC? `HOT`

```
┌─────────┐    ┌──────────────────┐    ┌───────────────┐
│ Client  │───>│ DispatcherServlet│───>│ HandlerMapping│
└─────────┘    └──────────────────┘    └───────────────┘
                       │                       │
                       │<──────────────────────┘
                       ↓                Handler (Controller)
               ┌───────────────┐
               │  Controller   │
               └───────────────┘
                       │ ModelAndView
               ┌───────────────┐
               │ ViewResolver  │
               └───────────────┘
                       │ View
               ┌───────────────┐
               │    Client     │
               └───────────────┘
```

---

### Q20. DispatcherServlet và ContextLoaderListener?

**DispatcherServlet:**
- Front controller trong Spring MVC
- Load spring bean config và initialize beans
- Scan packages cho `@Component`, `@Controller`, `@Service`, `@Repository`
- Route requests đến appropriate controller

**ContextLoaderListener:**
- Listener để start/shutdown Spring's root WebApplicationContext
- Tie lifecycle của ApplicationContext với ServletContext
- Define shared beans across different spring contexts

---

### Q21. Các annotations quan trọng trong Spring MVC?

```java
@Controller           // Mark as MVC controller
@RestController       // @Controller + @ResponseBody
@RequestMapping       // Map URL to handler method
@GetMapping           // HTTP GET
@PostMapping          // HTTP POST
@PutMapping           // HTTP PUT
@DeleteMapping        // HTTP DELETE
@PathVariable         // Extract from URL path
@RequestParam         // Extract from query params
@RequestBody          // Bind request body to object
@ResponseBody         // Write return value to response body
@ResponseStatus       // Set HTTP status code
@ModelAttribute       // Bind request params to model object
@SessionAttribute     // Store in session
```

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }

    @GetMapping
    public List<User> search(
            @RequestParam(defaultValue = "") String name,
            @RequestParam(defaultValue = "0") int page) {
        return userService.search(name, page);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public User create(@Valid @RequestBody CreateUserRequest request) {
        return userService.create(request);
    }
}
```

---

### Q22. Exception Handling trong Spring MVC?

**1. @ExceptionHandler (Controller level)**

```java
@Controller
public class UserController {

    @ExceptionHandler(UserNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(UserNotFoundException ex) {
        return new ErrorResponse(ex.getMessage());
    }
}
```

**2. @ControllerAdvice (Global)**

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ResourceNotFoundException ex) {
        return new ErrorResponse("NOT_FOUND", ex.getMessage());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .collect(Collectors.toMap(
                FieldError::getField,
                FieldError::getDefaultMessage
            ));
        return new ErrorResponse("VALIDATION_ERROR", errors);
    }
}
```

**3. HandlerExceptionResolver (Programmatic):** Implement interface để custom exception handling logic.

---

### Q23. Spring MVC Interceptor là gì?

Interceptors cho phép **intercept requests** tại 3 điểm:
- **preHandle:** Trước khi controller xử lý
- **postHandle:** Sau controller, trước view render
- **afterCompletion:** Sau khi response được gửi

```java
@Component
public class LoggingInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) {
        log.info("Request: {} {}", request.getMethod(), request.getRequestURI());
        request.setAttribute("startTime", System.currentTimeMillis());
        return true;  // Continue to controller
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                                HttpServletResponse response,
                                Object handler, Exception ex) {
        long startTime = (long) request.getAttribute("startTime");
        log.info("Request took: {} ms", System.currentTimeMillis() - startTime);
    }
}

@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Autowired
    private LoggingInterceptor loggingInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loggingInterceptor)
                .addPathPatterns("/api/**")
                .excludePathPatterns("/api/health");
    }
}
```

---

## 6. Spring Boot `MUST KNOW`

### Q24. Spring Boot là gì?

Spring Boot đơn giản hóa việc tạo Spring applications với:
- **Auto-configuration:** Tự động configure dựa trên classpath
- **Starter dependencies:** Gom nhóm dependencies phổ biến
- **Embedded server:** Tomcat, Jetty, Undertow built-in
- **Production-ready:** Actuator, metrics, health checks
- **No XML:** Hoàn toàn annotation-based

---

### Q25. @SpringBootApplication annotation?

```java
@SpringBootApplication
// = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

| Annotation | Purpose |
|---|---|
| @Configuration | Mark as configuration class |
| @EnableAutoConfiguration | Enable auto-config based on classpath |
| @ComponentScan | Scan current package and sub-packages |

---

### Q26. Spring vs Spring Boot?

| Aspect | Spring | Spring Boot |
|---|---|---|
| Configuration | Extensive XML/Java config | Auto-configuration |
| Server | External server required | Embedded server |
| Deployment | WAR file | JAR (executable) |
| Dependencies | Manual management | Starter POMs |
| Production | Manual setup | Actuator built-in |

---

### Q27. Spring Boot Actuator?

Actuator cung cấp production-ready features:

| Endpoint | Description |
|---|---|
| /actuator/health | Application health status |
| /actuator/info | Application info |
| /actuator/metrics | Application metrics |
| /actuator/env | Environment properties |
| /actuator/loggers | View/modify log levels |
| /actuator/threaddump | Thread dump |
| /actuator/prometheus | Prometheus format metrics |

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus
  endpoint:
    health:
      show-details: when_authorized
```

---

### Q28. Profiles trong Spring Boot?

```yaml
# application-dev.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb

# application-prod.yml
spring:
  datasource:
    url: jdbc:mysql://prod-server:3306/mydb
```

```bash
# Activate profile
java -jar app.jar --spring.profiles.active=prod

# Or via environment variable
SPRING_PROFILES_ACTIVE=prod java -jar app.jar
```

```java
@Configuration
@Profile("dev")
public class DevConfig {
    @Bean
    public DataSource dataSource() {
        return new H2DataSource();
    }
}
```

---

## 7. Data Access (JDBC, JPA, Hibernate)

### Q29. Spring JdbcTemplate là gì?

JdbcTemplate đơn giản hóa JDBC operations, loại bỏ boilerplate code: tự động open/close connections, handle exceptions, execute queries với parameters, map results to objects.

```java
@Repository
public class UserRepository {

    private final JdbcTemplate jdbcTemplate;

    public UserRepository(JdbcTemplate jdbcTemplate) {
        this.jdbcTemplate = jdbcTemplate;
    }

    public List<User> findAll() {
        return jdbcTemplate.query(
            "SELECT * FROM users",
            new BeanPropertyRowMapper<>(User.class)
        );
    }

    public User findById(Long id) {
        return jdbcTemplate.queryForObject(
            "SELECT * FROM users WHERE id = ?",
            new BeanPropertyRowMapper<>(User.class),
            id
        );
    }

    public int save(User user) {
        return jdbcTemplate.update(
            "INSERT INTO users (name, email) VALUES (?, ?)",
            user.getName(),
            user.getEmail()
        );
    }
}
```

---

### Q30. Transaction Management trong Spring? `HOT`

**Declarative Transaction (Recommended)**

```java
@Service
@Transactional(readOnly = true)  // Class level default
public class AccountService {

    public Account findById(Long id) {
        return accountRepository.findById(id).orElseThrow();
    }

    @Transactional  // Override: read-write
    public void transfer(Long from, Long to, BigDecimal amount) {
        Account source = accountRepository.findById(from).orElseThrow();
        Account target = accountRepository.findById(to).orElseThrow();

        source.debit(amount);
        target.credit(amount);

        accountRepository.save(source);
        accountRepository.save(target);
        // Nếu exception → rollback cả 2 operations
    }

    @Transactional(
        propagation = Propagation.REQUIRED,
        isolation = Isolation.READ_COMMITTED,
        timeout = 30,
        rollbackFor = BusinessException.class
    )
    public void complexOperation() { ... }
}
```

**Transaction Propagation:**

| Propagation | Behavior |
|---|---|
| REQUIRED (default) | Join existing or create new |
| REQUIRES_NEW | Always create new (suspend existing) |
| NESTED | Nested transaction (savepoint) |
| SUPPORTS | Join if exists, else non-transactional |
| MANDATORY | Must have existing, else exception |

---

### Q31. Spring Data JPA?

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // Query methods - Spring generates implementation!
    Optional<User> findByEmail(String email);
    List<User> findByNameContainingIgnoreCase(String name);
    List<User> findByStatusAndCreatedAtAfter(Status status, LocalDateTime date);

    // JPQL
    @Query("SELECT u FROM User u WHERE u.status = :status")
    List<User> findActiveUsers(@Param("status") Status status);

    // Native query
    @Query(value = "SELECT * FROM users WHERE email LIKE %?1%", nativeQuery = true)
    List<User> searchByEmail(String email);

    // Modifying
    @Modifying
    @Query("UPDATE User u SET u.status = :status WHERE u.id = :id")
    int updateStatus(@Param("id") Long id, @Param("status") Status status);
}
```

---

## 8. Spring Security

### Q32. Spring Security là gì?

Spring Security cung cấp comprehensive security cho Java applications:
- **Authentication:** Xác thực user (who are you?)
- **Authorization:** Phân quyền (what can you do?)
- **CSRF protection**
- **Session management**
- **Password encoding**

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .httpBasic(Customizer.withDefaults())
            .build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

---

## 9. Spring WebFlux (Reactive)

### Q33. Spring WebFlux là gì?

Spring WebFlux là reactive web framework, alternative cho Spring MVC:
- **Non-blocking I/O**
- **Reactive Streams** (Publisher/Subscriber)
- **Backpressure handling**
- Better cho **high-concurrency** scenarios

**Core types:**

| Type | Description |
|---|---|
| `Mono<T>` | 0 or 1 element |
| `Flux<T>` | 0 to N elements |

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public Mono<User> getUser(@PathVariable Long id) {
        return userService.findById(id);
    }

    @GetMapping
    public Flux<User> getAllUsers() {
        return userService.findAll();
    }

    @PostMapping
    public Mono<User> createUser(@RequestBody Mono<User> user) {
        return user.flatMap(userService::save);
    }
}
```

---

### Q34. WebClient là gì?

WebClient là non-blocking HTTP client, thay thế RestTemplate:

```java
@Service
public class UserClient {

    private final WebClient webClient;

    public UserClient(WebClient.Builder builder) {
        this.webClient = builder
            .baseUrl("http://user-service")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .build();
    }

    public Mono<User> getUser(Long id) {
        return webClient.get()
            .uri("/users/{id}", id)
            .retrieve()
            .bodyToMono(User.class)
            .onErrorResume(e -> Mono.empty());
    }

    public Flux<User> getAllUsers() {
        return webClient.get()
            .uri("/users")
            .retrieve()
            .bodyToFlux(User.class);
    }
}
```

---

## 10. Best Practices & Design Patterns

### Q35. Best Practices khi sử dụng Spring?

- **Constructor Injection:** Prefer over field injection
- **Avoid version numbers in schema:** Use latest configs
- **Divide configuration:** spring-jdbc.xml, spring-security.xml
- **Use correct annotations:** `@Service` for business, `@Repository` for DAO
- **Keep aspects narrow:** Avoid advice on unwanted methods
- **External properties:** Use property files, not hardcode
- **Use only needed modules:** Remove unused dependencies
- **Use @Transactional properly:** `readOnly=true` for queries
- **Stateless beans:** Avoid mutable instance variables in singleton

---

### Q36. Design Patterns trong Spring?

| Pattern | Usage trong Spring |
|---|---|
| Singleton | Default bean scope |
| Factory | BeanFactory, ApplicationContext |
| Prototype | Prototype bean scope |
| Proxy | AOP, @Transactional |
| Template Method | JdbcTemplate, RestTemplate |
| Front Controller | DispatcherServlet |
| DAO | Spring Data repositories |
| Dependency Injection | Core Spring concept |
| Observer | ApplicationEvent, @EventListener |

---

### Q37. Inject Properties vào Bean?

```properties
# application.properties
app.name=MyApplication
app.max-connections=100
```

```java
// Method 1: @Value
@Service
public class MyService {
    @Value("${app.name}")
    private String appName;

    @Value("${app.max-connections:50}")  // Default value
    private int maxConnections;
}

// Method 2: @ConfigurationProperties (type-safe, RECOMMENDED)
@Configuration
@ConfigurationProperties(prefix = "app")
public class AppConfig {
    private String name;
    private int maxConnections;

    // Getters and setters
}
```

---

## 📚 Quick Reference Card

**Core Annotations:**
`@Component` · `@Service` · `@Repository` · `@Controller` · `@Configuration` · `@Bean`

**DI Annotations:**
`@Autowired` · `@Qualifier` · `@Primary` · `@Value` · `@Scope`

**Web Annotations:**
`@RestController` · `@RequestMapping` · `@GetMapping` · `@PostMapping` · `@PathVariable` · `@RequestBody`

**Data Annotations:**
`@Entity` · `@Table` · `@Id` · `@GeneratedValue` · `@Transactional` · `@Query`

**AOP Annotations:**
`@Aspect` · `@Pointcut` · `@Before` · `@After` · `@Around`

**Test Annotations:**
`@SpringBootTest` · `@WebMvcTest` · `@DataJpaTest` · `@MockBean` · `@AutoConfigureMockMvc`

---

*Spring Framework Interview Q&A — Senior Developer Guide*
*Nguồn tham khảo: DigitalOcean, GeeksforGeeks, Spring Official Docs*
