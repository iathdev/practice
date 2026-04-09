# Q33: What is Spring WebFlux?
> **Dịch:** Spring WebFlux là gì?

## Tra loi ngan gon
> **Spring WebFlux** la reactive web framework cua Spring 5, ho tro **non-blocking I/O** va **backpressure**. Dua tren Project Reactor voi 2 kieu du lieu: **Mono** (0-1 ket qua) va **Flux** (0-N ket qua).

## Cach nho
```
Spring MVC    = Nha hang truyen thong (1 boi ban / 1 ban)
Spring WebFlux = Nha hang tu phuc vu  (1 boi ban / nhieu ban)

MVC:     1 thread cho 1 request (doi I/O thi thread ngoi khong)
WebFlux: 1 thread xu ly nhieu request (khong doi, chuyen request khac)
```

## So sanh MVC vs WebFlux

| | Spring MVC | Spring WebFlux |
|--|:---:|:---:|
| Mo hinh | Blocking | **Non-blocking** |
| Thread | 1 thread / request | Event loop (it thread) |
| Servlet | Servlet API | Reactive Streams |
| Server | Tomcat, Jetty | **Netty** (mac dinh), Tomcat |
| Kieu tra ve | Object, List | **Mono, Flux** |
| Use case | CRUD, don gian | High concurrency, streaming |

## Vi du code

### Controller style (annotation)
```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @Autowired
    private ReactiveUserRepository userRepo;

    // Tra ve Mono (0-1 ket qua)
    @GetMapping("/{id}")
    public Mono<User> getUser(@PathVariable String id) {
        return userRepo.findById(id);
    }

    // Tra ve Flux (0-N ket qua)
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

## Khi nao dung WebFlux?

```
NEN dung:
- Ung dung can xu ly NHIEU request dong thoi (10k+ concurrent)
- Streaming data (real-time feed, chat)
- Microservices goi nhieu API cung luc
- I/O intensive (nhieu goi API, doc file)

KHONG nen dung:
- CRUD don gian
- Team chua quen reactive programming
- Dung JDBC (blocking) - can R2DBC thay the
- CPU intensive (tinh toan nang)
```

## Diem quan trong nho phong van
1. WebFlux = **Non-blocking**, dua tren **Reactor** (Mono/Flux)
2. Chay tren **Netty** mac dinh (khong phai Tomcat)
3. **KHONG** nhanh hon MVC cho tung request, nhung xu ly **nhieu request** tot hon
4. Can dung **R2DBC** thay JDBC (vi JDBC la blocking)
