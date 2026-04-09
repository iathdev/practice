# Q27: What is Controller in Spring MVC framework?
> **Dịch:** Controller trong Spring MVC Framework là gì?

## Trả lời ngắn gọn
> **Controller** là component xử lý HTTP request từ client, gọi business logic (Service), và trả về response (View hoặc JSON). Trong Spring MVC, controller được đánh dấu bằng `@Controller` hoặc `@RestController`.

## Cách nhớ
```
Controller = Bồi bàn nhà hàng
1. Nhận order (HTTP Request)
2. Chuyển cho bếp (Service layer)
3. Mang món ra (Response)
```

## Ví dụ đầy đủ

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    // GET /api/users
    @GetMapping
    public List<User> list() {
        return userService.findAll();
    }

    // GET /api/users/1
    @GetMapping("/{id}")
    public ResponseEntity<User> getById(@PathVariable Long id) {
        return userService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    // POST /api/users
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public User create(@Valid @RequestBody UserDto dto) {
        return userService.create(dto);
    }

    // PUT /api/users/1
    @PutMapping("/{id}")
    public User update(@PathVariable Long id, @Valid @RequestBody UserDto dto) {
        return userService.update(id, dto);
    }

    // DELETE /api/users/1
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

## Controller CHỈ nên làm gì?
```
NÊN làm:                           KHÔNG NÊN làm:
- Nhận request                    - Viết business logic
- Validate input (@Valid)         - Truy cập database trực tiếp
- Gọi Service                     - Xử lý phức tạp
- Trả về response                 - Giữ state
```

## Điểm quan trọng nhớ phỏng vấn
1. Controller là **entry point** của request
2. Nên **mỏng** (thin controller) - logic ở **Service layer**
3. `@Controller` -> View, `@RestController` -> JSON
4. Dùng `ResponseEntity` khi cần control **status code và headers**
