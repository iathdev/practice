# Q4: What is Spring IoC container?
> **Dịch:** Spring IoC Container là gì?

## Trả lời ngắn gọn
> Spring IoC Container là **trái tim** của Spring Framework, chịu trách nhiệm **tạo, cấu hình, quản lý vòng đời** của các object (bean). IoC = Inversion of Control (đảo ngược quyền điều khiển).

## Cách nhớ
```
KHÔNG có IoC = Bạn tự nấu ăn (đi chợ, rửa rau, nấu, dọn dẹp)
CÓ IoC       = Đặt cơm Grab (chỉ nói cần gì, người khác làm hết)

Container = "Quản lý nhà hàng" - biết tạo món gì, phục vụ cho ai
```

## So sánh: Không IoC vs Có IoC

### Không có IoC (tự tạo dependency):
```java
public class OrderService {
    // TỰ TẠO - phụ thuộc chặt vào class cụ thể
    private UserRepository userRepo = new MySQLUserRepository();
    private EmailService email = new GmailEmailService();
    
    // Muốn đổi sang PostgreSQL? -> Phải SỬA CODE!
}
```

### Có IoC Container:
```java
@Service
public class OrderService {
    // CONTAINER TỰ INJECT - chỉ phụ thuộc vào interface
    private final UserRepository userRepo;
    private final EmailService email;
    
    @Autowired
    public OrderService(UserRepository userRepo, EmailService email) {
        this.userRepo = userRepo;  // Container quyết định inject cái nào
        this.email = email;
    }
    // Đổi từ MySQL sang PostgreSQL? -> Chỉ cần đổi config, KHÔNG sửa code!
}
```

## 2 loại IoC Container

### 1. BeanFactory (cơ bản)
```java
// Lazy loading - chỉ tạo bean khi được gọi
BeanFactory factory = new XmlBeanFactory(
    new ClassPathResource("beans.xml")
);
MyBean bean = factory.getBean("myBean", MyBean.class);
```

### 2. ApplicationContext (nâng cao - THƯỜNG DÙNG)
```java
// Eager loading - tạo tất cả singleton bean khi khởi động
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
MyBean bean = ctx.getBean(MyBean.class);
```

## ApplicationContext làm được nhiều hơn BeanFactory

| Tính năng | BeanFactory | ApplicationContext |
|-----------|:-----------:|:------------------:|
| Tạo & quản lý bean | Có | Có |
| Lazy loading | Mặc định | Tùy chọn |
| Event publishing | Không | Có |
| Internationalization (i18n) | Không | Có |
| AOP tích hợp | Cơ bản | Đầy đủ |
| Auto BeanPostProcessor | Không | Có |

## Container hoạt động như thế nào?

```
[Java Classes] + [Configuration] --> [IoC Container] --> [Ready Beans]

Configuration có thể là:
  - XML:        <bean id="..." class="...">
  - Annotation: @Component, @Service, @Repository
  - Java:       @Configuration + @Bean
```

## Ví dụ đầy đủ:
```java
// 1. Định nghĩa bean bằng annotation
@Service
public class UserService {
    @Autowired
    private UserRepository userRepo;
    
    public User findUser(Long id) {
        return userRepo.findById(id).orElse(null);
    }
}

// 2. Container tự động:
//    - Quét tìm @Service -> tạo UserService
//    - Quét tìm @Repository -> tạo UserRepository
//    - Inject UserRepository vào UserService
//    - Quản lý vòng đời của tất cả bean
```

## Điểm quan trọng nhớ phỏng vấn
1. IoC Container = **Đảo ngược quyền điều khiển** (container tạo object, không phải developer)
2. **2 loại**: BeanFactory (cơ bản) và ApplicationContext (thường dùng)
3. Container đọc **metadata** (XML/annotation/Java config) để biết cách tạo bean
4. Lợi ích chính: **Loose coupling**, dễ test, dễ thay đổi implementation
