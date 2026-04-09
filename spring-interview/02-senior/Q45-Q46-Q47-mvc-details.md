# Q45: How is an incoming request mapped to a controller and mapped to a method?
> **Dịch:** Request đến được ánh xạ (map) tới Controller và method như thế nào?

# Q46: What is Spring MVC Interceptor and how to use it?
> **Dịch:** Spring MVC Interceptor là gì và sử dụng như thế nào?

# Q47: What are some of the important Spring annotations you have used?
> **Dịch:** Kể tên một số annotation quan trọng trong Spring mà bạn đã dùng?

---

## Q45: Request Mapping Flow

### Tra loi ngan gon
> DispatcherServlet nhan request -> **HandlerMapping** tim controller dua tren `@RequestMapping` -> **HandlerAdapter** goi method phu hop -> Controller xu ly va tra ve response.

```
GET /api/users/1
     |
     v
DispatcherServlet
     |
     v
HandlerMapping kiem tra:
  @RequestMapping("/api/users") -> UserController
  @GetMapping("/{id}")          -> getUser(Long id)
     |
     v
HandlerAdapter goi: UserController.getUser(1)
     |
     v
Response: {"id": 1, "name": "Thai"}
```

### Cac annotation mapping

```java
@RestController
@RequestMapping("/api/users")   // Base path cho controller
public class UserController {

    @GetMapping("/{id}")         // GET /api/users/{id}
    @PostMapping                 // POST /api/users
    @PutMapping("/{id}")         // PUT /api/users/{id}
    @DeleteMapping("/{id}")      // DELETE /api/users/{id}
    @PatchMapping("/{id}")       // PATCH /api/users/{id}

    // Mapping theo nhieu tieu chi
    @GetMapping(
        value = "/search",
        params = "name",                                 // Phai co param name
        headers = "X-API-KEY",                           // Phai co header
        produces = MediaType.APPLICATION_JSON_VALUE       // Response JSON
    )
    public List<User> search(@RequestParam String name) { }
}
```

---

## Q46: Spring MVC Interceptor

### Tra loi ngan gon
> **Interceptor** la component chen vao luong xu ly request **truoc/sau** controller. Giong Filter nhung hoat dong o **Spring MVC level** (co truy cap Spring context).

```
Client -> Filter -> DispatcherServlet -> Interceptor -> Controller
                                              |
                                    preHandle() -> Controller -> postHandle() -> afterCompletion()
```

### Vi du: Logging Interceptor

```java
@Component
public class LoggingInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) {
        System.out.println("BEFORE: " + request.getMethod() + " " + request.getRequestURI());
        request.setAttribute("startTime", System.currentTimeMillis());
        return true; // true = tiep tuc, false = dung lai
    }

    @Override
    public void postHandle(HttpServletRequest request,
                           HttpServletResponse response,
                           Object handler, ModelAndView modelAndView) {
        System.out.println("AFTER controller, BEFORE view render");
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                                HttpServletResponse response,
                                Object handler, Exception ex) {
        long start = (long) request.getAttribute("startTime");
        System.out.println("COMPLETE: " + (System.currentTimeMillis() - start) + "ms");
    }
}

// Dang ky interceptor
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Autowired
    private LoggingInterceptor loggingInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loggingInterceptor)
            .addPathPatterns("/api/**")        // Ap dung cho /api/**
            .excludePathPatterns("/api/health"); // Bo qua health check
    }
}
```

### Filter vs Interceptor

| | Filter | Interceptor |
|--|:---:|:---:|
| Level | Servlet | Spring MVC |
| Access Spring beans | Kho | De (la Spring bean) |
| Access handler info | Khong | Co |
| Dung cho | CORS, encoding, auth | Logging, audit, permission |

---

## Q47: Important Spring Annotations

### Cac annotation quan trong nhat

```java
// === CORE ===
@Component          // Danh dau bean chung
@Service            // Business layer
@Repository         // Data layer
@Controller         // Web layer (View)
@RestController     // Web layer (JSON)
@Configuration      // Java config class
@Bean               // Khai bao bean thu cong

// === DEPENDENCY INJECTION ===
@Autowired          // Tu dong inject
@Qualifier("name")  // Chi dinh bean cu the
@Primary            // Bean uu tien
@Value("${key}")    // Inject gia tri tu properties

// === WEB ===
@RequestMapping     // Map URL
@GetMapping         // GET request
@PostMapping        // POST request
@PathVariable       // Lay tu URL path
@RequestParam       // Lay tu query string
@RequestBody        // Lay tu request body (JSON)
@ResponseBody       // Tra ve truc tiep (JSON)
@ResponseStatus     // Set HTTP status code

// === DATA ===
@Transactional      // Quan ly transaction
@Entity             // JPA entity
@Table              // Ten bang
@Id                 // Primary key

// === CONFIG ===
@SpringBootApplication  // Main class Spring Boot
@EnableAutoConfiguration
@ComponentScan
@Profile("dev")     // Active theo moi truong
@Conditional        // Bean co dieu kien

// === TEST ===
@SpringBootTest     // Integration test
@WebMvcTest         // Test MVC layer
@DataJpaTest        // Test JPA layer
@MockBean           // Mock bean trong test
```

## Diem quan trong nho phong van
1. Request mapping: **DispatcherServlet -> HandlerMapping -> Controller**
2. Interceptor hoat dong o **Spring MVC level**, Filter o **Servlet level**
3. Interceptor co truy cap **handler info** va **Spring context**
4. Nho cac nhom annotation: **Core, DI, Web, Data, Config, Test**
