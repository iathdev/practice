# Q11: What is the difference between Bean Factory and ApplicationContext?
> **Dịch:** Sự khác nhau giữa BeanFactory và ApplicationContext là gì?

## Trả lời ngắn gọn
> **BeanFactory** là container cơ bản (lazy loading). **ApplicationContext** là container nâng cao, kế thừa BeanFactory và thêm nhiều tính năng (event, i18n, AOP, eager loading).

## Cách nhớ
```
BeanFactory        = Quán cơm bình dân (chỉ có cơm)
ApplicationContext = Nhà hàng 5 sao (cơm + nước + tráng miệng + phục vụ)
```

## Bảng so sánh chi tiết

| Tính năng | BeanFactory | ApplicationContext |
|-----------|:-----------:|:------------------:|
| Tạo & quản lý Bean | Có | Có |
| Dependency Injection | Có | Có |
| Bean loading | **Lazy** (tạo khi gọi) | **Eager** (tạo khi khởi động) |
| Event publishing | Không | Có |
| i18n (đa ngôn ngữ) | Không | Có |
| AOP auto proxy | Không | Có |
| BeanPostProcessor tự động | Thủ công | Tự động |
| Resource loading | Không | Có |
| Environment abstraction | Không | Có |
| Annotation support | Hạn chế | Đầy đủ |

## Ví dụ code

### BeanFactory
```java
// Lazy loading - bean chỉ được tạo khi getBean()
BeanFactory factory = new XmlBeanFactory(
    new ClassPathResource("beans.xml")
);
// Bean chưa được tạo ở đây

UserService service = factory.getBean(UserService.class);
// Bean ĐƯỢC TẠO tại thời điểm này
```

### ApplicationContext
```java
// Eager loading - TẤT CẢ singleton bean được tạo ngay khi khởi động
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
// Tất cả bean ĐÃ ĐƯỢC TẠO rồi

UserService service = ctx.getBean(UserService.class);
// Chỉ lấy bean đã có sẵn
```

## Các implementation của ApplicationContext

```java
// 1. Annotation-based (thường dùng nhất)
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);

// 2. XML-based
ApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml");

// 3. Web application
// WebApplicationContext - tự động tạo bởi Spring Boot

// 4. Spring Boot (thường dùng)
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        ApplicationContext ctx = SpringApplication.run(MyApp.class, args);
    }
}
```

## Khi nào dùng cái nào?

```
99% trường hợp -> ApplicationContext (khuyên dùng)
BeanFactory    -> Chỉ khi cần tiết kiệm memory (IoT, mobile)
                  hoặc cần lazy loading toàn bộ
```

## Điểm quan trọng nhớ phỏng vấn
1. ApplicationContext **kế thừa** BeanFactory + thêm nhiều tính năng
2. BeanFactory: **Lazy**, ApplicationContext: **Eager** (mặc định)
3. Luôn dùng **ApplicationContext** trong thực tế
4. Spring Boot tự động tạo ApplicationContext qua `SpringApplication.run()`
