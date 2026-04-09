# Q6: What is the typical Bean life cycle in Spring Bean Factory Container?
> **Dịch:** Vòng đời điển hình của Bean trong Spring Bean Factory Container là gì?

## Tra loi ngan gon
> Bean lifecycle gom cac giai doan: **Tao -> Inject Dependencies -> Init -> Su dung -> Destroy**. Spring container quan ly toan bo qua trinh nay.

## Cach nho
```
Bean lifecycle giong nhu doi nguoi:
1. Sinh ra          = Instantiation (new)
2. Di hoc           = Populate Properties (inject dependencies)
3. Nhan ten         = BeanNameAware
4. Biet nha         = BeanFactoryAware
5. Tap the duc sang = @PostConstruct (init)
6. Di lam           = Ready to use
7. Ve huu           = @PreDestroy (destroy)
```

## So do lifecycle chi tiet

```
Container tao Bean (new)
        |
        v
Inject Dependencies (set properties)
        |
        v
BeanNameAware.setBeanName()        -- bean biet ten cua minh
        |
        v
BeanFactoryAware.setBeanFactory()  -- bean biet factory nao tao no
        |
        v
ApplicationContextAware.setApplicationContext()
        |
        v
BeanPostProcessor.postProcessBeforeInitialization()
        |
        v
@PostConstruct / InitializingBean.afterPropertiesSet() / init-method
        |
        v
BeanPostProcessor.postProcessAfterInitialization()
        |
        v
===== BEAN SAN SANG SU DUNG =====
        |
        v
@PreDestroy / DisposableBean.destroy() / destroy-method
        |
        v
Bean bi huy
```

## Vi du thuc te

```java
@Component
public class UserService implements InitializingBean, DisposableBean,
                                     BeanNameAware {
    
    @Autowired
    private UserRepository userRepo; // Buoc 2: Inject dependency

    private String beanName;

    // Buoc 3: Biet ten cua minh
    @Override
    public void setBeanName(String name) {
        this.beanName = name;
        System.out.println("1. setBeanName: " + name);
    }

    // Buoc 5a: @PostConstruct - THUONG DUNG NHAT
    @PostConstruct
    public void init() {
        System.out.println("2. @PostConstruct - Khoi tao tai nguyen");
        // Load cache, kiem tra config, v.v.
    }

    // Buoc 5b: InitializingBean
    @Override
    public void afterPropertiesSet() {
        System.out.println("3. afterPropertiesSet");
    }

    // ===== SU DUNG BEAN =====

    // Buoc 7a: @PreDestroy - THUONG DUNG NHAT
    @PreDestroy
    public void cleanup() {
        System.out.println("4. @PreDestroy - Don dep tai nguyen");
        // Dong connection, giai phong resource
    }

    // Buoc 7b: DisposableBean
    @Override
    public void destroy() {
        System.out.println("5. destroy");
    }
}
```

### Output khi chay:
```
1. setBeanName: userService
2. @PostConstruct - Khoi tao tai nguyen
3. afterPropertiesSet
--- App dang chay ---
4. @PreDestroy - Don dep tai nguyen
5. destroy
```

## Thu tu uu tien cua Init methods

```
Uu tien 1: @PostConstruct        (annotation - khuyen dung)
Uu tien 2: afterPropertiesSet()  (interface InitializingBean)
Uu tien 3: custom init-method    (XML/Java config)

Tuong tu cho Destroy:
Uu tien 1: @PreDestroy
Uu tien 2: destroy()             (interface DisposableBean)
Uu tien 3: custom destroy-method
```

## Diem quan trong nho phong van
1. **@PostConstruct** va **@PreDestroy** la cach don gian nhat (khuyen dung)
2. Prototype bean **KHONG** goi destroy method (container khong quan ly)
3. **BeanPostProcessor** chay cho **TAT CA** bean (dung de customize)
4. Thu tu: Constructor -> DI -> Init -> Use -> Destroy
