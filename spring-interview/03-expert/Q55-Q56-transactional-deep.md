# Q55: Where does the @Transactional annotation belong?
> **Dịch:** @Transactional nên đặt ở đâu?

# Q56: What actually happens when you annotate a method with @Transactional?
> **Dịch:** Điều gì thực sự xảy ra khi đánh dấu method bằng @Transactional?

---

## Q55: @Transactional dat o dau?

### Tra loi ngan gon
> Dat o **Service layer** (khong phai Controller hay Repository). Co the dat o **class level** (ap dung cho tat ca method) hoac **method level** (chi method do).

### Best practices
```java
// DUNG - Dat o Service layer
@Service
@Transactional(readOnly = true) // Mac dinh readonly cho tat ca method
public class UserService {

    public List<User> findAll() { // readOnly = true (ke thua class)
        return userRepo.findAll();
    }

    @Transactional // Override: readOnly = false cho method write
    public User create(UserDto dto) {
        return userRepo.save(toEntity(dto));
    }
}

// SAI - Khong nen dat o Controller
@RestController
public class UserController {
    @Transactional // KHONG NEN! Controller khong nen biet ve transaction
    @PostMapping("/users")
    public User create(@RequestBody UserDto dto) { }
}

// SAI - Khong nen dat o Repository (qua thap)
@Repository
public interface UserRepo extends JpaRepository<User, Long> {
    @Transactional // Khong can - Spring Data da co san
    List<User> findByName(String name);
}
```

### Tai sao Service layer?
```
Controller - khong biet ve database
Service    - biet LOGIC nao can transaction  <-- DAT O DAY
Repository - moi method la 1 query don le
```

---

## Q56: Dieu gi xay ra khi dung @Transactional?

### Tra loi ngan gon
> Spring tao **proxy** bao quanh bean. Khi method duoc goi, proxy: (1) mo transaction, (2) goi method goc, (3) commit neu thanh cong / rollback neu exception.

### So do chi tiet

```
Client goi: userService.create(dto)
     |
     v
+------------------------------------------+
| TransactionProxy (Spring tao tu dong)     |
|                                          |
| 1. Lay TransactionManager               |
| 2. transactionManager.getTransaction()   |
|    -> Lay connection tu DataSource       |
|    -> connection.setAutoCommit(false)    |
|                                          |
| 3. Goi TARGET method: create(dto)        |
|    |                                     |
|    +---> userRepo.save(entity)           |
|    |     (dung CUNG connection)          |
|    |                                     |
| 4a. Thanh cong -> connection.commit()    |
| 4b. Exception  -> connection.rollback()  |
|                                          |
| 5. Tra connection ve pool               |
+------------------------------------------+
```

### Code tuong duong cua proxy

```java
// Spring TAO PROXY tuong duong nhu nay:
public class UserService$$Proxy extends UserService {
    
    private TransactionManager txManager;
    private UserService target; // Bean goc

    @Override
    public User create(UserDto dto) {
        TransactionStatus tx = txManager.getTransaction(
            new DefaultTransactionDefinition());
        try {
            User result = target.create(dto); // Goi method THAT
            txManager.commit(tx);             // Commit
            return result;
        } catch (RuntimeException e) {
            txManager.rollback(tx);           // Rollback
            throw e;
        }
    }
}
```

### Cac buoc noi bo chi tiet

```
1. TransactionInterceptor nhan cuoc goi
2. Kiem tra TransactionAttribute (propagation, isolation, timeout...)
3. PlatformTransactionManager.getTransaction()
   - Voi JDBC: DataSourceTransactionManager
   - Voi JPA:  JpaTransactionManager
4. Luu TransactionStatus vao ThreadLocal (TransactionSynchronizationManager)
5. Thuc thi method goc
6. Neu thanh cong:
   - Flush EntityManager (JPA)
   - connection.commit()
7. Neu exception:
   - Kiem tra rollbackFor / noRollbackFor
   - connection.rollback() (neu can)
8. Don dep: tra connection, clear ThreadLocal
```

### Luu y QUAN TRONG: Self-invocation

```java
@Service
public class UserService {
    
    public void methodA() {
        this.methodB(); // GOI TRUC TIEP -> bypass proxy -> @Transactional BI BO QUA!
    }

    @Transactional
    public void methodB() {
        // Transactional KHONG hoat dong khi goi tu methodA!
    }
}

// GIAI PHAP 1: Inject self
@Service
public class UserService {
    @Autowired
    private UserService self; // Inject proxy cua chinh minh

    public void methodA() {
        self.methodB(); // Goi qua PROXY -> @Transactional HOAT DONG
    }
}

// GIAI PHAP 2: Tach ra service khac
```

## Diem quan trong nho phong van
1. Dat @Transactional o **Service layer** (best practice)
2. Spring tao **PROXY** de quan ly transaction tu dong
3. Proxy dung **TransactionManager** + **ThreadLocal** luu trang thai
4. **Self-invocation** khong hoat dong vi **bypass proxy**
5. Mac dinh chi rollback **RuntimeException** (unchecked)
