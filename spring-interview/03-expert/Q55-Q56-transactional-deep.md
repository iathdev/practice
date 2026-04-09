# Q55: Where does the @Transactional annotation belong?
> **Dịch:** @Transactional nên đặt ở đâu?

# Q56: What actually happens when you annotate a method with @Transactional?
> **Dịch:** Điều gì thực sự xảy ra khi đánh dấu method bằng @Transactional?

---

## Q55: @Transactional đặt ở đâu?

### Trả lời ngắn gọn
> Đặt ở **Service layer** (không phải Controller hay Repository). Có thể đặt ở **class level** (áp dụng cho tất cả method) hoặc **method level** (chỉ method đó).

### Best practices
```java
// ĐÚNG - Đặt ở Service layer
@Service
@Transactional(readOnly = true) // Mặc định readonly cho tất cả method
public class UserService {

    public List<User> findAll() { // readOnly = true (kế thừa class)
        return userRepo.findAll();
    }

    @Transactional // Override: readOnly = false cho method write
    public User create(UserDto dto) {
        return userRepo.save(toEntity(dto));
    }
}

// SAI - Không nên đặt ở Controller
@RestController
public class UserController {
    @Transactional // KHÔNG NÊN! Controller không nên biết về transaction
    @PostMapping("/users")
    public User create(@RequestBody UserDto dto) { }
}

// SAI - Không nên đặt ở Repository (quá thấp)
@Repository
public interface UserRepo extends JpaRepository<User, Long> {
    @Transactional // Không cần - Spring Data đã có sẵn
    List<User> findByName(String name);
}
```

### Tại sao Service layer?
```
Controller - không biết về database
Service    - biết LOGIC nào cần transaction  <-- ĐẶT Ở ĐÂY
Repository - mỗi method là 1 query đơn lẻ
```

---

## Q56: Điều gì xảy ra khi dùng @Transactional?

### Trả lời ngắn gọn
> Spring tạo **proxy** bao quanh bean. Khi method được gọi, proxy: (1) mở transaction, (2) gọi method gốc, (3) commit nếu thành công / rollback nếu exception.

### Sơ đồ chi tiết

```
Client gọi: userService.create(dto)
     |
     v
+------------------------------------------+
| TransactionProxy (Spring tạo tự động)     |
|                                          |
| 1. Lấy TransactionManager               |
| 2. transactionManager.getTransaction()   |
|    -> Lấy connection từ DataSource       |
|    -> connection.setAutoCommit(false)    |
|                                          |
| 3. Gọi TARGET method: create(dto)        |
|    |                                     |
|    +---> userRepo.save(entity)           |
|    |     (dùng CÙNG connection)          |
|    |                                     |
| 4a. Thành công -> connection.commit()    |
| 4b. Exception  -> connection.rollback()  |
|                                          |
| 5. Trả connection về pool               |
+------------------------------------------+
```

### Code tương đương của proxy

```java
// Spring TẠO PROXY tương đương như này:
public class UserService$$Proxy extends UserService {
    
    private TransactionManager txManager;
    private UserService target; // Bean gốc

    @Override
    public User create(UserDto dto) {
        TransactionStatus tx = txManager.getTransaction(
            new DefaultTransactionDefinition());
        try {
            User result = target.create(dto); // Gọi method THẬT
            txManager.commit(tx);             // Commit
            return result;
        } catch (RuntimeException e) {
            txManager.rollback(tx);           // Rollback
            throw e;
        }
    }
}
```

### Các bước nội bộ chi tiết

```
1. TransactionInterceptor nhận cuộc gọi
2. Kiểm tra TransactionAttribute (propagation, isolation, timeout...)
3. PlatformTransactionManager.getTransaction()
   - Với JDBC: DataSourceTransactionManager
   - Với JPA:  JpaTransactionManager
4. Lưu TransactionStatus vào ThreadLocal (TransactionSynchronizationManager)
5. Thực thi method gốc
6. Nếu thành công:
   - Flush EntityManager (JPA)
   - connection.commit()
7. Nếu exception:
   - Kiểm tra rollbackFor / noRollbackFor
   - connection.rollback() (nếu cần)
8. Dọn dẹp: trả connection, clear ThreadLocal
```

### Lưu ý QUAN TRỌNG: Self-invocation

```java
@Service
public class UserService {
    
    public void methodA() {
        this.methodB(); // GỌI TRỰC TIẾP -> bypass proxy -> @Transactional BỊ BỎ QUA!
    }

    @Transactional
    public void methodB() {
        // Transactional KHÔNG hoạt động khi gọi từ methodA!
    }
}

// GIẢI PHÁP 1: Inject self
@Service
public class UserService {
    @Autowired
    private UserService self; // Inject proxy của chính mình

    public void methodA() {
        self.methodB(); // Gọi qua PROXY -> @Transactional HOẠT ĐỘNG
    }
}

// GIẢI PHÁP 2: Tách ra service khác
```

## Điểm quan trọng nhớ phỏng vấn
1. Đặt @Transactional ở **Service layer** (best practice)
2. Spring tạo **PROXY** để quản lý transaction tự động
3. Proxy dùng **TransactionManager** + **ThreadLocal** lưu trạng thái
4. **Self-invocation** không hoạt động vì **bypass proxy**
5. Mặc định chỉ rollback **RuntimeException** (unchecked)
