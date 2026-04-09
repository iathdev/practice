# Q22: What is the difference between Bean Factory and Application Context?
> **Dịch:** Sự khác nhau giữa Bean Factory và Application Context là gì?

## Trả lời ngắn gọn
> (Câu này tương tự Q11) **BeanFactory** cung cấp cơ chế DI cơ bản với lazy loading. **ApplicationContext** kế thừa BeanFactory, thêm **event publishing, i18n, AOP, eager loading** và nhiều tính năng enterprise khác.

## Bảng so sánh nhanh

```
BeanFactory                    ApplicationContext
+-----------+                  +------------------+
| DI cơ bản |                  | DI cơ bản        |
|           |                  | + Event system   |
|           |                  | + i18n           |
|           |                  | + AOP            |
|           |                  | + Eager loading  |
|           |                  | + Environment    |
|           |                  | + Resource loader|
+-----------+                  +------------------+
   Xe đạp                         Ôtô
```

## Đặc biệt: Event System (chỉ có trong ApplicationContext)

```java
// Publish event
@Service
public class UserService {
    @Autowired
    private ApplicationEventPublisher publisher;

    public void register(User user) {
        userRepo.save(user);
        publisher.publishEvent(new UserRegisteredEvent(user));
    }
}

// Listen event
@Component
public class WelcomeEmailListener {
    @EventListener
    public void onRegister(UserRegisteredEvent event) {
        emailService.sendWelcome(event.getUser());
    }
}
```

## Khi nào dùng BeanFactory?
- Ứng dụng **rất nhỏ**, cần tiết kiệm memory
- Embedded system, IoT
- **Hầu như không bao giờ** dùng trong thực tế

## Kết luận
> Luôn dùng **ApplicationContext**. Spring Boot mặc định dùng `AnnotationConfigServletWebServerApplicationContext`.
