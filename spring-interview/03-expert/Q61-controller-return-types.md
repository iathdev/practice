# Q61: What are some of the valid return types of a controller method?
> **Dịch:** Các kiểu trả về hợp lệ của method trong Controller là gì?

## Trả lời ngắn gọn
> Controller method có thể trả về nhiều kiểu: **String** (view name), **ModelAndView**, **ResponseEntity**, **Object** (JSON), **void**, **Mono/Flux** (reactive), **DeferredResult** (async).

## Các kiểu trả về

### 1. String - Tên View
```java
@Controller
public class PageController {
    @GetMapping("/home")
    public String home(Model model) {
        model.addAttribute("name", "Thai");
        return "home";              // -> templates/home.html
        // return "redirect:/login"; // -> redirect
        // return "forward:/other";  // -> forward
    }
}
```

### 2. ModelAndView - View + Data
```java
@GetMapping("/users")
public ModelAndView users() {
    ModelAndView mav = new ModelAndView("userList");
    mav.addObject("users", userService.findAll());
    mav.setStatus(HttpStatus.OK);
    return mav;
}
```

### 3. ResponseEntity<T> - HTTP Response đầy đủ (THƯỜNG DÙNG NHẤT cho REST)
```java
@GetMapping("/users/{id}")
public ResponseEntity<User> getUser(@PathVariable Long id) {
    return userService.findById(id)
        .map(user -> ResponseEntity.ok()
            .header("X-Custom", "value")
            .body(user))
        .orElse(ResponseEntity.notFound().build());
}

// Các cách tạo ResponseEntity:
ResponseEntity.ok(body);                    // 200
ResponseEntity.created(uri).body(body);     // 201
ResponseEntity.noContent().build();         // 204
ResponseEntity.badRequest().body(error);    // 400
ResponseEntity.notFound().build();          // 404
ResponseEntity.status(HttpStatus.I_AM_A_TEAPOT).build(); // 418
```

### 4. Object trực tiếp - Tự động serialize JSON
```java
@RestController
public class ApiController {
    @GetMapping("/users")
    public List<User> getUsers() {
        return userService.findAll(); // -> JSON array
    }

    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id); // -> JSON object
    }
}
```

### 5. void - Không trả về gì
```java
@DeleteMapping("/users/{id}")
@ResponseStatus(HttpStatus.NO_CONTENT) // 204
public void deleteUser(@PathVariable Long id) {
    userService.delete(id); // Không cần trả về gì
}
```

### 6. Mono / Flux (Reactive)
```java
@GetMapping("/users/{id}")
public Mono<User> getUser(@PathVariable String id) {
    return userService.findById(id);
}

@GetMapping("/users")
public Flux<User> getUsers() {
    return userService.findAll();
}
```

### 7. DeferredResult / Callable (Async)
```java
@GetMapping("/async")
public DeferredResult<String> async() {
    DeferredResult<String> result = new DeferredResult<>();
    // Xử lý trên thread khác
    executorService.submit(() -> {
        result.setResult("Done!");
    });
    return result; // Giải phóng request thread ngay
}

@GetMapping("/callable")
public Callable<String> callable() {
    return () -> {
        Thread.sleep(1000);
        return "Done!"; // Chạy trên thread riêng
    };
}
```

## Bảng tóm tắt

| Return type | Dùng khi | Controller type |
|------------|---------|:---:|
| String | Trả về view name | @Controller |
| ModelAndView | View + data | @Controller |
| ResponseEntity<T> | Full control HTTP response | @RestController |
| Object (T) | Auto JSON | @RestController |
| void | Không trả về (DELETE) | Cả hai |
| Mono<T> / Flux<T> | Reactive | WebFlux |

## Điểm quan trọng nhớ phỏng vấn
1. **ResponseEntity** là linh hoạt nhất (control status, headers, body)
2. `@RestController` + Object -> tự động **JSON** qua Jackson
3. **void** + `@ResponseStatus` cho DELETE/PUT không cần body
4. **DeferredResult** cho long-polling / async processing
