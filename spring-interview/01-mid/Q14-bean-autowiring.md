# Q14: What is bean auto wiring?
> **Dịch:** Bean Auto Wiring (tự động nối dây) là gì?

## Tra loi ngan gon
> **Autowiring** la co che Spring **tu dong inject dependency** vao bean ma khong can khai bao thu cong. Spring se tu tim bean phu hop dua tren **type**, **name**, hoac **constructor**.

## Cach nho
```
Khong autowiring = Tu cam sac dien thoai (phai tim o cam, cam day)
Autowiring       = Sac khong day (dat len la tu sac)
```

## So sanh: Thu cong vs Autowiring

### Thu cong (XML):
```xml
<bean id="userService" class="com.example.UserService">
    <property name="userRepo" ref="userRepository"/> <!-- Khai bao thu cong -->
</bean>
```

### Autowiring (annotation):
```java
@Service
public class UserService {
    @Autowired  // Spring TU DONG tim va inject UserRepository
    private UserRepository userRepo;
}
```

## 3 cach Autowiring bang Annotation

### 1. Field Injection (nhanh nhung KHONG khuyen dung)
```java
@Service
public class UserService {
    @Autowired
    private UserRepository userRepo; // Inject truc tiep vao field
}
// Nhuoc: Khong test duoc, khong thay dependency ro rang
```

### 2. Constructor Injection (KHUYEN DUNG)
```java
@Service
public class UserService {
    private final UserRepository userRepo;

    // @Autowired - co the bo neu chi co 1 constructor
    public UserService(UserRepository userRepo) {
        this.userRepo = userRepo;
    }
}
// Uu: Final field, de test, dependency ro rang
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
// Dung khi dependency la OPTIONAL
```

## Xu ly khi co nhieu bean cung type

```java
// 2 bean cung implement UserRepository
@Repository
public class MySQLUserRepo implements UserRepository { }

@Repository
public class MongoUserRepo implements UserRepository { }

// Cach 1: @Qualifier - chi dinh bean cu the
@Autowired
@Qualifier("mySQLUserRepo")
private UserRepository userRepo;

// Cach 2: @Primary - danh dau bean uu tien
@Primary
@Repository
public class MySQLUserRepo implements UserRepository { }

// Cach 3: Dat ten bien trung voi bean name
@Autowired
private UserRepository mySQLUserRepo; // Tim theo ten
```

## Diem quan trong nho phong van
1. **Constructor injection** la best practice (final, testable)
2. Neu chi co **1 constructor**, `@Autowired` co the **bo qua**
3. Nhieu bean cung type -> dung `@Qualifier` hoac `@Primary`
4. `@Autowired(required = false)` cho dependency **khong bat buoc**
