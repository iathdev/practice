# Java Interview Questions — Expert (Q125–Q137)

---

## Q125. Does GC occur in Permanent Generation (Perm Gen)?

**Có, nhưng ít và khác với heap GC.**

**Perm Gen GC (Java 7 trở về):**
- Xảy ra khi Perm Gen đầy
- Thu hồi: Class definitions không còn dùng, Interned Strings
- Trigger: Full GC (không phải Minor GC)
- Ít phổ biến hơn heap GC vì classes thường không bị unload

**Điều kiện class được GC từ Perm Gen:**
```java
// Class được GC khi:
// 1. Không còn instance nào tồn tại
// 2. ClassLoader đã bị GC
// 3. Class object không còn reference

// Ví dụ: Dynamic class generation (Proxies, CGLIB)
// Nếu tạo quá nhiều → PermGen OutOfMemoryError

// Java 7: -XX:PermSize=256m -XX:MaxPermSize=512m
// Java 8+: Perm Gen → Metaspace (native memory, không bị OOM như Perm Gen)
```

**Java 8+: Metaspace thay Perm Gen:**
```
Java 8 thay đổi:
- Metaspace trong native memory → tự động grow
- Không còn -XX:MaxPermSize
- Thay bằng -XX:MaxMetaspaceSize=256m
- Class GC vẫn xảy ra trong Metaspace
```

**OutOfMemoryError:**
- Java 7: `OutOfMemoryError: PermGen space`
- Java 8+: `OutOfMemoryError: Metaspace`

---

## Q126. What does `synchronized` mean in Java?

`synchronized` là cơ chế **mutual exclusion** — đảm bảo chỉ một thread có thể thực thi code đã synchronized tại một thời điểm.

**Cơ chế hoạt động — Monitor Lock:**
```java
// Mỗi Java object có một "monitor" (intrinsic lock)
// synchronized block/method = acquire monitor khi vào, release khi ra

public class BankAccount {
    private double balance;
    
    // synchronized method — lock on 'this'
    public synchronized void deposit(double amount) {
        // Thread phải acquire monitor của 'this'
        balance += amount;
        // Monitor tự động release khi ra khỏi method
    }
    
    // synchronized block — lock on specific object
    public void withdraw(double amount) {
        synchronized(this) {
            if (balance >= amount) {
                balance -= amount;
            }
        }
    }
    
    // static synchronized — lock on Class object
    public static synchronized void resetAll() {
        // Lock on BankAccount.class
    }
}
```

**Synchronized đảm bảo:**
1. **Mutual exclusion:** Một thread tại một thời điểm
2. **Visibility:** Thay đổi của thread này visible với thread tiếp theo acquire lock
3. **Ordering:** Happens-before relationship

**Synchronized KHÔNG đảm bảo:**
```java
volatile int count = 0;
// synchronized bảo vệ
synchronized(this) { count++; } // thread-safe

// Nhưng hai synchronized blocks riêng KHÔNG atomic cùng nhau:
synchronized(this) { int temp = count; }
// ... thread khác có thể chen vào ở đây
synchronized(this) { count = temp + 1; }
```

**Reentrant:**
```java
public synchronized void outer() {
    inner(); // OK — cùng thread có thể acquire lại monitor
}
public synchronized void inner() {} // reentrant lock
```

---

## Q127. What is the RMI Architecture?

**RMI có 3 layers (tầng):**

```
┌─────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                     │
│  Client Code ──────→ Remote Interface ←────── Server    │
└─────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────┐
│                      PROXY LAYER                         │
│  Stub (client proxy)  ←──────────→  Skeleton (server)  │
│  - Marshal args                     - Unmarshal args     │
│  - Unmarshal result                 - Marshal result     │
│  - Call transport                   - Invoke method      │
└─────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────┐
│                    TRANSPORT LAYER                        │
│  TCP/IP Connection (JRMP - Java Remote Method Protocol)  │
│  or IIOP (for CORBA interop)                            │
└─────────────────────────────────────────────────────────┘
```

**Các thành phần:**
```java
// 1. Remote Interface — Contract
public interface Service extends Remote {
    String process(String data) throws RemoteException;
}

// 2. Implementation — Server-side object
public class ServiceImpl extends UnicastRemoteObject implements Service {
    public ServiceImpl() throws RemoteException { super(); }
    
    @Override
    public String process(String data) throws RemoteException {
        return data.toUpperCase();
    }
}

// 3. RMI Registry — Naming service
Registry registry = LocateRegistry.createRegistry(1099);
registry.bind("Service", new ServiceImpl());

// 4. Stub — Auto-generated proxy
// (Java 5+: không cần rmic, dynamic proxy)
Service stub = (Service) registry.lookup("Service");

// 5. Transport — JRMP over TCP
// Tự động xử lý bởi java.rmi package
```

---

## Q128. What is DGC (Distributed Garbage Collection)?

**DGC** là cơ chế GC cho remote objects trong RMI — đảm bảo server object bị GC khi không còn client nào reference đến nó.

**Vấn đề:**
```
Client A ──→ Remote Object trên Server
Client B ──→ Remote Object trên Server
Client A ngắt kết nối

Khi nào Remote Object có thể bị GC?
→ Chỉ khi TẤT CẢ clients không còn reference!
```

**DGC hoạt động:**

```java
// DGC dùng reference counting qua leasing:

// 1. Khi client nhận stub → "dirty call" tới server
//    Server tăng reference count

// 2. Client định kỳ gửi "lease renewal" (mặc định 10 phút)

// 3. Khi client gọi GC local → "clean call" tới server
//    Server giảm reference count

// 4. Khi reference count = 0 → Remote object eligible for GC

// Lease duration (mặc định 600,000ms = 10 phút)
System.setProperty("java.rmi.dgc.leaseValue", "600000");
```

**DGC Interface:**
```java
// java.rmi.dgc.DGC
public interface DGC extends Remote {
    Lease dirty(ObjID[] ids, long sequenceNum, Lease lease) throws RemoteException;
    void clean(ObjID[] ids, long sequenceNum, VMID vmid, boolean strong) throws RemoteException;
}
```

**Tóm tắt:** DGC = distributed reference counting với lease-based protocol để phát hiện dead clients.

---

## Q129. What is the difference between HashSet and TreeSet?

| | HashSet | TreeSet |
|--|---------|---------|
| Thứ tự | **Không có** | **Sorted** (natural hoặc Comparator) |
| Performance | O(1) add/remove/contains | O(log n) |
| Null | Cho phép 1 null | **Không** cho phép null |
| Backed by | HashMap | TreeMap |
| Implements | Set | NavigableSet, SortedSet |
| Comparator | Không | Có thể custom |

```java
// HashSet — không có thứ tự, nhanh
Set<String> hs = new HashSet<>();
hs.add("banana"); hs.add("apple"); hs.add("cherry");
System.out.println(hs); // [cherry, apple, banana] hoặc khác

// TreeSet — sắp theo natural order
Set<String> ts = new TreeSet<>();
ts.add("banana"); ts.add("apple"); ts.add("cherry");
System.out.println(ts); // [apple, banana, cherry] luôn sorted!

// TreeSet — custom Comparator
TreeSet<String> byLength = new TreeSet<>(Comparator.comparingInt(String::length));
byLength.add("banana"); byLength.add("fig"); byLength.add("apple");
System.out.println(byLength); // [fig, apple, banana] hoặc [fig, apple/banana, ...]

// TreeSet navigation methods
TreeSet<Integer> nums = new TreeSet<>(Set.of(1,3,5,7,9));
System.out.println(nums.floor(6));    // 5 (≤ 6)
System.out.println(nums.ceiling(6));  // 7 (≥ 6)
System.out.println(nums.headSet(5));  // [1, 3] (< 5)
System.out.println(nums.tailSet(5));  // [5, 7, 9] (≥ 5)
System.out.println(nums.subSet(3,7)); // [3, 5] (3 ≤ x < 7)
```

---

## Q130. What are the problems with Double Brace Initialization?

```java
// Double Brace Initialization (DBI)
List<String> list = new ArrayList<String>() {{
    add("A");
    add("B");
}};
```

**Vấn đề 1: Anonymous class creation**
```java
// DBI tạo anonymous subclass của ArrayList
// → Tốn memory hơn
// → Thêm .class file vào JAR
// → Class loader phải load thêm class

// Bytecode tương đương:
class MyClass$1 extends ArrayList<String> {
    {
        add("A");
        add("B");
    }
}
List<String> list = new MyClass$1();
```

**Vấn đề 2: Memory Leak (nghiêm trọng nhất)**
```java
public class MyClass {
    private String data = "important data";
    
    public Map<String, String> createMap() {
        // Anonymous class giữ implicit reference đến outer MyClass!
        return new HashMap<String, String>() {{
            put("key", "value");
        }};
        // Map này giữ reference đến MyClass instance
        // Nếu Map tồn tại lâu → MyClass không được GC!
    }
}
```

**Vấn đề 3: Serialization failure**
```java
List<String> list = new ArrayList<String>() {{ add("A"); }};
// Anonymous class → serialVersionUID khác nhau trên các JVM
// Có thể fail serialization
```

**Vấn đề 4: equals() failure**
```java
List<String> l1 = new ArrayList<String>() {{ add("A"); }};
List<String> l2 = new ArrayList<String>() {{ add("A"); }};
System.out.println(l1.getClass() == l2.getClass()); // false! Hai anonymous classes khác nhau
```

**Solutions:**
```java
// Java 9+ — tốt nhất
List<String> list = new ArrayList<>(List.of("A", "B"));

// Stream
List<String> list = Stream.of("A", "B").collect(Collectors.toList());
```

---

## Q131. How does monitor-based synchronization work?

**Monitor** là cơ chế đồng bộ built-in của mỗi Java object, gồm 2 phần:
1. **Mutex (lock):** Đảm bảo mutual exclusion
2. **Wait Set:** Nơi threads đợi điều kiện (condition variable)

```java
// Monitor operations:
// ENTER: acquire lock khi vào synchronized block
// EXIT: release lock khi ra (kể cả exception)
// WAIT: nhả lock, vào wait set
// NOTIFY: đánh thức một thread từ wait set
// NOTIFYALL: đánh thức tất cả threads từ wait set

public class BoundedBuffer<T> {
    private final Queue<T> buffer = new LinkedList<>();
    private final int maxSize;
    
    public BoundedBuffer(int maxSize) { this.maxSize = maxSize; }
    
    public synchronized void put(T item) throws InterruptedException {
        while (buffer.size() == maxSize) {
            wait(); // nhả lock, chờ consumer lấy bớt
        }
        buffer.add(item);
        notifyAll(); // báo cho consumer biết có item mới
    }
    
    public synchronized T take() throws InterruptedException {
        while (buffer.isEmpty()) {
            wait(); // nhả lock, chờ producer thêm vào
        }
        T item = buffer.remove();
        notifyAll(); // báo cho producer biết có chỗ trống
        return item;
    }
}
```

**Bytecode level:**
```
monitorenter  // acquire monitor
... code ...
monitorexit   // release monitor
```

**Spurious wakeups:** Thread có thể thức dậy không lý do → luôn dùng `while` thay vì `if` với `wait()`.

---

## Q132. Why can `String.length()` be inaccurate?

**Vấn đề:** `length()` trả về số `char` (UTF-16 code units), không phải số **Unicode code points** hay **characters**.

```java
// Ký tự ASCII — length() chính xác
String s = "hello";
System.out.println(s.length()); // 5 ✓

// Ký tự UTF-16 đơn — chính xác
String emoji = "你好";
System.out.println(emoji.length()); // 2 ✓

// Supplementary characters (code points > 0xFFFF) — SAI!
// Ví dụ: Emoji 😀 (U+1F600) dùng 2 char (surrogate pair)
String emoji2 = "😀";
System.out.println(emoji2.length()); // 2 ← SAI! Chỉ 1 emoji
System.out.println(emoji2.codePointCount(0, emoji2.length())); // 1 ✓

String mixed = "Hello😀World";
System.out.println(mixed.length()); // 12 (5 + 2 + 5)
System.out.println(mixed.codePointCount(0, mixed.length())); // 11 ✓
```

**Giải thích:**
- Java `String` dùng **UTF-16** encoding
- Unicode có code points từ U+0000 đến U+10FFFF
- Từ U+10000 trở lên (Supplementary characters) cần **2 char** (surrogate pair)
- `length()` đếm char, không đếm code points

**API đúng cho Unicode:**
```java
// Số ký tự thực (code points)
int realLength = str.codePointCount(0, str.length());

// Duyệt theo code point
str.codePoints().forEach(cp -> {
    System.out.println(new String(Character.toChars(cp)));
});

// Java 11+
str.chars()         // IntStream của char values
str.codePoints()    // IntStream của code point values
```

---

## Q133. Provide examples where `finally` block won't execute.

Mặc dù `finally` **gần như luôn** chạy, có một số ngoại lệ:

**1. `System.exit()` được gọi:**
```java
try {
    System.out.println("try");
    System.exit(0); // JVM shutdown ngay lập tức!
} finally {
    System.out.println("finally"); // KHÔNG chạy!
}
```

**2. JVM crash (SIGSEGV, OutOfMemoryError không recoverable):**
```java
try {
    throw new OutOfMemoryError(); // có thể không recover được
} finally {
    // Có thể không chạy nếu JVM crash hoàn toàn
}
```

**3. Infinite loop hoặc deadlock trong try:**
```java
try {
    while (true) {} // infinite loop
} finally {
    // KHÔNG bao giờ chạy
}
```

**4. Thread bị kill (Thread.stop() — deprecated):**
```java
// Thread.stop() throws ThreadDeath — finally chạy!
// Nhưng native kill signals có thể bỏ qua
```

**5. `Runtime.halt()` (ít biết):**
```java
try {
    Runtime.getRuntime().halt(0); // forceful shutdown, không chạy shutdown hooks
} finally {
    // KHÔNG chạy!
}
```

**Thực tế:** Trong 99.9% trường hợp, `finally` đảm bảo chạy. Chỉ `System.exit()` và JVM crash là exception phổ biến.

---

## Q134. What is the difference between `volatile` and `static`?

| | `volatile` | `static` |
|--|-----------|---------|
| Mục đích | **Visibility** across threads | **Shared** across instances |
| Memory | Main memory (không cache) | Method area (class-level) |
| Scope | Per-instance, no caching | Per-class, shared |
| Concurrency | Visibility guarantee | Không có concurrency guarantee |
| Atomicity | Không | Không |

```java
public class Example {
    static int staticVar = 0;      // một biến, chia sẻ giữa instances
    volatile int volatileVar = 0;  // mỗi instance có, không cache CPU
    static volatile int both = 0;  // class-level + no caching (phổ biến!)
    
    // Static — shared state, nhưng KHÔNG thread-safe
    static void badIncrement() {
        staticVar++; // race condition!
    }
    
    // Volatile — visibility, nhưng vẫn KHÔNG thread-safe cho compound operations
    void stillBad() {
        volatileVar++; // read-modify-write vẫn không atomic!
    }
    
    // Thread-safe: AtomicInteger
    static AtomicInteger counter = new AtomicInteger(0);
    static void goodIncrement() {
        counter.incrementAndGet(); // atomic!
    }
}
```

**static volatile cùng nhau:**
```java
// Phổ biến cho Singleton
public class Singleton {
    private static volatile Singleton instance;
    
    public static Singleton getInstance() {
        if (instance == null) {
            synchronized(Singleton.class) {
                if (instance == null) {
                    instance = new Singleton(); // visible to all threads
                }
            }
        }
        return instance;
    }
}
```

---

## Q135. When to prefer ArrayList vs LinkedList?

**Prefer ArrayList trong hầu hết trường hợp:**

```java
// 1. Random access — O(1) vs O(n)
List<Integer> al = new ArrayList<>(million_elements);
al.get(500_000); // O(1) — instant

List<Integer> ll = new LinkedList<>(million_elements);
ll.get(500_000); // O(n) — must traverse 500,000 nodes!

// 2. Iteration — ArrayList nhanh hơn vì cache-friendly
for (Integer i : arrayList) { } // cache-friendly, data contiguous
for (Integer i : linkedList) { } // cache-miss nhiều, nodes scattered in memory

// 3. Memory — LinkedList tốn gấp đôi (prev, next pointers)
// ArrayList: [data, data, data, data] — compact
// LinkedList: [prev|data|next] → [prev|data|next] → ... — overhead mỗi node
```

**Prefer LinkedList khi:**
```java
// Thêm/xóa đầu list thường xuyên — O(1) vs O(n)
LinkedList<Task> queue = new LinkedList<>();
queue.addFirst(urgentTask); // O(1)
queue.removeFirst();        // O(1)

// Dùng như Deque/Queue
Deque<String> deque = new LinkedList<>();
deque.offerFirst("front");
deque.offerLast("back");
deque.pollFirst();
deque.pollLast();
```

**Benchmark thực tế:**
```
Operation       | ArrayList  | LinkedList
----------------|-----------|------------
get(index)      | 0.001 ms  | 0.3 ms (N/2 hops)
add(end)        | 0.002 ms  | 0.003 ms
add(beginning)  | 0.5 ms    | 0.002 ms
contains()      | O(n)      | O(n) (same)
```

**Quy tắc ngón tay cái:** Dùng `ArrayList`. Chỉ switch sang `LinkedList` khi profile cho thấy head insertion là bottleneck.

---

## Q136. What is the difference between SoftReference and WeakReference?

| | SoftReference | WeakReference |
|--|--------------|---------------|
| GC khi nào | **Low memory** | **Bất kỳ lúc nào** (next GC cycle) |
| Tồn tại | Đến khi OOM gần xảy ra | Chỉ đến next GC |
| Dùng cho | **Memory-sensitive cache** | **Canonicalize mappings**, listeners |
| Thứ tự | Trước WeakRef | Sau SoftRef |

```java
// SoftReference — survive miễn còn memory
SoftReference<byte[]> softCache = new SoftReference<>(new byte[10 * 1024 * 1024]);
// GC sẽ collect khi JVM sắp OOM

byte[] data = softCache.get();
if (data == null) {
    data = loadFromDisk(); // cache miss, reload
    softCache = new SoftReference<>(data);
}

// WeakReference — GC gần như ngay khi không có strong ref
WeakReference<ExpensiveObject> weakRef = new WeakReference<>(obj);
obj = null; // remove strong reference

System.gc();
System.out.println(weakRef.get()); // null — đã bị GC!

// WeakHashMap — keys là WeakReference
Map<Widget, WidgetInfo> map = new WeakHashMap<>();
// Key (Widget) sẽ bị GC khi không còn strong reference
// → Entry tự động bị remove khỏi map!
```

**ReferenceQueue:**
```java
ReferenceQueue<Object> queue = new ReferenceQueue<>();
WeakReference<Object> ref = new WeakReference<>(new Object(), queue);

// Sau khi GC, ref được enqueue vào queue
System.gc();
Reference<?> cleared = queue.poll();
if (cleared != null) {
    System.out.println("Object was GC'd");
}
```

**Hierarchy:**
```
Strong Reference → không bị GC
SoftReference   → GC khi low memory (cache)
WeakReference   → GC sớm nhất (canonical maps)
PhantomReference → GC sau finalize() (cleanup actions)
```

---

## Q137. How to implement a Singleton pattern correctly?

**Singleton** đảm bảo chỉ có một instance của class trong toàn bộ JVM.

**Cách 1: Eager Initialization (đơn giản nhất)**
```java
public class Singleton {
    // Tạo khi class được load — thread-safe nhờ class loading
    private static final Singleton INSTANCE = new Singleton();
    
    private Singleton() {} // ngăn tạo từ bên ngoài
    
    public static Singleton getInstance() {
        return INSTANCE;
    }
}
// Nhược điểm: tạo dù không dùng
```

**Cách 2: Lazy + Double-Checked Locking (thread-safe, lazy)**
```java
public class Singleton {
    private static volatile Singleton instance; // volatile bắt buộc!
    
    private Singleton() {}
    
    public static Singleton getInstance() {
        if (instance == null) {                    // check 1 — no lock
            synchronized(Singleton.class) {
                if (instance == null) {            // check 2 — với lock
                    instance = new Singleton();    // volatile đảm bảo visibility
                }
            }
        }
        return instance;
    }
}
// Tại sao cần volatile?
// new Singleton() gồm 3 bước:
// 1. Cấp phát memory
// 2. Khởi tạo object
// 3. Gán reference
// Không có volatile → CPU có thể reorder 2 và 3
// → Thread khác thấy non-null reference nhưng object chưa khởi tạo!
```

**Cách 3: Initialization-on-demand Holder (khuyến nghị)**
```java
public class Singleton {
    private Singleton() {}
    
    // Inner class không load cho đến khi được reference
    private static class Holder {
        static final Singleton INSTANCE = new Singleton();
    }
    
    public static Singleton getInstance() {
        return Holder.INSTANCE; // thread-safe nhờ class loading
    }
}
// Lazy + thread-safe + không cần volatile + không cần synchronized!
// Class loading là thread-safe theo JVM specification
```

**Cách 4: Enum (tốt nhất — Joshua Bloch)**
```java
public enum Singleton {
    INSTANCE;
    
    private int value; // có thể có state
    
    public void doSomething() {
        System.out.println("Singleton method");
    }
    
    public int getValue() { return value; }
    public void setValue(int value) { this.value = value; }
}

// Sử dụng
Singleton.INSTANCE.doSomething();
Singleton.INSTANCE.setValue(42);

// Tại sao Enum tốt nhất:
// 1. Thread-safe (class loading guarantee)
// 2. Serialization-safe (Enum không tạo instance mới khi deserialize)
// 3. Reflection-safe (không thể tạo instance qua reflection)
// 4. Code ngắn gọn
```

**Vấn đề của các cách khác:**
```java
// Reflection attack (vượt qua Singleton)
Constructor<Singleton> c = Singleton.class.getDeclaredConstructor();
c.setAccessible(true);
Singleton s2 = c.newInstance(); // TẠO INSTANCE MỚI!

// Giải pháp: throw exception trong constructor
private Singleton() {
    if (INSTANCE != null) {
        throw new RuntimeException("Use getInstance()!");
    }
}

// Serialization attack
// Deserialize tạo instance mới → phá vỡ Singleton
// Giải pháp: implement readResolve()
protected Object readResolve() {
    return INSTANCE; // return singleton thay vì deserialized object
}
```

**So sánh các cách:**

| Cách | Lazy | Thread-safe | Serialization-safe | Reflection-safe |
|------|------|------------|-------------------|-----------------|
| Eager | Không | Có | Cần readResolve | Cần guard |
| DCL + volatile | Có | Có | Cần readResolve | Cần guard |
| Holder | Có | Có | Cần readResolve | Cần guard |
| **Enum** | Không | **Có** | **Có** | **Có** |

**Khuyến nghị:** Dùng **Enum** nếu không cần lazy init. Dùng **Holder** nếu cần lazy init.

---

## Tổng kết Expert Level

| Topic | Key Insight |
|-------|-------------|
| Perm Gen GC | Xảy ra trong Full GC; Java 8+ dùng Metaspace |
| synchronized | Monitor lock: mutex + wait set; reentrant |
| RMI Architecture | App → Stub/Skeleton → Transport (JRMP) |
| DGC | Distributed ref counting với leasing protocol |
| HashSet vs TreeSet | O(1) unordered vs O(log n) sorted |
| Double Brace Init | Anonymous class → memory leak, serialization issue |
| Monitor | Mutex + condition variable built into every object |
| String.length() | Đếm char (UTF-16), không phải Unicode code points |
| finally exception | System.exit(), JVM crash, infinite loop |
| volatile vs static | Visibility vs shared state |
| ArrayList vs LinkedList | ArrayList cho hầu hết; LinkedList cho head ops |
| Soft vs Weak Ref | Cache (low mem) vs canonicalize (next GC) |
| Singleton | Enum (tốt nhất) hoặc Holder (lazy) |
