# Q57: Explain the difference between annotation-config vs component-scan
> **Dịch:** Giải thích sự khác nhau giữa annotation-config và component-scan

## Trả lời ngắn gọn
> `<context:annotation-config>` chỉ **kích hoạt** các annotation (@Autowired, @PostConstruct...) cho bean ĐÃ ĐĂNG KÝ. `<context:component-scan>` làm tất cả những gì annotation-config làm CỘNG THÊM **tự động scan** và **đăng ký** bean mới từ package.

## Cách nhớ
```
annotation-config = Bật điện trong nhà (nhà ĐÃ xây xong)
component-scan    = Xây nhà + Bật điện (xây xong tự bật)
```

## So sánh

| | annotation-config | component-scan |
|--|:---:|:---:|
| Kích hoạt annotations | Có | Có |
| Tự động tìm bean mới | **KHÔNG** | **CÓ** |
| Scan package | Không | Có |
| Bao gồm annotation-config | Không | **Có** (tự động) |

## Ví dụ

### annotation-config: chỉ kích hoạt annotation
```xml
<!-- Bean phải ĐĂNG KÝ THỦ CÔNG -->
<bean id="userRepo" class="com.example.UserRepository"/>
<bean id="userService" class="com.example.UserService"/>

<!-- Chỉ kích hoạt @Autowired, @PostConstruct... cho các bean trên -->
<context:annotation-config/>
```

```java
// @Autowired HOẠT ĐỘNG vì đã bật annotation-config
// Nhưng bean vẫn phải đăng ký trong XML
@Service  // KHÔNG TỰ ĐỘNG tạo bean! (chỉ có annotation-config)
public class UserService {
    @Autowired  // Hoạt động!
    private UserRepository userRepo;
}
```

### component-scan: scan + kích hoạt (THƯỜNG DÙNG)
```xml
<!-- Tự động scan package, tìm @Component/@Service/@Repository/@Controller -->
<!-- ĐỒNG THỜI kích hoạt annotations (bao gồm annotation-config) -->
<context:component-scan base-package="com.example"/>

<!-- KHÔNG cần đăng ký bean thủ công nữa! -->
```

```java
@Service  // TỰ ĐỘNG được scan và đăng ký thành bean!
public class UserService {
    @Autowired  // Hoạt động!
    private UserRepository userRepo;
}
```

### Trong Spring Boot (hiện đại)
```java
@SpringBootApplication  // ĐÃ BAO GỒM component-scan cho package hiện tại
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
// Không cần XML nào cả!
```

## Điểm quan trọng nhớ phỏng vấn
1. `annotation-config` = **kích hoạt** annotations cho bean đã có
2. `component-scan` = **tìm + đăng ký** bean MỚI + kích hoạt annotations
3. `component-scan` **bao gồm** annotation-config (không cần cả 2)
4. Spring Boot: `@SpringBootApplication` = `@ComponentScan` + `@EnableAutoConfiguration` + `@Configuration`
