# Q19: Can we send an Object as the response of Controller handler method?
> **Dịch:** Có thể trả về Object làm response từ method của Controller không?

## Trả lời ngắn gọn
> **Có!** Dùng `@ResponseBody` hoặc `@RestController`, Spring sẽ tự động chuyển đổi (serialize) Object thành **JSON/XML** qua **HttpMessageConverter** (mặc định là Jackson).

## Cách nhớ
```
@Controller    = Trả về tên View (HTML)
@RestController = Trả về Object -> tự động chuyển JSON
```

## Ví dụ

### Trả về Object (JSON)
```java
@RestController // = @Controller + @ResponseBody
@RequestMapping("/api")
public class UserController {

    // Trả về 1 object -> JSON
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return new User(1L, "Thai", "thai@email.com");
    }
    // Response: {"id": 1, "name": "Thai", "email": "thai@email.com"}

    // Trả về List -> JSON array
    @GetMapping("/users")
    public List<User> getUsers() {
        return List.of(
            new User(1L, "Thai", "thai@email.com"),
            new User(2L, "Dong", "dong@email.com")
        );
    }
    // Response: [{"id": 1, ...}, {"id": 2, ...}]

    // Trả về ResponseEntity (kèm HTTP status, headers)
    @GetMapping("/users/{id}/detail")
    public ResponseEntity<User> getUserDetail(@PathVariable Long id) {
        User user = userService.findById(id);
        if (user == null) {
            return ResponseEntity.notFound().build(); // 404
        }
        return ResponseEntity.ok(user); // 200 + JSON
    }
}
```

### Quá trình chuyển đổi
```
Object (Java) --> HttpMessageConverter --> JSON (Response)
                  (Jackson mặc định)

User{id=1, name="Thai"} --> {"id": 1, "name": "Thai"}
```

### Customize JSON output
```java
public class User {
    private Long id;

    @JsonProperty("full_name")  // Đổi tên field trong JSON
    private String name;

    @JsonIgnore                 // Ẩn field, không trả về
    private String password;

    @JsonFormat(pattern = "dd/MM/yyyy")  // Format ngày
    private LocalDate birthday;
}
// Output: {"id": 1, "full_name": "Thai", "birthday": "09/04/2026"}
```

## Điểm quan trọng nhớ phỏng vấn
1. `@RestController` tự động serialize Object -> JSON
2. **Jackson** là thư viện mặc định để chuyển đổi JSON
3. Dùng **ResponseEntity** khi cần control HTTP status và headers
4. Customize với `@JsonProperty`, `@JsonIgnore`, `@JsonFormat`
