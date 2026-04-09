# Q28: What is Bean Factory?
> **Dịch:** Bean Factory là gì?

## Tra loi ngan gon
> **BeanFactory** la IoC container **co ban nhat** trong Spring, cung cap co che **Dependency Injection**. No dung **lazy loading** - chi tao bean khi duoc yeu cau (getBean()). La interface goc ma ApplicationContext ke thua.

## Cach nho
```
BeanFactory = Tiem tap hoa nho (chi ban hang khi co nguoi hoi mua)
ApplicationContext = Sieu thi (bay san hang, co them dich vu)
```

## Vi du su dung

```java
// Tao BeanFactory tu XML (cach cu)
BeanFactory factory = new XmlBeanFactory(
    new ClassPathResource("beans.xml")
);

// Luc nay bean CHUA duoc tao

// Bean chi duoc tao khi goi getBean()
UserService service = factory.getBean("userService", UserService.class);
```

## Interface hierarchy

```
BeanFactory (interface co ban)
    |
    +-- ListableBeanFactory
    |
    +-- HierarchicalBeanFactory
    |
    +-- ApplicationContext (ke thua tat ca + them tinh nang)
            |
            +-- WebApplicationContext
            +-- AnnotationConfigApplicationContext
```

## Dac diem cua BeanFactory

| Dac diem | Mo ta |
|----------|-------|
| Lazy loading | Bean tao khi getBean() |
| DI | Ho tro day du |
| Lifecycle | Ho tro co ban |
| Event | Khong |
| i18n | Khong |
| AOP | Han che |

## Diem quan trong nho phong van
1. BeanFactory la container **co ban nhat**
2. **Lazy loading** - chi tao bean khi can
3. Trong thuc te luon dung **ApplicationContext** thay vi BeanFactory
4. `XmlBeanFactory` da **deprecated** tu Spring 3.1
