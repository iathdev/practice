# Q62: What is the difference between @Inject and @Autowired? Which one to use?
> **Dịch:** Sự khác nhau giữa @Inject và @Autowired là gì? Khi nào dùng cái nào?

## Tra loi ngan gon
> `@Autowired` la annotation cua **Spring**. `@Inject` la annotation chuan **JSR-330** (Java standard). Ca hai hoat dong **giong nhau** ve co ban, nhung `@Autowired` co them thuoc tinh `required`.

## Cach nho
```
@Autowired = iPhone (cua Apple/Spring - nhieu tinh nang rieng)
@Inject    = USB-C (chuan chung - dung cho moi framework)
```

## So sanh chi tiet

| | @Autowired | @Inject |
|--|:---:|:---:|
| Thu vien | Spring Framework | JSR-330 (javax.inject) |
| Thuoc tinh `required` | Co (`required=false`) | **KHONG** |
| Qualifier | `@Qualifier` (Spring) | `@Named` (JSR-330) |
| Default | byType | byType |
| Optional DI | `required=false` | Dung `Optional<T>` hoac `@Nullable` |

## Vi du so sanh

### @Autowired (Spring)
```java
@Service
public class UserService {
    @Autowired
    private UserRepository userRepo;

    @Autowired(required = false)  // Khong loi neu khong co bean
    private CacheService cache;

    @Autowired
    @Qualifier("emailNotifier")   // Chi dinh bean cu the
    private Notifier notifier;
}
```

### @Inject (JSR-330)
```java
@Service
public class UserService {
    @Inject
    private UserRepository userRepo;

    @Inject                        // Khong co `required` attribute
    private Optional<CacheService> cache; // Dung Optional thay the

    @Inject
    @Named("emailNotifier")       // @Named thay vi @Qualifier
    private Notifier notifier;
}
```

## Khi nao dung cai nao?

```
Dung @Autowired khi:
  - Project chi dung Spring (99% truong hop)
  - Can `required = false`
  - Team quen dung Spring annotations

Dung @Inject khi:
  - Can portable giua cac DI framework (Spring, Guice, CDI)
  - Theo chuan Java (JSR-330)
  - Muon giam phu thuoc vao Spring API

Thuc te: HAU HET project dung @Autowired
         Hoac tot hon: CONSTRUCTOR INJECTION (khong can annotation nao ca)
```

### Best practice: Constructor Injection (khong can ca 2!)
```java
@Service
public class UserService {
    private final UserRepository userRepo;

    // Khong can @Autowired hay @Inject
    // Spring tu dong inject khi chi co 1 constructor
    public UserService(UserRepository userRepo) {
        this.userRepo = userRepo;
    }
}
```

## Diem quan trong nho phong van
1. `@Autowired` = Spring, `@Inject` = Java standard (JSR-330)
2. Chuc nang **tuong tu**, `@Autowired` co them `required`
3. **Constructor injection** la best practice - **khong can** annotation nao
4. Thuc te: **dung @Autowired** (vi da dung Spring roi)
