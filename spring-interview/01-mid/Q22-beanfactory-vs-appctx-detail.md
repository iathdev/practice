# Q22: What is the difference between Bean Factory and Application Context?
> **Dịch:** Sự khác nhau giữa Bean Factory và Application Context là gì?

## Tra loi ngan gon
> (Cau nay tuong tu Q11) **BeanFactory** cung cap co che DI co ban voi lazy loading. **ApplicationContext** ke thua BeanFactory, them **event publishing, i18n, AOP, eager loading** va nhieu tinh nang enterprise khac.

## Bang so sanh nhanh

```
BeanFactory                    ApplicationContext
+-----------+                  +------------------+
| DI co ban |                  | DI co ban        |
|           |                  | + Event system   |
|           |                  | + i18n           |
|           |                  | + AOP            |
|           |                  | + Eager loading  |
|           |                  | + Environment    |
|           |                  | + Resource loader|
+-----------+                  +------------------+
   Xe dap                         Oto
```

## Dac biet: Event System (chi co trong ApplicationContext)

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

## Khi nao dung BeanFactory?
- Ung dung **rat nho**, can tiet kiem memory
- Embedded system, IoT
- **Hau nhu khong bao gio** dung trong thuc te

## Ket luan
> Luon dung **ApplicationContext**. Spring Boot mac dinh dung `AnnotationConfigServletWebServerApplicationContext`.
