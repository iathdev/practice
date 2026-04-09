# Q63: What's the difference between @Component, @Repository & @Service?
> **Dịch:** Sự khác nhau giữa @Component, @Repository và @Service là gì?

## Tra loi ngan gon
> Ca 3 deu la **Spring bean** annotations. `@Component` la annotation chung. `@Service` danh dau **business layer** (khong co chuc nang dac biet). `@Repository` danh dau **data layer** va them **exception translation** (chuyen SQL exception thanh Spring DataAccessException).

(Cau nay trung voi Q44 - day la ban tom tat ngan gon)

## Tom tat 1 trang

```java
// @Component - Bean chung, khong thuoc layer cu the
@Component
public class EmailHelper {
    public boolean isValid(String email) {
        return email.contains("@");
    }
}

// @Service - Business layer (KHONG co chuc nang dac biet ve ky thuat)
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

// @Repository - Data layer (CO exception translation)
@Repository
public class UserDaoImpl implements UserDao {
    @Autowired
    private JdbcTemplate jdbc;

    public User findById(Long id) {
        // Neu JDBC nem SQLException
        // -> Spring TU DONG chuyen thanh DataAccessException
        // -> Developer khong can catch SQLException
        return jdbc.queryForObject("SELECT * FROM users WHERE id=?",
            new BeanPropertyRowMapper<>(User.class), id);
    }
}
```

## Bang so sanh cuoi cung

| | @Component | @Service | @Repository |
|--|:---:|:---:|:---:|
| Layer | Chung | Business | Data |
| Auto scan | Co | Co | Co |
| Exception translation | Khong | Khong | **CO** |
| Chuc nang ky thuat rieng | Khong | Khong | Chuyen exception |
| Ngu nghia | Generic | Business logic | Data access |

## Diem quan trong nho phong van
1. `@Service` va `@Component` **giong nhau** ve ky thuat (chi khac ngu nghia)
2. `@Repository` **them exception translation** -> diem khac biet CHINH
3. Dung dung annotation theo layer -> code **ro rang, chuyen nghiep**
4. Spring Data JPA: `interface extends JpaRepository` **TU DONG** la @Repository
