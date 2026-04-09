# Q33: What is Spring WebFlux?
> **Dịch:** Spring WebFlux là gì?

## Trả lời ngắn gọn
> **Spring WebFlux** là reactive web framework của Spring 5, hỗ trợ **non-blocking I/O** và **backpressure**. Dựa trên Project Reactor với 2 kiểu dữ liệu: **Mono** (0-1 kết quả) và **Flux** (0-N kết quả).

## Cách nhớ
```
Spring MVC    = Nhà hàng truyền thống (1 bồi bàn / 1 bàn)
Spring WebFlux = Nhà hàng tự phục vụ  (1 bồi bàn / nhiều bàn)

MVC:     1 thread cho 1 request (đợi I/O thì thread ngồi không)
WebFlux: 1 thread xử lý nhiều request (không đợi, chuyển request khác)
```

## So sánh MVC vs WebFlux

| | Spring MVC | Spring WebFlux |
|--|:---:|:---:|
| Mô hình | Blocking | **Non-blocking** |
| Thread | 1 thread / request | Event loop (ít thread) |
| Servlet | Servlet API | Reactive Streams |
| Server | Tomcat, Jetty | **Netty** (mặc định), Tomcat |
| Kiểu trả về | Object, List | **Mono, Flux** |
| Use case | CRUD, đơn giản | High concurrency, streaming |

## Ví dụ code

### Controller style (annotation)
```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @Autowired
    private ReactiveUserRepository userRepo;

    // Trả về Mono (0-1 kết quả)
    @GetMapping("/{id}")
    public Mono<User> getUser(@PathVariable String id) {
        return userRepo.findById(id);
    }

    // Trả về Flux (0-N kết quả)
    @GetMapping
    public Flux<User> getAllUsers() {
        return userRepo.findAll();
    }

    // Save
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Mono<User> create(@RequestBody User user) {
        return userRepo.save(user);
    }

    // Streaming (Server-Sent Events)
    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<User> streamUsers() {
        return userRepo.findAll().delayElements(Duration.ofSeconds(1));
    }
}
```

### Router style (functional)
```java
@Configuration
public class RouterConfig {

    @Bean
    public RouterFunction<ServerResponse> routes(UserHandler handler) {
        return RouterFunctions.route()
            .GET("/api/users", handler::getAll)
            .GET("/api/users/{id}", handler::getById)
            .POST("/api/users", handler::create)
            .build();
    }
}

@Component
public class UserHandler {
    public Mono<ServerResponse> getAll(ServerRequest request) {
        return ServerResponse.ok().body(userRepo.findAll(), User.class);
    }
}
```

## Khi nào dùng WebFlux?

```
NÊN dùng:
- Ứng dụng cần xử lý NHIỀU request đồng thời (10k+ concurrent)
- Streaming data (real-time feed, chat)
- Microservices gọi nhiều API cùng lúc
- I/O intensive (nhiều gọi API, đọc file)

KHÔNG nên dùng:
- CRUD đơn giản
- Team chưa quen reactive programming
- Dùng JDBC (blocking) - cần R2DBC thay thế
- CPU intensive (tính toán nặng)
```

## Điểm quan trọng nhớ phỏng vấn
1. WebFlux = **Non-blocking**, dựa trên **Reactor** (Mono/Flux)
2. Chạy trên **Netty** mặc định (không phải Tomcat)
3. **KHÔNG** nhanh hơn MVC cho từng request, nhưng xử lý **nhiều request** tốt hơn
4. Cần dùng **R2DBC** thay JDBC (vì JDBC là blocking)
