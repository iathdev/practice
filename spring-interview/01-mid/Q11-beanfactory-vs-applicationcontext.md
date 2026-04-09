# Q11: What is the difference between Bean Factory and ApplicationContext?
> **Dịch:** Sự khác nhau giữa BeanFactory và ApplicationContext là gì?

## Tra loi ngan gon
> **BeanFactory** la container co ban (lazy loading). **ApplicationContext** la container nang cao, ke thua BeanFactory va them nhieu tinh nang (event, i18n, AOP, eager loading).

## Cach nho
```
BeanFactory        = Quan com binh dan (chi co com)
ApplicationContext = Nha hang 5 sao (com + nuoc + trang mieng + phuc vu)
```

## Bang so sanh chi tiet

| Tinh nang | BeanFactory | ApplicationContext |
|-----------|:-----------:|:------------------:|
| Tao & quan ly Bean | Co | Co |
| Dependency Injection | Co | Co |
| Bean loading | **Lazy** (tao khi goi) | **Eager** (tao khi khoi dong) |
| Event publishing | Khong | Co |
| i18n (da ngon ngu) | Khong | Co |
| AOP auto proxy | Khong | Co |
| BeanPostProcessor tu dong | Thu cong | Tu dong |
| Resource loading | Khong | Co |
| Environment abstraction | Khong | Co |
| Annotation support | Han che | Day du |

## Vi du code

### BeanFactory
```java
// Lazy loading - bean chi duoc tao khi getBean()
BeanFactory factory = new XmlBeanFactory(
    new ClassPathResource("beans.xml")
);
// Bean chua duoc tao o day

UserService service = factory.getBean(UserService.class);
// Bean DUOC TAO tai thoi diem nay
```

### ApplicationContext
```java
// Eager loading - TAT CA singleton bean duoc tao ngay khi khoi dong
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);
// Tat ca bean DA DUOC TAO roi

UserService service = ctx.getBean(UserService.class);
// Chi lay bean da co san
```

## Cac implementation cua ApplicationContext

```java
// 1. Annotation-based (thuong dung nhat)
ApplicationContext ctx = new AnnotationConfigApplicationContext(AppConfig.class);

// 2. XML-based
ApplicationContext ctx = new ClassPathXmlApplicationContext("beans.xml");

// 3. Web application
// WebApplicationContext - tu dong tao boi Spring Boot

// 4. Spring Boot (thuong dung)
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        ApplicationContext ctx = SpringApplication.run(MyApp.class, args);
    }
}
```

## Khi nao dung cai nao?

```
99% truong hop -> ApplicationContext (khuyen dung)
BeanFactory    -> Chi khi can tiet kiem memory (IoT, mobile)
                  hoac can lazy loading toan bo
```

## Diem quan trong nho phong van
1. ApplicationContext **ke thua** BeanFactory + them nhieu tinh nang
2. BeanFactory: **Lazy**, ApplicationContext: **Eager** (mac dinh)
3. Luon dung **ApplicationContext** trong thuc te
4. Spring Boot tu dong tao ApplicationContext qua `SpringApplication.run()`
