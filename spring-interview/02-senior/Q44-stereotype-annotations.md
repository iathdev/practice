# Q44: What's the difference between @Component, @Controller, @Repository & @Service?
> **Dịch:** Sự khác nhau giữa @Component, @Controller, @Repository và @Service là gì?

## Trả lời ngắn gọn
> Tất cả đều là **stereotype annotations** để đánh dấu class là Spring bean. `@Controller`, `@Service`, `@Repository` là **specialization** (biến thể chuyên biệt) của `@Component`, thêm chức năng riêng cho từng layer.

## Cách nhớ
```
@Component  = Nhân viên (chung chung)
@Controller = Lễ tân      (tiếp khách - web layer)
@Service    = Kỹ sư       (xử lý công việc - business layer)
@Repository = Thủ kho     (quản lý kho hàng - data layer)
```

## So sánh

| Annotation | Layer | Chức năng đặc biệt |
|-----------|-------|-------------------|
| `@Component` | Chung | Không có gì đặc biệt |
| `@Controller` | Web/Presentation | Xử lý HTTP request, DispatcherServlet nhận diện |
| `@Service` | Business/Service | Không có gì đặc biệt (chỉ mang ý nghĩa ngữ nghĩa) |
| `@Repository` | Data/Persistence | **Tự động dịch** exception sang DataAccessException |

## Đặc biệt của @Repository

```java
@Repository
public class UserDao {
    @Autowired
    private JdbcTemplate jdbcTemplate;

    public User findById(Long id) {
        // Nếu JDBC ném SQLException
        // Spring tự động chuyển thành DataAccessException (unchecked)
        // -> Không cần try-catch SQLException khắp nơi
        return jdbcTemplate.queryForObject("SELECT * FROM users WHERE id=?",
            new BeanPropertyRowMapper<>(User.class), id);
    }
}
```

## Kiến trúc tầng (layered architecture)

```
@Controller / @RestController    <-- Presentation Layer
        |
@Service                         <-- Business Layer
        |
@Repository                      <-- Data Access Layer
        |
    Database
```

## Ví dụ đầy đủ
```java
@RestController                           // Web layer
@RequestMapping("/api/users")
public class UserController {
    private final UserService service;
    public UserController(UserService service) { this.service = service; }

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) { return service.findById(id); }
}

@Service                                  // Business layer
public class UserService {
    private final UserRepository repo;
    public UserService(UserRepository repo) { this.repo = repo; }

    public User findById(Long id) {
        return repo.findById(id).orElseThrow(() -> new NotFoundException("User", id));
    }
}

@Repository                               // Data layer
public interface UserRepository extends JpaRepository<User, Long> { }
```

## Điểm quan trọng nhớ phỏng vấn
1. `@Service` và `@Component` **về kỹ thuật** giống nhau (chỉ khác ngữ nghĩa)
2. `@Repository` thêm **exception translation** (quan trọng!)
3. `@Controller` giúp **DispatcherServlet** nhận diện handler
4. Dùng đúng annotation theo **layer** để code rõ ràng và nhất quán
