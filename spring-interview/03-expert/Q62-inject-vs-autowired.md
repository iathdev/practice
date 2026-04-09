# Q62: What is the difference between @Inject and @Autowired? Which one to use?
> **Dịch:** Sự khác nhau giữa @Inject và @Autowired là gì? Khi nào dùng cái nào?

## Trả lời ngắn gọn
> `@Autowired` là annotation của **Spring**. `@Inject` là annotation chuẩn **JSR-330** (Java standard). Cả hai hoạt động **giống nhau** về cơ bản, nhưng `@Autowired` có thêm thuộc tính `required`.

## Cách nhớ
```
@Autowired = iPhone (của Apple/Spring - nhiều tính năng riêng)
@Inject    = USB-C (chuẩn chung - dùng cho mọi framework)
```

## So sánh chi tiết

| | @Autowired | @Inject |
|--|:---:|:---:|
| Thư viện | Spring Framework | JSR-330 (javax.inject) |
| Thuộc tính `required` | Có (`required=false`) | **KHÔNG** |
| Qualifier | `@Qualifier` (Spring) | `@Named` (JSR-330) |
| Default | byType | byType |
| Optional DI | `required=false` | Dùng `Optional<T>` hoặc `@Nullable` |

## Ví dụ so sánh

### @Autowired (Spring)
```java
@Service
public class UserService {
    @Autowired
    private UserRepository userRepo;

    @Autowired(required = false)  // Không lỗi nếu không có bean
    private CacheService cache;

    @Autowired
    @Qualifier("emailNotifier")   // Chỉ định bean cụ thể
    private Notifier notifier;
}
```

### @Inject (JSR-330)
```java
@Service
public class UserService {
    @Inject
    private UserRepository userRepo;

    @Inject                        // Không có `required` attribute
    private Optional<CacheService> cache; // Dùng Optional thay thế

    @Inject
    @Named("emailNotifier")       // @Named thay vì @Qualifier
    private Notifier notifier;
}
```

## Khi nào dùng cái nào?

```
Dùng @Autowired khi:
  - Project chỉ dùng Spring (99% trường hợp)
  - Cần `required = false`
  - Team quen dùng Spring annotations

Dùng @Inject khi:
  - Cần portable giữa các DI framework (Spring, Guice, CDI)
  - Theo chuẩn Java (JSR-330)
  - Muốn giảm phụ thuộc vào Spring API

Thực tế: HẦU HẾT project dùng @Autowired
         Hoặc tốt hơn: CONSTRUCTOR INJECTION (không cần annotation nào cả)
```

### Best practice: Constructor Injection (không cần cả 2!)
```java
@Service
public class UserService {
    private final UserRepository userRepo;

    // Không cần @Autowired hay @Inject
    // Spring tự động inject khi chỉ có 1 constructor
    public UserService(UserRepository userRepo) {
        this.userRepo = userRepo;
    }
}
```

## Điểm quan trọng nhớ phỏng vấn
1. `@Autowired` = Spring, `@Inject` = Java standard (JSR-330)
2. Chức năng **tương tự**, `@Autowired` có thêm `required`
3. **Constructor injection** là best practice - **không cần** annotation nào
4. Thực tế: **dùng @Autowired** (vì đã dùng Spring rồi)
