# Q4: What is Spring IoC container?
> **Dịch:** Spring IoC Container là gì?

## Tra loi ngan gon
> Spring IoC Container la **trai tim** cua Spring Framework, chiu trach nhiem **tao, cau hinh, quan ly vong doi** cua cac object (bean). IoC = Inversion of Control (dao nguoc quyen dieu khien).

## Cach nho
```
KHONG co IoC = Ban tu nau an (di cho, rua rau, nau, don dep)
CO IoC       = Dat com Grab (chi noi can gi, nguoi khac lam het)

Container = "Quan ly nha hang" - biet tao mon gi, phuc vu cho ai
```

## So sanh: Khong IoC vs Co IoC

### Khong co IoC (tu tao dependency):
```java
public class OrderService {
    // TU TAO - phu thuoc chat vao class cu the
    private UserRepository userRepo = new MySQLUserRepository();
    private EmailService email = new GmailEmailService();
    
    // Muon doi sang PostgreSQL? -> Phai SUA CODE!
}
```

### Co IoC Container:
```java
@Service
public class OrderService {
    // CONTAINER TU INJECT - chi phu thuoc vao interface
    private final UserRepository userRepo;
    private final EmailService email;
    
    @Autowired
    public OrderService(UserRepository userRepo, EmailService email) {
        this.userRepo = userRepo;  // Container quyet dinh inject cai nao
        this.email = email;
    }
    // Doi tu MySQL sang PostgreSQL? -> Chi can doi config, KHONG sua code!
}
```

## 2 loai IoC Container

### 1. BeanFactory (co ban)
```java
// Lazy loading - chi tao bean khi duoc goi
BeanFactory factory = new XmlBeanFactory(
    new ClassPathResource("beans.xml")
);
MyBean bean = factory.getBean("myBean", MyBean.class);
```

### 2. ApplicationContext (nang cao - THUONG DUNG)
```java
// Eager loading - tao tat ca singleton bean khi khoi dong
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
MyBean bean = ctx.getBean(MyBean.class);
```

## ApplicationContext lam duoc nhieu hon BeanFactory

| Tinh nang | BeanFactory | ApplicationContext |
|-----------|:-----------:|:------------------:|
| Tao & quan ly bean | Co | Co |
| Lazy loading | Mac dinh | Tuy chon |
| Event publishing | Khong | Co |
| Internationalization (i18n) | Khong | Co |
| AOP tich hop | Co ban | Day du |
| Auto BeanPostProcessor | Khong | Co |

## Container hoat dong nhu the nao?

```
[Java Classes] + [Configuration] --> [IoC Container] --> [Ready Beans]

Configuration co the la:
  - XML:        <bean id="..." class="...">
  - Annotation: @Component, @Service, @Repository
  - Java:       @Configuration + @Bean
```

## Vi du day du:
```java
// 1. Dinh nghia bean bang annotation
@Service
public class UserService {
    @Autowired
    private UserRepository userRepo;
    
    public User findUser(Long id) {
        return userRepo.findById(id).orElse(null);
    }
}

// 2. Container tu dong:
//    - Quet tim @Service -> tao UserService
//    - Quet tim @Repository -> tao UserRepository
//    - Inject UserRepository vao UserService
//    - Quan ly vong doi cua tat ca bean
```

## Diem quan trong nho phong van
1. IoC Container = **Dao nguoc quyen dieu khien** (container tao object, khong phai developer)
2. **2 loai**: BeanFactory (co ban) va ApplicationContext (thuong dung)
3. Container doc **metadata** (XML/annotation/Java config) de biet cach tao bean
4. Loi ich chinh: **Loose coupling**, de test, de thay doi implementation
