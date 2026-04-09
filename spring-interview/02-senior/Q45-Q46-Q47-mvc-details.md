# Q45: How is an incoming request mapped to a controller and mapped to a method?
> **Dịch:** Request đến được ánh xạ (map) tới Controller và method như thế nào?

# Q46: What is Spring MVC Interceptor and how to use it?
> **Dịch:** Spring MVC Interceptor là gì và sử dụng như thế nào?

# Q47: What are some of the important Spring annotations you have used?
> **Dịch:** Kể tên một số annotation quan trọng trong Spring mà bạn đã dùng?

---

## Q45: Request Mapping Flow

### Trả lời ngắn gọn
> DispatcherServlet nhận request -> **HandlerMapping** tìm controller dựa trên `@RequestMapping` -> **HandlerAdapter** gọi method phù hợp -> Controller xử lý và trả về response.

```
GET /api/users/1
     |
     v
DispatcherServlet
     |
     v
HandlerMapping kiểm tra:
  @RequestMapping("/api/users") -> UserController
  @GetMapping("/{id}")          -> getUser(Long id)
     |
     v
HandlerAdapter gọi: UserController.getUser(1)
     |
     v
Response: {"id": 1, "name": "Thai"}
```

### Các annotation mapping

```java
@RestController
@RequestMapping("/api/users")   // Base path cho controller
public class UserController {

    @GetMapping("/{id}")         // GET /api/users/{id}
    @PostMapping                 // POST /api/users
    @PutMapping("/{id}")         // PUT /api/users/{id}
    @DeleteMapping("/{id}")      // DELETE /api/users/{id}
    @PatchMapping("/{id}")       // PATCH /api/users/{id}

    // Mapping theo nhiều tiêu chí
    @GetMapping(
        value = "/search",
        params = "name",                                 // Phải có param name
        headers = "X-API-KEY",                           // Phải có header
        produces = MediaType.APPLICATION_JSON_VALUE       // Response JSON
    )
    public List<User> search(@RequestParam String name) { }
}
```

---

## Q46: Spring MVC Interceptor

### Trả lời ngắn gọn
> **Interceptor** là component chèn vào luồng xử lý request **trước/sau** controller. Giống Filter nhưng hoạt động ở **Spring MVC level** (có truy cập Spring context).

```
Client -> Filter -> DispatcherServlet -> Interceptor -> Controller
                                              |
                                    preHandle() -> Controller -> postHandle() -> afterCompletion()
```

### Ví dụ: Logging Interceptor

```java
@Component
public class LoggingInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                             HttpServletResponse response,
                             Object handler) {
        System.out.println("BEFORE: " + request.getMethod() + " " + request.getRequestURI());
        request.setAttribute("startTime", System.currentTimeMillis());
        return true; // true = tiếp tục, false = dừng lại
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

// Đăng ký interceptor
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Autowired
    private LoggingInterceptor loggingInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(loggingInterceptor)
            .addPathPatterns("/api/**")        // Áp dụng cho /api/**
            .excludePathPatterns("/api/health"); // Bỏ qua health check
    }
}
```

### Filter vs Interceptor

| | Filter | Interceptor |
|--|:---:|:---:|
| Level | Servlet | Spring MVC |
| Access Spring beans | Khó | Dễ (là Spring bean) |
| Access handler info | Không | Có |
| Dùng cho | CORS, encoding, auth | Logging, audit, permission |

---

## Q47: Important Spring Annotations

### Các annotation quan trọng nhất

```java
// === CORE ===
@Component          // Đánh dấu bean chung
@Service            // Business layer
@Repository         // Data layer
@Controller         // Web layer (View)
@RestController     // Web layer (JSON)
@Configuration      // Java config class
@Bean               // Khai báo bean thủ công

// === DEPENDENCY INJECTION ===
@Autowired          // Tự động inject
@Qualifier("name")  // Chỉ định bean cụ thể
@Primary            // Bean ưu tiên
@Value("${key}")    // Inject giá trị từ properties

// === WEB ===
@RequestMapping     // Map URL
@GetMapping         // GET request
@PostMapping        // POST request
@PathVariable       // Lấy từ URL path
@RequestParam       // Lấy từ query string
@RequestBody        // Lấy từ request body (JSON)
@ResponseBody       // Trả về trực tiếp (JSON)
@ResponseStatus     // Set HTTP status code

// === DATA ===
@Transactional      // Quản lý transaction
@Entity             // JPA entity
@Table              // Tên bảng
@Id                 // Primary key

// === CONFIG ===
@SpringBootApplication  // Main class Spring Boot
@EnableAutoConfiguration
@ComponentScan
@Profile("dev")     // Active theo môi trường
@Conditional        // Bean có điều kiện

// === TEST ===
@SpringBootTest     // Integration test
@WebMvcTest         // Test MVC layer
@DataJpaTest        // Test JPA layer
@MockBean           // Mock bean trong test
```

## Điểm quan trọng nhớ phỏng vấn
1. Request mapping: **DispatcherServlet -> HandlerMapping -> Controller**
2. Interceptor hoạt động ở **Spring MVC level**, Filter ở **Servlet level**
3. Interceptor có truy cập **handler info** và **Spring context**
4. Nhớ các nhóm annotation: **Core, DI, Web, Data, Config, Test**
