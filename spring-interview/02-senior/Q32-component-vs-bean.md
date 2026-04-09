# Q32: Compare @Component (v2.5) versus @Bean (v3.0)
> **Dịch:** So sánh @Component (v2.5) và @Bean (v3.0)

## Tra loi ngan gon
> `@Component` danh dau tren **class** (tu dong scan). `@Bean` danh dau tren **method** trong @Configuration class (khai bao thu cong). Dung `@Component` cho class ban viet, `@Bean` cho class thu vien ben ngoai.

## Cach nho
```
@Component = Tuyen nhan vien TRUC TIEP (den nha no danh dau)
@Bean      = Tuyen qua TRUNG GIAN (nho HR tao profile)
```

## So sanh chi tiet

| | @Component | @Bean |
|--|:---:|:---:|
| Dat o dau | **Class** level | **Method** level |
| Cach tao bean | Auto scan | Thu cong trong @Configuration |
| Dung cho | Class BAN viet | Class **thu vien** (khong sua duoc source) |
| Ten bean | Ten class (camelCase) | Ten method |
| Customize tao | Han che | **Linh hoat** (logic trong method) |

## Vi du

### @Component - cho class ban viet
```java
@Component  // Spring tu dong phat hien va tao bean
public class EmailService {
    public void send(String to, String body) {
        // logic gui email
    }
}
```

### @Bean - cho class ben ngoai (khong sua duoc source)
```java
@Configuration
public class AppConfig {

    @Bean  // Ban KHONG the them @Component vao ObjectMapper vi no la class cua Jackson
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

    // Co the co logic phuc tap khi tao bean
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

### Khi nao dung @Bean bat buoc?
```java
// 1. Class tu thu vien (khong sua duoc source)
@Bean
public ObjectMapper objectMapper() { return new ObjectMapper(); }

// 2. Can logic phuc tap khi tao
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http.csrf().disable().build();
}

// 3. Can nhieu bean cung type
@Bean("mysqlDS")
public DataSource mysqlDataSource() { /* ... */ }

@Bean("postgresDS")
public DataSource postgresDataSource() { /* ... */ }
```

## Diem quan trong nho phong van
1. `@Component` = **class level**, auto scan | `@Bean` = **method level**, thu cong
2. Dung `@Bean` khi **khong the** them annotation vao class (thu vien)
3. `@Bean` cho phep **logic phuc tap** khi tao bean
4. `@Component` ket hop component scanning, `@Bean` ket hop `@Configuration`
