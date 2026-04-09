# Q17: Does Spring Bean provide thread safety?
> **Dịch:** Spring Bean có đảm bảo an toàn luồng (thread safety) không?

## Tra loi ngan gon
> **KHONG!** Spring **khong dam bao** thread safety cho bean. Singleton bean duoc chia se giua nhieu thread, nen neu bean chua **mutable state** thi co the bi **race condition**. Developer phai tu xu ly thread safety.

## Cach nho
```
Singleton bean = Can ho chung cu (nhieu nguoi dung chung)
  - Khong co do ca nhan (stateless) -> An toan
  - Co do ca nhan (stateful) -> Se bi xung dot!
```

## Van de: Singleton + State = NGUY HIEM

```java
// SAI - KHONG THREAD SAFE!
@Service // Singleton - 1 instance chia se cho nhieu thread
public class CounterService {
    private int count = 0; // STATE - chia se giua cac thread

    public void increment() {
        count++; // Thread A va B goi cung luc -> RACE CONDITION!
    }

    public int getCount() {
        return count;
    }
}

// Thread A: doc count = 5
// Thread B: doc count = 5
// Thread A: ghi count = 6
// Thread B: ghi count = 6  <- MAT 1 lan increment!
```

## Giai phap

### 1. KHONG chua state (BEST PRACTICE)
```java
@Service
public class UserService {
    private final UserRepository userRepo; // dependency, khong phai state

    public User findById(Long id) {
        return userRepo.findById(id).orElse(null);
        // Moi du lieu deu la LOCAL variable -> thread safe
    }
}
```

### 2. Dung Prototype scope
```java
@Component
@Scope("prototype") // Moi thread 1 instance rieng
public class ShoppingCart {
    private List<Item> items = new ArrayList<>(); // OK vi moi thread co rieng
}
```

### 3. Dung ThreadLocal
```java
@Component
public class UserContext {
    private ThreadLocal<User> currentUser = new ThreadLocal<>();

    public void set(User user) { currentUser.set(user); }
    public User get() { return currentUser.get(); }
    public void clear() { currentUser.remove(); }
}
```

### 4. Dung Synchronized / Lock
```java
@Service
public class CounterService {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        count.incrementAndGet(); // Thread safe
    }
}
```

### 5. Dung ConcurrentHashMap thay HashMap
```java
@Component
public class CacheService {
    // SAI: private Map<String, Object> cache = new HashMap<>();
    private Map<String, Object> cache = new ConcurrentHashMap<>(); // Thread safe
}
```

## Bang tom tat

| Tinh huong | Thread safe? | Giai phap |
|-----------|:---:|-----------|
| Singleton + khong state | Co | Khong can lam gi |
| Singleton + mutable state | **KHONG** | Tranh state / dung AtomicXxx |
| Prototype | Co | Moi thread 1 instance |
| Request/Session scope | Co | Gan voi 1 request/session |

## Diem quan trong nho phong van
1. Spring **KHONG** dam bao thread safety
2. Best practice: **Stateless** singleton bean (khong chua data thay doi)
3. Neu can state: dung **prototype**, **ThreadLocal**, hoac **concurrent collections**
4. `final` field (dependency) **khong phai** la state -> an toan
