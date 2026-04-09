# Q18: What is Spring IoC Container?
> **Dịch:** Spring IoC Container là gì?

## Tra loi ngan gon
> Spring IoC Container la thanh phan **cot loi** cua Spring, chiu trach nhiem **tao, cau hinh va quan ly** cac object (bean). No doc **metadata** (annotation/XML/Java config) de biet cach tao va wire cac bean lai voi nhau.

## Cach nho
```
IoC = Inversion of Control = Dao nguoc quyen dieu khien
  Truoc: Developer tao object (new UserService())
  Sau:   Container tao object (container quyet dinh)

Giong nhu: Ban khong tu tuyen nhan vien, ma nho HR lam
```

## IoC Container lam nhung gi?

```
1. DOC metadata (cau hinh)
   - @Component, @Service, @Repository, @Controller
   - @Configuration + @Bean
   - XML config (cu)

2. TAO bean (instantiate)
   - Goi constructor
   - Quan ly singleton/prototype

3. INJECT dependency
   - Constructor injection
   - Setter injection
   - Field injection

4. QUAN LY vong doi
   - Init (@PostConstruct)
   - Use
   - Destroy (@PreDestroy)
```

## Vi du minh hoa

```java
// 1. Dinh nghia cac bean
@Repository
public class UserRepository {
    public User findById(Long id) { /*...*/ }
}

@Service
public class UserService {
    private final UserRepository repo;

    public UserService(UserRepository repo) { // Container inject repo
        this.repo = repo;
    }
}

@RestController
public class UserController {
    private final UserService service;

    public UserController(UserService service) { // Container inject service
        this.service = service;
    }
}

// 2. Container tu dong:
//    - Quet package -> tim cac class co @Component/@Service/@Repository/@Controller
//    - Tao UserRepository
//    - Tao UserService, inject UserRepository vao
//    - Tao UserController, inject UserService vao
//    - Tat ca deu la singleton (mac dinh)
```

## 2 loai Container

```
                  IoC Container
                       |
          +------------+------------+
          |                         |
     BeanFactory            ApplicationContext
     (co ban)               (nang cao - DUNG CAI NAY)
                                    |
                    +---------------+---------------+
                    |               |               |
    AnnotationConfig     ClassPathXml      WebApplicationContext
    ApplicationContext   ApplicationContext
```

## Diem quan trong nho phong van
1. IoC Container = **Nao** cua Spring (quan ly moi thu)
2. Doc **metadata** -> **Tao** bean -> **Inject** dependency -> **Quan ly** lifecycle
3. 2 loai: **BeanFactory** (co ban) va **ApplicationContext** (thuong dung)
4. Spring Boot: `SpringApplication.run()` tao ApplicationContext tu dong
