# Q28: What is Bean Factory?
> **Dịch:** Bean Factory là gì?

## Trả lời ngắn gọn
> **BeanFactory** là IoC container **cơ bản nhất** trong Spring, cung cấp cơ chế **Dependency Injection**. Nó dùng **lazy loading** - chỉ tạo bean khi được yêu cầu (getBean()). Là interface gốc mà ApplicationContext kế thừa.

## Cách nhớ
```
BeanFactory = Tiệm tạp hóa nhỏ (chỉ bán hàng khi có người hỏi mua)
ApplicationContext = Siêu thị (bày sẵn hàng, có thêm dịch vụ)
```

## Ví dụ sử dụng

```java
// Tạo BeanFactory từ XML (cách cũ)
BeanFactory factory = new XmlBeanFactory(
    new ClassPathResource("beans.xml")
);

// Lúc này bean CHƯA được tạo

// Bean chỉ được tạo khi gọi getBean()
UserService service = factory.getBean("userService", UserService.class);
```

## Interface hierarchy

```
BeanFactory (interface cơ bản)
    |
    +-- ListableBeanFactory
    |
    +-- HierarchicalBeanFactory
    |
    +-- ApplicationContext (kế thừa tất cả + thêm tính năng)
            |
            +-- WebApplicationContext
            +-- AnnotationConfigApplicationContext
```

## Đặc điểm của BeanFactory

| Đặc điểm | Mô tả |
|----------|-------|
| Lazy loading | Bean tạo khi getBean() |
| DI | Hỗ trợ đầy đủ |
| Lifecycle | Hỗ trợ cơ bản |
| Event | Không |
| i18n | Không |
| AOP | Hạn chế |

## Điểm quan trọng nhớ phỏng vấn
1. BeanFactory là container **cơ bản nhất**
2. **Lazy loading** - chỉ tạo bean khi cần
3. Trong thực tế luôn dùng **ApplicationContext** thay vì BeanFactory
4. `XmlBeanFactory` đã **deprecated** từ Spring 3.1
