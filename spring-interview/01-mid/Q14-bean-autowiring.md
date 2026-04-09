# Q14: What is bean auto wiring?
> **Dịch:** Bean Auto Wiring (tự động nối dây) là gì?

## Trả lời ngắn gọn
> **Autowiring** là cơ chế Spring **tự động inject dependency** vào bean mà không cần khai báo thủ công. Spring sẽ tự tìm bean phù hợp dựa trên **type**, **name**, hoặc **constructor**.

## Cách nhớ
```
Không autowiring = Tự cắm sạc điện thoại (phải tìm ổ cắm, cắm dây)
Autowiring       = Sạc không dây (đặt lên là tự sạc)
```

## So sánh: Thủ công vs Autowiring

### Thủ công (XML):
```xml
<bean id="userService" class="com.example.UserService">
    <property name="userRepo" ref="userRepository"/> <!-- Khai báo thủ công -->
</bean>
```

### Autowiring (annotation):
```java
@Service
public class UserService {
    @Autowired  // Spring TỰ ĐỘNG tìm và inject UserRepository
    private UserRepository userRepo;
}
```

## 3 cách Autowiring bằng Annotation

### 1. Field Injection (nhanh nhưng KHÔNG khuyên dùng)
```java
@Service
public class UserService {
    @Autowired
    private UserRepository userRepo; // Inject trực tiếp vào field
}
// Nhược: Không test được, không thấy dependency rõ ràng
```

### 2. Constructor Injection (KHUYÊN DÙNG)
```java
@Service
public class UserService {
    private final UserRepository userRepo;

    // @Autowired - có thể bỏ nếu chỉ có 1 constructor
    public UserService(UserRepository userRepo) {
        this.userRepo = userRepo;
    }
}
// Ưu: Final field, dễ test, dependency rõ ràng
```

### 3. Setter Injection
```java
@Service
public class UserService {
    private UserRepository userRepo;

    @Autowired
    public void setUserRepo(UserRepository userRepo) {
        this.userRepo = userRepo;
    }
}
// Dùng khi dependency là OPTIONAL
```

## Xử lý khi có nhiều bean cùng type

```java
// 2 bean cùng implement UserRepository
@Repository
public class MySQLUserRepo implements UserRepository { }

@Repository
public class MongoUserRepo implements UserRepository { }

// Cách 1: @Qualifier - chỉ định bean cụ thể
@Autowired
@Qualifier("mySQLUserRepo")
private UserRepository userRepo;

// Cách 2: @Primary - đánh dấu bean ưu tiên
@Primary
@Repository
public class MySQLUserRepo implements UserRepository { }

// Cách 3: Đặt tên biến trùng với bean name
@Autowired
private UserRepository mySQLUserRepo; // Tìm theo tên
```

## Điểm quan trọng nhớ phỏng vấn
1. **Constructor injection** là best practice (final, testable)
2. Nếu chỉ có **1 constructor**, `@Autowired` có thể **bỏ qua**
3. Nhiều bean cùng type -> dùng `@Qualifier` hoặc `@Primary`
4. `@Autowired(required = false)` cho dependency **không bắt buộc**
