# Q6: What is the typical Bean life cycle in Spring Bean Factory Container?
> **Dịch:** Vòng đời điển hình của Bean trong Spring Bean Factory Container là gì?

## Trả lời ngắn gọn
> Bean lifecycle gồm các giai đoạn: **Tạo -> Inject Dependencies -> Init -> Sử dụng -> Destroy**. Spring container quản lý toàn bộ quá trình này.

## Cách nhớ
```
Bean lifecycle giống như đời người:
1. Sinh ra          = Instantiation (new)
2. Đi học           = Populate Properties (inject dependencies)
3. Nhận tên         = BeanNameAware
4. Biết nhà         = BeanFactoryAware
5. Tập thể dục sáng = @PostConstruct (init)
6. Đi làm           = Ready to use
7. Về hưu           = @PreDestroy (destroy)
```

## Sơ đồ lifecycle chi tiết

```
Container tạo Bean (new)
        |
        v
Inject Dependencies (set properties)
        |
        v
BeanNameAware.setBeanName()        -- bean biết tên của mình
        |
        v
BeanFactoryAware.setBeanFactory()  -- bean biết factory nào tạo nó
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
===== BEAN SẴN SÀNG SỬ DỤNG =====
        |
        v
@PreDestroy / DisposableBean.destroy() / destroy-method
        |
        v
Bean bị hủy
```

## Ví dụ thực tế

```java
@Component
public class UserService implements InitializingBean, DisposableBean,
                                     BeanNameAware {
    
    @Autowired
    private UserRepository userRepo; // Bước 2: Inject dependency

    private String beanName;

    // Bước 3: Biết tên của mình
    @Override
    public void setBeanName(String name) {
        this.beanName = name;
        System.out.println("1. setBeanName: " + name);
    }

    // Bước 5a: @PostConstruct - THƯỜNG DÙNG NHẤT
    @PostConstruct
    public void init() {
        System.out.println("2. @PostConstruct - Khởi tạo tài nguyên");
        // Load cache, kiểm tra config, v.v.
    }

    // Bước 5b: InitializingBean
    @Override
    public void afterPropertiesSet() {
        System.out.println("3. afterPropertiesSet");
    }

    // ===== SỬ DỤNG BEAN =====

    // Bước 7a: @PreDestroy - THƯỜNG DÙNG NHẤT
    @PreDestroy
    public void cleanup() {
        System.out.println("4. @PreDestroy - Dọn dẹp tài nguyên");
        // Đóng connection, giải phóng resource
    }

    // Bước 7b: DisposableBean
    @Override
    public void destroy() {
        System.out.println("5. destroy");
    }
}
```

### Output khi chạy:
```
1. setBeanName: userService
2. @PostConstruct - Khởi tạo tài nguyên
3. afterPropertiesSet
--- App đang chạy ---
4. @PreDestroy - Dọn dẹp tài nguyên
5. destroy
```

## Thứ tự ưu tiên của Init methods

```
Ưu tiên 1: @PostConstruct        (annotation - khuyên dùng)
Ưu tiên 2: afterPropertiesSet()  (interface InitializingBean)
Ưu tiên 3: custom init-method    (XML/Java config)

Tương tự cho Destroy:
Ưu tiên 1: @PreDestroy
Ưu tiên 2: destroy()             (interface DisposableBean)
Ưu tiên 3: custom destroy-method
```

## Điểm quan trọng nhớ phỏng vấn
1. **@PostConstruct** và **@PreDestroy** là cách đơn giản nhất (khuyên dùng)
2. Prototype bean **KHÔNG** gọi destroy method (container không quản lý)
3. **BeanPostProcessor** chạy cho **TẤT CẢ** bean (dùng để customize)
4. Thứ tự: Constructor -> DI -> Init -> Use -> Destroy
