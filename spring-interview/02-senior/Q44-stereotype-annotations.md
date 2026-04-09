# Q44: What's the difference between @Component, @Controller, @Repository & @Service?
> **Dịch:** Sự khác nhau giữa @Component, @Controller, @Repository và @Service là gì?

## Tra loi ngan gon
> Tat ca deu la **stereotype annotations** de danh dau class la Spring bean. `@Controller`, `@Service`, `@Repository` la **specialization** cua `@Component`, them chuc nang rieng cho tung layer.

## Cach nho
```
@Component  = Nhan vien (chung chung)
@Controller = Le tan      (tiep khach - web layer)
@Service    = Ky su       (xu ly cong viec - business layer)
@Repository = Thu kho     (quan ly kho hang - data layer)
```

## So sanh

| Annotation | Layer | Chuc nang dac biet |
|-----------|-------|-------------------|
| `@Component` | Chung | Khong co gi dac biet |
| `@Controller` | Web/Presentation | Xu ly HTTP request, DispatcherServlet nhan dien |
| `@Service` | Business/Service | Khong co gi dac biet (semantic only) |
| `@Repository` | Data/Persistence | **Tu dong dich** exception sang DataAccessException |

## Dac biet cua @Repository

```java
@Repository
public class UserDao {
    @Autowired
    private JdbcTemplate jdbcTemplate;

    public User findById(Long id) {
        // Neu JDBC nem SQLException
        // Spring tu dong chuyen thanh DataAccessException (unchecked)
        // -> Khong can try-catch SQLException khap noi
        return jdbcTemplate.queryForObject("SELECT * FROM users WHERE id=?",
            new BeanPropertyRowMapper<>(User.class), id);
    }
}
```

## Kien truc tang (layered architecture)

```
@Controller / @RestController    <-- Presentation Layer
        |
@Service                         <-- Business Layer
        |
@Repository                      <-- Data Access Layer
        |
    Database
```

## Vi du day du
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

## Diem quan trong nho phong van
1. `@Service` va `@Component` **ve ky thuat** giong nhau (chi khac ngu nghia)
2. `@Repository` them **exception translation** (quan trong!)
3. `@Controller` giup **DispatcherServlet** nhan dien handler
4. Dung dung annotation theo **layer** de code ro rang va nhat quan
