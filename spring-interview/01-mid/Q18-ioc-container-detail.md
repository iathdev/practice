# Q18: What is Spring IoC Container?
> **Dịch:** Spring IoC Container là gì?

## Trả lời ngắn gọn
> Spring IoC Container là thành phần **cốt lõi** của Spring, chịu trách nhiệm **tạo, cấu hình và quản lý** các object (bean). Nó đọc **metadata** (annotation/XML/Java config) để biết cách tạo và wire các bean lại với nhau.

## Cách nhớ
```
IoC = Inversion of Control = Đảo ngược quyền điều khiển
  Trước: Developer tạo object (new UserService())
  Sau:   Container tạo object (container quyết định)

Giống như: Bạn không tự tuyển nhân viên, mà nhờ HR làm
```

## IoC Container làm những gì?

```
1. ĐỌC metadata (cấu hình)
   - @Component, @Service, @Repository, @Controller
   - @Configuration + @Bean
   - XML config (cũ)

2. TẠO bean (instantiate)
   - Gọi constructor
   - Quản lý singleton/prototype

3. INJECT dependency
   - Constructor injection
   - Setter injection
   - Field injection

4. QUẢN LÝ vòng đời
   - Init (@PostConstruct)
   - Use
   - Destroy (@PreDestroy)
```

## Ví dụ minh họa

```java
// 1. Định nghĩa các bean
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

// 2. Container tự động:
//    - Quét package -> tìm các class có @Component/@Service/@Repository/@Controller
//    - Tạo UserRepository
//    - Tạo UserService, inject UserRepository vào
//    - Tạo UserController, inject UserService vào
//    - Tất cả đều là singleton (mặc định)
```

## 2 loại Container

```
                  IoC Container
                       |
          +------------+------------+
          |                         |
     BeanFactory            ApplicationContext
     (cơ bản)               (nâng cao - DÙNG CÁI NÀY)
                                    |
                    +---------------+---------------+
                    |               |               |
    AnnotationConfig     ClassPathXml      WebApplicationContext
    ApplicationContext   ApplicationContext
```

## Điểm quan trọng nhớ phỏng vấn
1. IoC Container = **Bộ não** của Spring (quản lý mọi thứ)
2. Đọc **metadata** -> **Tạo** bean -> **Inject** dependency -> **Quản lý** lifecycle
3. 2 loại: **BeanFactory** (cơ bản) và **ApplicationContext** (thường dùng)
4. Spring Boot: `SpringApplication.run()` tạo ApplicationContext tự động
