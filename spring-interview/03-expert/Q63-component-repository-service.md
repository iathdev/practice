# Q63: What's the difference between @Component, @Repository & @Service?
> **Dịch:** Sự khác nhau giữa @Component, @Repository và @Service là gì?

## Trả lời ngắn gọn
> Cả 3 đều là **Spring bean** annotations. `@Component` là annotation chung. `@Service` đánh dấu **business layer** (không có chức năng đặc biệt). `@Repository` đánh dấu **data layer** và thêm **exception translation** (chuyển SQL exception thành Spring DataAccessException).

(Câu này trùng với Q44 - đây là bản tóm tắt ngắn gọn)

## Tóm tắt 1 trang

```java
// @Component - Bean chung, không thuộc layer cụ thể
@Component
public class EmailHelper {
    public boolean isValid(String email) {
        return email.contains("@");
    }
}

// @Service - Business layer (KHÔNG có chức năng đặc biệt về kỹ thuật)
@Service
public class UserService {
    private final UserRepository repo;
    public UserService(UserRepository repo) { this.repo = repo; }

    public User register(UserDto dto) {
        // Business logic: validate, transform, save
        if (repo.existsByEmail(dto.getEmail())) {
            throw new DuplicateEmailException();
        }
        return repo.save(toEntity(dto));
    }
}

// @Repository - Data layer (CÓ exception translation)
@Repository
public class UserDaoImpl implements UserDao {
    @Autowired
    private JdbcTemplate jdbc;

    public User findById(Long id) {
        // Nếu JDBC ném SQLException
        // -> Spring TỰ ĐỘNG chuyển thành DataAccessException
        // -> Developer không cần catch SQLException
        return jdbc.queryForObject("SELECT * FROM users WHERE id=?",
            new BeanPropertyRowMapper<>(User.class), id);
    }
}
```

## Bảng so sánh cuối cùng

| | @Component | @Service | @Repository |
|--|:---:|:---:|:---:|
| Layer | Chung | Business | Data |
| Auto scan | Có | Có | Có |
| Exception translation | Không | Không | **CÓ** |
| Chức năng kỹ thuật riêng | Không | Không | Chuyển exception |
| Ngữ nghĩa | Generic | Business logic | Data access |

## Điểm quan trọng nhớ phỏng vấn
1. `@Service` và `@Component` **giống nhau** về kỹ thuật (chỉ khác ngữ nghĩa)
2. `@Repository` **thêm exception translation** -> điểm khác biệt CHÍNH
3. Dùng đúng annotation theo layer -> code **rõ ràng, chuyên nghiệp**
4. Spring Data JPA: `interface extends JpaRepository` **TỰ ĐỘNG** là @Repository
