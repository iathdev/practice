# Q10: What are the types of the transaction management Spring supports?
> **Dịch:** Spring hỗ trợ những loại quản lý transaction nào?

## Trả lời ngắn gọn
> Spring hỗ trợ **2 loại** quản lý transaction:
> 1. **Programmatic** - code thủ công trong method
> 2. **Declarative** - dùng annotation `@Transactional` (KHUYÊN DÙNG)

## Cách nhớ
```
Programmatic  = Lái xe số sàn (tự sang số, đạp côn, nhả côn)
Declarative   = Lái xe số tự động (chỉ cần đạp ga, xe tự làm)
```

## Loại 1: Programmatic Transaction (thủ công)

```java
@Service
public class OrderService {
    @Autowired
    private TransactionTemplate transactionTemplate;

    public void createOrder(Order order) {
        transactionTemplate.execute(status -> {
            try {
                orderRepo.save(order);
                paymentService.charge(order);
                inventoryService.reduce(order);
                return null; // commit
            } catch (Exception e) {
                status.setRollbackOnly(); // rollback
                throw e;
            }
        });
    }
}
```
- Ưu: Kiểm soát chi tiết
- Nhược: Code dài, khó bảo trì

## Loại 2: Declarative Transaction (annotation - KHUYÊN DÙNG)

```java
@Service
public class OrderService {

    @Transactional  // Chỉ cần 1 annotation!
    public void createOrder(Order order) {
        orderRepo.save(order);        // Thành công
        paymentService.charge(order); // Thành công
        inventoryService.reduce(order); // THẤT BẠI -> Rollback TẤT CẢ!
    }
}
```
- Ưu: Đơn giản, sạch sẽ, dễ bảo trì
- Nhược: Ít kiểm soát chi tiết hơn

## Các thuộc tính của @Transactional

```java
@Transactional(
    propagation = Propagation.REQUIRED,      // Mặc định
    isolation = Isolation.READ_COMMITTED,    // Mức cô lập
    timeout = 30,                            // Timeout (giây)
    readOnly = false,                        // Đọc/ghi
    rollbackFor = Exception.class,           // Rollback khi nào
    noRollbackFor = MailException.class      // Không rollback khi nào
)
public void myMethod() { }
```

## Propagation (Lan truyền transaction)

```
REQUIRED (mặc định)
  A() gọi B(): B dùng chung transaction với A
  |-------- Transaction A ---------|
       |--- B dùng chung ---|

REQUIRES_NEW
  A() gọi B(): B tạo transaction MỚI, tạm dừng A
  |-------- Transaction A (tạm dừng) ---------|
       |--- Transaction B (mới) ---|

NESTED
  A() gọi B(): B tạo savepoint trong A
  |-------- Transaction A ---------|
       |--- Savepoint B ---|
```

| Propagation | Mô tả |
|-------------|-------|
| REQUIRED | Dùng transaction hiện tại, tạo mới nếu chưa có |
| REQUIRES_NEW | Luôn tạo transaction mới |
| SUPPORTS | Có transaction thì dùng, không có thì thôi |
| NOT_SUPPORTED | Không dùng transaction |
| MANDATORY | BẮT BUỘC phải có transaction sẵn |
| NEVER | BẮT BUỘC không có transaction |
| NESTED | Tạo savepoint (nested) |

## Lưu ý quan trọng

```java
// SAI - Tự gọi nội bộ, @Transactional KHÔNG hoạt động!
@Service
public class UserService {
    public void methodA() {
        this.methodB(); // Không qua proxy -> @Transactional bị bỏ qua!
    }

    @Transactional
    public void methodB() { }
}

// ĐÚNG - Gọi từ bean khác
@Service
public class OrderService {
    @Autowired
    private UserService userService;

    public void process() {
        userService.methodB(); // Qua proxy -> @Transactional hoạt động!
    }
}
```

## Điểm quan trọng nhớ phỏng vấn
1. **Declarative** (@Transactional) là best practice
2. Mặc định chỉ **rollback RuntimeException** (unchecked), muốn rollback tất cả: `rollbackFor = Exception.class`
3. **Self-invocation** không hoạt động vì bypass proxy
4. `@Transactional(readOnly = true)` cho các method chỉ đọc -> **tăng performance**
