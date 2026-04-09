# Q21: Is the DispatcherServlet instantiated via an application context?
> **Dịch:** DispatcherServlet có được tạo bởi Application Context không?

## Tra loi ngan gon
> **Khong hoan toan.** DispatcherServlet duoc **Servlet Container** (Tomcat) tao ra, nhung no se tu tao **WebApplicationContext rieng** (child context) de quan ly cac bean lien quan den web (controllers, view resolvers). Context nay ke thua tu **Root ApplicationContext** (parent).

## Cach nho
```
Root Context (cha)     = Tong giam doc (quan ly Service, Repository)
Web Context (con)      = Truong phong kinh doanh (quan ly Controller, ViewResolver)
DispatcherServlet      = Nhan vien le tan (do Web Context tao ra)
```

## Kien truc 2 tang Context

```
+------------------------------------------+
| Root ApplicationContext                   |
| (ContextLoaderListener tao)              |
|                                          |
| @Service, @Repository, DataSource,       |
| Transaction Manager, Security Config     |
+------------------------------------------+
           |  (parent)
           v
+------------------------------------------+
| WebApplicationContext                     |
| (DispatcherServlet tao)                  |
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
        // SpringApplication.run() lam tat ca:
        // 1. Tao ApplicationContext
        // 2. Tao DispatcherServlet
        // 3. Dang ky vao embedded Tomcat
        SpringApplication.run(MyApp.class, args);
    }
}
// Spring Boot don gian hoa: chi co 1 context duy nhat
```

## Diem quan trong nho phong van
1. DispatcherServlet duoc **Servlet Container** (Tomcat) tao
2. No tu tao **WebApplicationContext** rieng (child context)
3. Child context truy cap duoc bean cua Parent, **khong nguoc lai**
4. Spring Boot don gian hoa: thuong chi co **1 context**
