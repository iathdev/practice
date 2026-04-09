# Q57: Explain the difference between annotation-config vs component-scan
> **Dịch:** Giải thích sự khác nhau giữa annotation-config và component-scan

## Tra loi ngan gon
> `<context:annotation-config>` chi **kich hoat** cac annotation (@Autowired, @PostConstruct...) cho bean DA DANG KY. `<context:component-scan>` lam tat ca nhung gi annotation-config lam CONG THEM **tu dong scan** va **dang ky** bean moi tu package.

## Cach nho
```
annotation-config = Bat dien trong nha (nha DA xay xong)
component-scan    = Xay nha + Bat dien (xay xong tu bat)
```

## So sanh

| | annotation-config | component-scan |
|--|:---:|:---:|
| Kich hoat annotations | Co | Co |
| Tu dong tim bean moi | **KHONG** | **CO** |
| Scan package | Khong | Co |
| Bao gom annotation-config | Khong | **Co** (tu dong) |

## Vi du

### annotation-config: chi kich hoat annotation
```xml
<!-- Bean phai DANG KY THU CONG -->
<bean id="userRepo" class="com.example.UserRepository"/>
<bean id="userService" class="com.example.UserService"/>

<!-- Chi kich hoat @Autowired, @PostConstruct... cho cac bean tren -->
<context:annotation-config/>
```

```java
// @Autowired HOAT DONG vi da bat annotation-config
// Nhung bean van phai dang ky trong XML
@Service  // KHONG TU DONG tao bean! (chi co annotation-config)
public class UserService {
    @Autowired  // Hoat dong!
    private UserRepository userRepo;
}
```

### component-scan: scan + kich hoat (THUONG DUNG)
```xml
<!-- Tu dong scan package, tim @Component/@Service/@Repository/@Controller -->
<!-- DONG THOI kich hoat annotations (bao gom annotation-config) -->
<context:component-scan base-package="com.example"/>

<!-- KHONG can dang ky bean thu cong nua! -->
```

```java
@Service  // TU DONG duoc scan va dang ky thanh bean!
public class UserService {
    @Autowired  // Hoat dong!
    private UserRepository userRepo;
}
```

### Trong Spring Boot (hien dai)
```java
@SpringBootApplication  // DA BAO GOM component-scan cho package hien tai
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
// Khong can XML nao ca!
```

## Diem quan trong nho phong van
1. `annotation-config` = **kich hoat** annotations cho bean da co
2. `component-scan` = **tim + dang ky** bean MOI + kich hoat annotations
3. `component-scan` **bao gom** annotation-config (khong can ca 2)
4. Spring Boot: `@SpringBootApplication` = `@ComponentScan` + `@EnableAutoConfiguration` + `@Configuration`
