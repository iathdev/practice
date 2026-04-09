# Q32: Compare @Component (v2.5) versus @Bean (v3.0)
> **Dịch:** So sánh @Component (v2.5) và @Bean (v3.0)

## Trả lời ngắn gọn
> `@Component` đánh dấu trên **class** (tự động scan). `@Bean` đánh dấu trên **method** trong @Configuration class (khai báo thủ công). Dùng `@Component` cho class bạn viết, `@Bean` cho class thư viện bên ngoài.

## Cách nhớ
```
@Component = Tuyển nhân viên TRỰC TIẾP (đến nhà nó đánh dấu)
@Bean      = Tuyển qua TRUNG GIAN (nhờ HR tạo profile)
```

## So sánh chi tiết

| | @Component | @Bean |
|--|:---:|:---:|
| Đặt ở đâu | **Class** level | **Method** level |
| Cách tạo bean | Auto scan | Thủ công trong @Configuration |
| Dùng cho | Class BẠN viết | Class **thư viện** (không sửa được source) |
| Tên bean | Tên class (camelCase) | Tên method |
| Customize tạo | Hạn chế | **Linh hoạt** (logic trong method) |

## Ví dụ

### @Component - cho class bạn viết
```java
@Component  // Spring tự động phát hiện và tạo bean
public class EmailService {
    public void send(String to, String body) {
        // logic gửi email
    }
}
```

### @Bean - cho class bên ngoài (không sửa được source)
```java
@Configuration
public class AppConfig {

    @Bean  // Bạn KHÔNG thể thêm @Component vào ObjectMapper vì nó là class của Jackson
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
            .registerModule(new JavaTimeModule())
            .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
    }

    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplateBuilder()
            .setConnectTimeout(Duration.ofSeconds(5))
            .build();
    }

    // Có thể có logic phức tạp khi tạo bean
    @Bean
    @Profile("prod")
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://prod-server/mydb");
        config.setMaximumPoolSize(20);
        return new HikariDataSource(config);
    }
}
```

### Khi nào dùng @Bean bắt buộc?
```java
// 1. Class từ thư viện (không sửa được source)
@Bean
public ObjectMapper objectMapper() { return new ObjectMapper(); }

// 2. Cần logic phức tạp khi tạo
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http.csrf().disable().build();
}

// 3. Cần nhiều bean cùng type
@Bean("mysqlDS")
public DataSource mysqlDataSource() { /* ... */ }

@Bean("postgresDS")
public DataSource postgresDataSource() { /* ... */ }
```

## Điểm quan trọng nhớ phỏng vấn
1. `@Component` = **class level**, auto scan | `@Bean` = **method level**, thủ công
2. Dùng `@Bean` khi **không thể** thêm annotation vào class (thư viện)
3. `@Bean` cho phép **logic phức tạp** khi tạo bean
4. `@Component` kết hợp component scanning, `@Bean` kết hợp `@Configuration`
