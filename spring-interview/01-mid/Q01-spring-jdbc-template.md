# Q1: What is Spring JDBCTemplate class and how to use it?
> **Dịch:** Spring JDBCTemplate là gì và sử dụng nó như thế nào?

## Tra loi ngan gon
> JdbcTemplate la class giup **don gian hoa** viec lam viec voi JDBC, tu dong xu ly mo/dong connection, xu ly exception.

## Tai sao can JdbcTemplate?
JDBC truyen thong rat **verbose** (nhieu code lap lai):
- Mo connection
- Tao PreparedStatement
- Set params
- Xu ly ResultSet
- Dong connection
- Bat exception

**JdbcTemplate lam tat ca nhung viec nay cho ban!**

## Cach nho
```
JDBC thuong = Tu lai xe (phai tu mo cua, khoi dong, lai, do, tat may)
JdbcTemplate = Uber     (chi can noi dia chi, xe tu chay)
```

## Vi du so sanh

### JDBC truyen thong (DAI DONG):
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
    // xu ly exception
} finally {
    // dong rs, ps, conn (them 15 dong nua!)
}
```

### JdbcTemplate (GON GANG):
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

    // Query danh sach
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

## Cac method chinh cua JdbcTemplate

| Method | Muc dich | Vi du |
|--------|----------|-------|
| `query()` | SELECT tra ve List | `query(sql, rowMapper)` |
| `queryForObject()` | SELECT tra ve 1 object | `queryForObject(sql, rowMapper, id)` |
| `update()` | INSERT / UPDATE / DELETE | `update(sql, params...)` |
| `execute()` | DDL (CREATE TABLE...) | `execute(sql)` |
| `batchUpdate()` | Chay nhieu lenh 1 luc | `batchUpdate(sql, batchArgs)` |

## Cau hinh trong Spring Boot
```yaml
# application.yml - chi can khai bao, Spring Boot tu tao JdbcTemplate
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: root
    password: 123456
```

## Diem quan trong nho phong van
1. JdbcTemplate **khong phai ORM** (khac voi Hibernate/JPA)
2. Tu dong xu ly **resource management** (dong connection)
3. Chuyen SQLException thanh **DataAccessException** (unchecked)
4. **Thread-safe** - co the dung chung trong nhieu thread
