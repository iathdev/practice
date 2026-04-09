# Q21: Is the DispatcherServlet instantiated via an application context?
> **Dịch:** DispatcherServlet có được tạo bởi Application Context không?

## Trả lời ngắn gọn
> **Không hoàn toàn.** DispatcherServlet được **Servlet Container** (Tomcat) tạo ra, nhưng nó sẽ tự tạo **WebApplicationContext riêng** (child context) để quản lý các bean liên quan đến web (controllers, view resolvers). Context này kế thừa từ **Root ApplicationContext** (parent).

## Cách nhớ
```
Root Context (cha)     = Tổng giám đốc (quản lý Service, Repository)
Web Context (con)      = Trưởng phòng kinh doanh (quản lý Controller, ViewResolver)
DispatcherServlet      = Nhân viên lễ tân (do Web Context tạo ra)
```

## Kiến trúc 2 tầng Context

```
+------------------------------------------+
| Root ApplicationContext                   |
| (ContextLoaderListener tạo)              |
|                                          |
| @Service, @Repository, DataSource,       |
| Transaction Manager, Security Config     |
+------------------------------------------+
           |  (parent)
           v
+------------------------------------------+
| WebApplicationContext                     |
| (DispatcherServlet tạo)                  |
|                                          |
| @Controller, ViewResolver,              |
| HandlerMapping, @ControllerAdvice        |
+------------------------------------------+
```

## Trong Spring Boot

```java
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        // SpringApplication.run() làm tất cả:
        // 1. Tạo ApplicationContext
        // 2. Tạo DispatcherServlet
        // 3. Đăng ký vào embedded Tomcat
        SpringApplication.run(MyApp.class, args);
    }
}
// Spring Boot đơn giản hóa: chỉ có 1 context duy nhất
```

## Điểm quan trọng nhớ phỏng vấn
1. DispatcherServlet được **Servlet Container** (Tomcat) tạo
2. Nó tự tạo **WebApplicationContext** riêng (child context)
3. Child context truy cập được bean của Parent, **không ngược lại**
4. Spring Boot đơn giản hóa: thường chỉ có **1 context**
