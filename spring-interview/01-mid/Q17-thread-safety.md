# Q17: Does Spring Bean provide thread safety?
> **Dịch:** Spring Bean có đảm bảo an toàn luồng (thread safety) không?

## Trả lời ngắn gọn
> **KHÔNG!** Spring **không đảm bảo** thread safety cho bean. Singleton bean được chia sẻ giữa nhiều thread, nên nếu bean chứa **mutable state** thì có thể bị **race condition**. Developer phải tự xử lý thread safety.

## Cách nhớ
```
Singleton bean = Căn hộ chung cư (nhiều người dùng chung)
  - Không có đồ cá nhân (stateless) -> An toàn
  - Có đồ cá nhân (stateful) -> Sẽ bị xung đột!
```

## Vấn đề: Singleton + State = NGUY HIỂM

```java
// SAI - KHÔNG THREAD SAFE!
@Service // Singleton - 1 instance chia sẻ cho nhiều thread
public class CounterService {
    private int count = 0; // STATE - chia sẻ giữa các thread

    public void increment() {
        count++; // Thread A và B gọi cùng lúc -> RACE CONDITION!
    }

    public int getCount() {
        return count;
    }
}

// Thread A: đọc count = 5
// Thread B: đọc count = 5
// Thread A: ghi count = 6
// Thread B: ghi count = 6  <- MẤT 1 lần increment!
```

## Giải pháp

### 1. KHÔNG chứa state (BEST PRACTICE)
```java
@Service
public class UserService {
    private final UserRepository userRepo; // dependency, không phải state

    public User findById(Long id) {
        return userRepo.findById(id).orElse(null);
        // Mọi dữ liệu đều là LOCAL variable -> thread safe
    }
}
```

### 2. Dùng Prototype scope
```java
@Component
@Scope("prototype") // Mỗi thread 1 instance riêng
public class ShoppingCart {
    private List<Item> items = new ArrayList<>(); // OK vì mỗi thread có riêng
}
```

### 3. Dùng ThreadLocal
```java
@Component
public class UserContext {
    private ThreadLocal<User> currentUser = new ThreadLocal<>();

    public void set(User user) { currentUser.set(user); }
    public User get() { return currentUser.get(); }
    public void clear() { currentUser.remove(); }
}
```

### 4. Dùng Synchronized / Lock
```java
@Service
public class CounterService {
    private final AtomicInteger count = new AtomicInteger(0);

    public void increment() {
        count.incrementAndGet(); // Thread safe
    }
}
```

### 5. Dùng ConcurrentHashMap thay HashMap
```java
@Component
public class CacheService {
    // SAI: private Map<String, Object> cache = new HashMap<>();
    private Map<String, Object> cache = new ConcurrentHashMap<>(); // Thread safe
}
```

## Bảng tóm tắt

| Tình huống | Thread safe? | Giải pháp |
|-----------|:---:|-----------|
| Singleton + không state | Có | Không cần làm gì |
| Singleton + mutable state | **KHÔNG** | Tránh state / dùng AtomicXxx |
| Prototype | Có | Mỗi thread 1 instance |
| Request/Session scope | Có | Gắn với 1 request/session |

## Điểm quan trọng nhớ phỏng vấn
1. Spring **KHÔNG** đảm bảo thread safety
2. Best practice: **Stateless** singleton bean (không chứa data thay đổi)
3. Nếu cần state: dùng **prototype**, **ThreadLocal**, hoặc **concurrent collections**
4. `final` field (dependency) **không phải** là state -> an toàn
