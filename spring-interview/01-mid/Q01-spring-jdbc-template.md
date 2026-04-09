# Q1: What is Spring JDBCTemplate class and how to use it?
> **Dịch:** Spring JDBCTemplate là gì và sử dụng nó như thế nào?

## Trả lời ngắn gọn
> JdbcTemplate là class giúp **đơn giản hóa** việc làm việc với JDBC, tự động xử lý mở/đóng connection, xử lý exception.

## Tại sao cần JdbcTemplate?
JDBC truyền thống rất **verbose** (nhiều code lặp lại):
- Mở connection
- Tạo PreparedStatement
- Set params
- Xử lý ResultSet
- Đóng connection
- Bắt exception

**JdbcTemplate làm tất cả những việc này cho bạn!**

## Cách nhớ
```
JDBC thường = Tự lái xe (phải tự mở cửa, khởi động, lái, đỗ, tắt máy)
JdbcTemplate = Uber     (chỉ cần nói địa chỉ, xe tự chạy)
```

## Ví dụ so sánh

### JDBC truyền thống (DÀI DÒNG):
```java
Connection conn = null;
PreparedStatement ps = null;
ResultSet rs = null;
try {
    conn = dataSource.getConnection();
    ps = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
    ps.setLong(1, id);
    rs = ps.executeQuery();
    if (rs.next()) {
        User user = new User();
        user.setId(rs.getLong("id"));
        user.setName(rs.getString("name"));
        return user;
    }
} catch (SQLException e) {
    // xử lý exception
} finally {
    // đóng rs, ps, conn (thêm 15 dòng nữa!)
}
```

### JdbcTemplate (GỌN GÀNG):
```java
@Repository
public class UserDao {
    @Autowired
    private JdbcTemplate jdbcTemplate;

    // Query 1 object
    public User findById(Long id) {
        return jdbcTemplate.queryForObject(
            "SELECT * FROM users WHERE id = ?",
            new BeanPropertyRowMapper<>(User.class),
            id
        );
    }

    // Query danh sách
    public List<User> findAll() {
        return jdbcTemplate.query(
            "SELECT * FROM users",
            new BeanPropertyRowMapper<>(User.class)
        );
    }

    // Insert / Update / Delete
    public int save(User user) {
        return jdbcTemplate.update(
            "INSERT INTO users (name, email) VALUES (?, ?)",
            user.getName(), user.getEmail()
        );
    }
}
```

## Các method chính của JdbcTemplate

| Method | Mục đích | Ví dụ |
|--------|----------|-------|
| `query()` | SELECT trả về List | `query(sql, rowMapper)` |
| `queryForObject()` | SELECT trả về 1 object | `queryForObject(sql, rowMapper, id)` |
| `update()` | INSERT / UPDATE / DELETE | `update(sql, params...)` |
| `execute()` | DDL (CREATE TABLE...) | `execute(sql)` |
| `batchUpdate()` | Chạy nhiều lệnh 1 lúc | `batchUpdate(sql, batchArgs)` |

## Cấu hình trong Spring Boot
```yaml
# application.yml - chỉ cần khai báo, Spring Boot tự tạo JdbcTemplate
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: root
    password: 123456
```

## Điểm quan trọng nhớ phỏng vấn
1. JdbcTemplate **không phải ORM** (khác với Hibernate/JPA)
2. Tự động xử lý **resource management** (đóng connection)
3. Chuyển SQLException thành **DataAccessException** (unchecked)
4. **Thread-safe** - có thể dùng chung trong nhiều thread
