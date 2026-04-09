# Java Interview Questions — Senior (Q95–Q124)

---

## Q95. What are the best practices for using Collections?

**1. Lập trình theo interface, không theo implementation:**
```java
// Tốt
List<String> list = new ArrayList<>();
Map<String, Integer> map = new HashMap<>();

// Xấu — khó thay đổi implementation
ArrayList<String> list = new ArrayList<>();
```

**2. Chọn đúng Collection:**
```java
// Cần unique + unsorted → HashSet
Set<String> unique = new HashSet<>();

// Cần unique + sorted → TreeSet
Set<String> sorted = new TreeSet<>();

// Cần key-value + fast lookup → HashMap
Map<String, User> cache = new HashMap<>();

// Cần thread-safe → concurrent collections
Map<String, User> concurrent = new ConcurrentHashMap<>();
List<String> safeList = new CopyOnWriteArrayList<>();
```

**3. Khởi tạo với initial capacity:**
```java
// Tránh resize nhiều lần
List<String> list = new ArrayList<>(1000);
Map<String, String> map = new HashMap<>(256);
```

**4. Dùng Collections utility:**
```java
List<String> sorted = new ArrayList<>(list);
Collections.sort(sorted);
Collections.shuffle(list);
int max = Collections.max(list, Comparator.comparingInt(String::length));
```

**5. Immutable Collections (Java 9+):**
```java
List<String> immutable = List.of("A", "B", "C");
Map<String, Integer> immMap = Map.of("key1", 1, "key2", 2);
Set<String> immSet = Set.of("X", "Y", "Z");
```

---

## Q96. Why is `char[]` preferred over `String` for passwords?

**String là immutable và ở trong String Pool:**
```java
String password = "mySecret123";
// "mySecret123" tồn tại trong String Pool
// Nếu heap dump → plaintext password trong memory!
// GC không đảm bảo clear ngay
```

**char[] có thể xóa ngay sau khi dùng:**
```java
char[] password = {'m','y','S','e','c','r','e','t'};
// Dùng xong:
Arrays.fill(password, '\0'); // xóa ngay khỏi memory!
// Heap dump → không còn password

// Thực tế trong Java security APIs:
KeyStore.PasswordProtection protection = 
    new KeyStore.PasswordProtection(password);
Arrays.fill(password, '\0'); // xóa sau khi dùng
```

**Lý do:**
1. `String` ở trong pool → sống lâu hơn cần thiết
2. `String` không thể xóa nội dung
3. `char[]` có thể `Arrays.fill('\0')` để xóa
4. Nếu memory dump → `char[]` đã xóa, `String` vẫn còn

---

## Q97. What are the differences between HashMap and Hashtable?

| | HashMap | Hashtable |
|--|---------|-----------|
| Thread-safe | **Không** | **Có** (synchronized) |
| Null key | **1 null key** | **Không** cho phép |
| Null value | Cho phép | Không cho phép |
| Performance | **Nhanh hơn** | Chậm hơn |
| Iterator | Fail-fast | Fail-safe (Enumeration) |
| Kế thừa | AbstractMap | Dictionary (legacy) |
| Introduced | Java 1.2 | Java 1.0 |
| Legacy | Không | **Có** |

```java
// HashMap
Map<String, String> hm = new HashMap<>();
hm.put(null, "value");   // OK
hm.put("key", null);     // OK

// Hashtable
Hashtable<String, String> ht = new Hashtable<>();
// ht.put(null, "value"); // NullPointerException!
// ht.put("key", null);   // NullPointerException!

// Thread-safe alternatives to Hashtable:
Map<String, String> safe = new ConcurrentHashMap<>(); // tốt hơn Hashtable
Map<String, String> wrapped = Collections.synchronizedMap(new HashMap<>());
```

---

## Q98. What is a Marker Interface?

**Marker Interface** (hay Tagging Interface) là interface **không có method** — chỉ "đánh dấu" class để JVM hoặc framework xử lý đặc biệt.

```java
// Marker interfaces trong Java standard library
public interface Serializable {}      // cho phép serialize
public interface Cloneable {}         // cho phép clone()
public interface RandomAccess {}      // đánh dấu List hỗ trợ random access nhanh
public interface Remote {}            // dùng trong RMI

// Custom marker interface
public interface Auditable {}

public class Order implements Auditable {
    // ...
}

// Framework check marker
if (obj instanceof Auditable) {
    auditService.log(obj);
}
```

**Tại sao dùng Marker Interface thay vì Annotation?**
```java
// Annotation (Java 5+) thường tốt hơn:
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Auditable {
    String level() default "INFO"; // có thể có attributes!
}

// Annotation linh hoạt hơn: có attributes
// Marker interface: dùng instanceof check, không cần Reflection
```

---

## Q99. When to use LinkedList vs ArrayList?

**Dùng ArrayList khi:**
- Cần **random access** (`get(index)`) thường xuyên — O(1)
- Thêm/xóa ở **cuối** list — O(1) amortized
- Ít thêm/xóa ở giữa/đầu — O(n) shift
- Cần **memory efficiency** — không có overhead per-node

**Dùng LinkedList khi:**
- Thêm/xóa ở **đầu** list thường xuyên — O(1)
- Dùng như **Queue/Deque** — `addFirst`, `removeFirst`
- Không cần random access

```java
// ArrayList — tốt cho search và access by index
List<String> al = new ArrayList<>();
String s = al.get(500); // O(1)

// LinkedList — tốt cho insertion/deletion ở đầu
LinkedList<String> ll = new LinkedList<>();
ll.addFirst("head");   // O(1)
ll.removeFirst();      // O(1)
ll.addLast("tail");    // O(1)

// LinkedList như Queue
Queue<Task> queue = new LinkedList<>();
queue.offer(task);       // enqueue
Task t = queue.poll();   // dequeue

// LinkedList như Stack (Deque)
Deque<String> stack = new LinkedList<>();
stack.push("item");
String top = stack.pop();
```

**Thực tế:** 90% trường hợp, `ArrayList` tốt hơn. Chỉ dùng `LinkedList` khi có use case rõ ràng.

---

## Q100. What is Spring MVC and how does it differ from pure Servlets?

**Pure Servlet (Controller):**
```java
@WebServlet("/users/*")
public class UserServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) {
        String path = req.getPathInfo(); // /123
        // Parse path, gọi service, tạo HTML/JSON response...
        // Phải tự làm tất cả: routing, parsing, response format
    }
}
```

**Spring MVC:**
```java
@RestController
@RequestMapping("/users")
public class UserController {
    @Autowired
    private UserService userService;
    
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        return ResponseEntity.ok(user); // Spring tự convert sang JSON
    }
    
    @PostMapping
    public ResponseEntity<User> createUser(@RequestBody @Valid UserDto dto) {
        return ResponseEntity.status(201).body(userService.create(dto));
    }
}
```

| | Pure Servlet | Spring MVC |
|--|-------------|------------|
| Routing | Thủ công | `@RequestMapping`, `@GetMapping` |
| JSON | Thủ công (Gson/Jackson) | Tự động |
| DI | Thủ công | `@Autowired` |
| Validation | Thủ công | `@Valid`, `@NotNull` |
| Exception handling | Thủ công | `@ExceptionHandler` |
| Testing | Khó | Dễ với MockMvc |

---

## Q101. What is the Builder Pattern?

**Builder Pattern** dùng khi object có nhiều fields, tránh constructor dài và không rõ nghĩa.

```java
// Vấn đề — Telescoping Constructor
public Person(String name, int age, String email, String phone, String address) {}
// Gọi: new Person("Alice", 30, "a@b.com", null, null) — khó đọc!

// Giải pháp — Builder Pattern
public class Person {
    private final String name;   // required
    private final int age;       // required
    private final String email;  // optional
    private final String phone;  // optional
    private final String address;// optional
    
    private Person(Builder builder) {
        this.name = builder.name;
        this.age = builder.age;
        this.email = builder.email;
        this.phone = builder.phone;
        this.address = builder.address;
    }
    
    public static class Builder {
        private final String name;
        private final int age;
        private String email = "";
        private String phone = "";
        private String address = "";
        
        public Builder(String name, int age) {
            this.name = name;
            this.age = age;
        }
        
        public Builder email(String email) { this.email = email; return this; }
        public Builder phone(String phone) { this.phone = phone; return this; }
        public Builder address(String address) { this.address = address; return this; }
        
        public Person build() {
            return new Person(this);
        }
    }
}

// Sử dụng — rõ ràng, dễ đọc
Person p = new Person.Builder("Alice", 30)
    .email("alice@example.com")
    .phone("0901234567")
    .build();
```

**Lombok @Builder:**
```java
@Builder
public class Person {
    private String name;
    private int age;
    private String email;
}
// Tự động generate Builder
Person p = Person.builder().name("Alice").age(30).build();
```

---

## Q102. What is the difference between Internet Applets and Filesystem Applets?

| | Internet Applet | Filesystem Applet |
|--|-----------------|-------------------|
| Load từ | Remote server (HTTP) | Local filesystem |
| Sandbox | Strict (sandboxed) | Relaxed |
| File access | Không được | Có thể được |
| Network | Chỉ về origin | Toàn quyền |
| Trust | Untrusted | Trusted |
| Security | SecurityManager strict | Ít hạn chế hơn |

**Internet Applet:**
```html
<applet code="MyApp.class" codebase="http://example.com/applets/">
```
→ Chạy với đầy đủ security restrictions

**Filesystem Applet:**
```bash
appletviewer file:///path/to/MyApp.html
```
→ Chạy với ít restriction hơn (vẫn có sandbox nhưng lỏng hơn)

---

## Q103. How to prevent deadlock with N threads and N resources?

**Bài toán:** N threads, mỗi thread cần 2 resources → dễ deadlock.

**Giải pháp 1: Thứ tự tài nguyên nhất quán (Resource Ordering)**
```java
// Thay vì mỗi thread lock theo thứ tự riêng,
// tất cả lock theo thứ tự tăng dần ID
public void transfer(Account from, Account to, double amount) {
    Account first = from.getId() < to.getId() ? from : to;
    Account second = from.getId() < to.getId() ? to : from;
    
    synchronized(first) {
        synchronized(second) {
            from.debit(amount);
            to.credit(amount);
        }
    }
}
```

**Giải pháp 2: tryLock với timeout**
```java
public boolean transfer(Account from, Account to, double amount) 
        throws InterruptedException {
    while (true) {
        if (from.lock.tryLock(100, TimeUnit.MILLISECONDS)) {
            try {
                if (to.lock.tryLock(100, TimeUnit.MILLISECONDS)) {
                    try {
                        from.debit(amount);
                        to.credit(amount);
                        return true;
                    } finally {
                        to.lock.unlock();
                    }
                }
            } finally {
                from.lock.unlock();
            }
        }
        Thread.sleep(10); // back off trước khi thử lại
    }
}
```

**Giải pháp 3: Single lock ordering (Dijkstra's Banker's Algorithm)**
- Cấp phát tài nguyên chỉ khi không gây deadlock potential

---

## Q104. Does Java have default parameter values?

**Java KHÔNG có default parameter values** như Python hay C++.

**Workarounds:**

**1. Method Overloading:**
```java
public void connect(String host) {
    connect(host, 8080); // default port
}
public void connect(String host, int port) {
    connect(host, port, 30); // default timeout
}
public void connect(String host, int port, int timeout) {
    // actual implementation
}
```

**2. Builder Pattern:**
```java
ConnConfig config = ConnConfig.builder()
    .host("localhost")
    // port và timeout dùng default
    .build();
```

**3. Optional:**
```java
public void connect(String host, Optional<Integer> port) {
    int p = port.orElse(8080);
}
connect("localhost", Optional.empty()); // dùng default
```

**4. Varargs:**
```java
public void log(String message, Object... args) {
    // args rỗng nếu không truyền
}
```

---

## Q105. What is RMISecurityManager?

`RMISecurityManager` là SecurityManager đặc biệt cho RMI applications, cho phép tải class từ remote locations (stub classes, implementation classes).

```java
// Cài đặt RMISecurityManager
if (System.getSecurityManager() == null) {
    System.setSecurityManager(new RMISecurityManager());
    // hoặc từ Java 2+:
    System.setSecurityManager(new SecurityManager());
}
```

**Cần thiết khi:**
- Client cần download class định nghĩa từ server
- Dynamic class loading qua RMI
- Client không có sẵn stub class

**Cần security policy file:**
```
// java.policy
grant {
    permission java.net.SocketPermission "*:1024-65535", "connect,accept";
    permission java.net.SocketPermission "*:80", "connect";
};
```

**Lưu ý:** `RMISecurityManager` đã deprecated từ Java 17 do SecurityManager deprecated.

---

## Q106. What is `java.rmi.Naming`?

`java.rmi.Naming` cung cấp các method để bind/lookup remote objects với RMI Registry sử dụng **URL-style names**.

```java
// Bind (server side)
Calculator calc = new CalculatorImpl();
Naming.rebind("rmi://localhost:1099/Calculator", calc);
// URL format: rmi://host:port/name

// Lookup (client side)
Calculator calc = (Calculator) Naming.lookup("rmi://localhost:1099/Calculator");
int result = calc.add(3, 4);

// List bound names
String[] names = Naming.list("rmi://localhost:1099/");
for (String name : names) {
    System.out.println(name);
}

// Unbind
Naming.unbind("rmi://localhost:1099/Calculator");
```

**So sánh với Registry:**
```java
// Registry API — lower level
Registry registry = LocateRegistry.getRegistry("localhost", 1099);
registry.bind("Calculator", calc);
Calculator c = (Calculator) registry.lookup("Calculator");

// Naming API — convenience wrapper
Naming.bind("rmi://localhost:1099/Calculator", calc);
Calculator c = (Calculator) Naming.lookup("rmi://localhost/Calculator");
```

---

## Q107. What is the difference between Inner Class and Static Nested Class?

| | Inner Class | Static Nested Class |
|--|-------------|---------------------|
| Keyword | Không có `static` | `static` |
| Outer instance | **Cần** | **Không cần** |
| Truy cập outer | Có thể truy cập tất cả | Chỉ static members |
| Static members | Không có | Có thể có |
| Tạo instance | `new Outer().new Inner()` | `new Outer.Nested()` |
| Dùng cho | Iterator, callback | Builder, helper |

```java
public class Outer {
    private int x = 10;
    private static int y = 20;
    
    // Inner class — có implicit reference đến Outer instance
    public class Inner {
        void show() {
            System.out.println(x); // OK — truy cập outer instance field
            System.out.println(y); // OK — static field
        }
    }
    
    // Static nested class — không có Outer instance reference
    public static class Nested {
        void show() {
            // System.out.println(x); // Lỗi! Không có Outer instance
            System.out.println(y); // OK — static field
        }
    }
}

// Tạo instance
Outer.Inner inner = new Outer().new Inner();
Outer.Nested nested = new Outer.Nested(); // không cần Outer instance
```

**Memory concern:** Inner class giữ reference đến Outer instance → có thể gây memory leak nếu Inner sống lâu hơn Outer.

---

## Q108. What is the difference between Protocol and Interface?

| | Protocol | Interface |
|--|----------|-----------|
| Ngữ cảnh | Network/Communication | OOP/Java |
| Định nghĩa | Quy tắc giao tiếp giữa hệ thống | Contract cho Java class |
| Ví dụ | HTTP, FTP, TCP/IP | Runnable, Serializable, List |
| Thực thi | Bởi client-server systems | Bởi Java classes |
| Language | Ngôn ngữ agnostic | Java-specific |

**Protocol trong Java (RMI context):**
- RMI sử dụng **JRMP** (Java Remote Method Protocol) hoặc IIOP
- Protocol xác định format của data được serialize và gửi qua network

**Interface trong Java:**
```java
// Java Interface — contract
public interface Comparable<T> {
    int compareTo(T o);
}

// Class thực thi contract
public class Integer implements Comparable<Integer> {
    @Override
    public int compareTo(Integer o) {
        return Integer.compare(this.value, o.value);
    }
}
```

---

## Q109. What is the Boyer-Moore algorithm?

**Boyer-Moore** là thuật toán tìm kiếm chuỗi nhanh, tốt hơn brute force.

**Hai heuristics chính:**
1. **Bad Character Rule:** Khi mismatch, skip dựa vào ký tự không khớp
2. **Good Suffix Rule:** Khi mismatch, skip dựa vào suffix đã khớp

```java
// Ví dụ tìm "PATTERN" trong "TEXT"
// Boyer-Moore duyệt từ phải sang trái trong pattern

public static int boyerMoore(String text, String pattern) {
    int n = text.length();
    int m = pattern.length();
    
    // Bad character table
    int[] badChar = new int[256];
    Arrays.fill(badChar, -1);
    for (int i = 0; i < m; i++) {
        badChar[pattern.charAt(i)] = i;
    }
    
    int s = 0; // shift từ text
    while (s <= n - m) {
        int j = m - 1;
        
        while (j >= 0 && pattern.charAt(j) == text.charAt(s + j)) {
            j--;
        }
        
        if (j < 0) {
            return s; // tìm thấy!
        }
        
        // Shift dựa vào bad character rule
        s += Math.max(1, j - badChar[text.charAt(s + j)]);
    }
    
    return -1; // không tìm thấy
}
```

**Hiệu năng:**
- Tốt nhất: O(n/m) — skip nhiều ký tự
- Tệ nhất: O(n*m) — ít gặp
- Thực tế: Rất nhanh với text tự nhiên

---

## Q110. What is Servlet Chaining?

**Servlet Chaining** là kỹ thuật nhiều Servlet xử lý một request theo chuỗi, output của Servlet này là input của Servlet tiếp theo.

```java
// Servlet 1 — Authentication
public class AuthServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) {
        if (isAuthenticated(req)) {
            // Forward đến Servlet tiếp theo
            RequestDispatcher rd = req.getRequestDispatcher("/data");
            rd.forward(req, resp);
        } else {
            resp.sendRedirect("/login");
        }
    }
}

// Servlet 2 — Data Processing
public class DataServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) {
        // Xử lý data và forward đến View
        req.setAttribute("data", processData());
        RequestDispatcher rd = req.getRequestDispatcher("/WEB-INF/view.jsp");
        rd.forward(req, resp);
    }
}
```

**Servlet Filter (modern approach):**
```java
@WebFilter("/*")
public class AuthFilter implements Filter {
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) {
        if (isAuthenticated((HttpServletRequest) req)) {
            chain.doFilter(req, res); // tiếp tục chuỗi
        } else {
            ((HttpServletResponse) res).sendRedirect("/login");
        }
    }
}
```

---

## Q111. What is the basic principle of RMI?

**RMI (Remote Method Invocation)** cho phép Java object gọi method của object trên JVM khác (thậm chí trên máy khác) như thể gọi local method.

**Nguyên tắc hoạt động:**

```
Client JVM                          Server JVM
─────────────────────────────────────────────────
Client code                         Remote Object
    │                                     │
    ↓ gọi method()                        │
Stub (proxy)                         Skeleton
    │                                     │
    ├─── marshal args ──────────────────→ │
    │    (serialize)                  unmarshal args
    │                              invoke actual method
    │                              marshal return value
    ←─── unmarshal result ──────────────── │
    │    (deserialize)
    ↓
Client receives result
```

**3 thành phần:**
1. **Stub:** Client-side proxy, marshal/unmarshal
2. **Skeleton:** Server-side proxy, nhận request, invoke method
3. **RMI Registry:** Naming service, bind/lookup remote objects

---

## Q112. What is Connection Pooling?

**Connection Pooling** là kỹ thuật tái sử dụng database connections thay vì tạo mới mỗi request.

**Vấn đề không có pool:**
```java
// Mỗi request → tạo connection mới → expensive!
Connection conn = DriverManager.getConnection(url, user, pass);
// ... xử lý
conn.close(); // disconnect
// 50ms ~ 200ms cho mỗi connection creation!
```

**Với Connection Pool:**
```java
// HikariCP (pool phổ biến nhất)
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:mysql://localhost/mydb");
config.setUsername("user");
config.setPassword("pass");
config.setMaximumPoolSize(10);    // tối đa 10 connections
config.setMinimumIdle(2);         // giữ ít nhất 2 connections
config.setConnectionTimeout(30000); // timeout 30s

HikariDataSource ds = new HikariDataSource(config);

// Lấy connection từ pool
try (Connection conn = ds.getConnection()) {
    // Sử dụng connection
    PreparedStatement ps = conn.prepareStatement("SELECT ...");
    // ...
} // Connection trả về pool, không phải close thật!
```

**Lợi ích:**
- Giảm latency (không tạo connection mới)
- Giới hạn số connections đến DB
- Tự động reconnect khi connection chết
- Connection validation

---

## Q113. What is Binding in RMI?

**Binding** là đăng ký remote object với RMI Registry bằng một tên.

```java
// Server-side binding
public class RMIServer {
    public static void main(String[] args) throws Exception {
        // Tạo remote object
        Calculator calc = new CalculatorImpl();
        
        // bind() — đăng ký với tên, lỗi nếu tên đã tồn tại
        Naming.bind("Calculator", calc);
        
        // rebind() — đăng ký hoặc ghi đè nếu đã tồn tại
        Naming.rebind("Calculator", calc);
        
        // unbind() — xóa binding
        Naming.unbind("Calculator");
        
        System.out.println("Server ready");
    }
}

// Client-side lookup
public class RMIClient {
    public static void main(String[] args) throws Exception {
        // Tìm object theo tên
        Calculator calc = (Calculator) Naming.lookup("rmi://localhost/Calculator");
        System.out.println(calc.add(2, 3)); // 5
    }
}
```

**Registry API (lower level):**
```java
Registry registry = LocateRegistry.createRegistry(1099); // server
registry.bind("Calculator", calc);

Registry reg = LocateRegistry.getRegistry("localhost", 1099); // client
Calculator c = (Calculator) reg.lookup("Calculator");
```

---

## Q114. How to detect the client machine in a Servlet?

```java
protected void doGet(HttpServletRequest request, HttpServletResponse response) {
    // IP address
    String ipAddress = request.getRemoteAddr();
    // Nếu có proxy/load balancer:
    String realIP = request.getHeader("X-Forwarded-For");
    if (realIP == null) realIP = ipAddress;
    
    // Hostname
    String hostname = request.getRemoteHost();
    
    // Port
    int port = request.getRemotePort();
    
    // User-Agent (browser/OS info)
    String userAgent = request.getHeader("User-Agent");
    
    // Locale
    Locale locale = request.getLocale();
    
    // Session ID
    HttpSession session = request.getSession();
    String sessionId = session.getId();
    
    System.out.println("Client IP: " + realIP);
    System.out.println("User-Agent: " + userAgent);
    System.out.println("Locale: " + locale);
}
```

**Lưu ý:**
- `getRemoteAddr()` trả về IP của proxy nếu có proxy
- Dùng `X-Forwarded-For` header để lấy real client IP
- IPv6 có thể trả về `0:0:0:0:0:0:0:1` cho localhost

---

## Q115. What is Marshalling and Demarshalling?

**Marshalling** = chuyển đổi object thành format có thể truyền qua network (byte stream).  
**Demarshalling** = khôi phục object từ byte stream.

```
Object (Java) 
    ↓ Marshalling (Serialization)
Byte Stream (network)
    ↓ Demarshalling (Deserialization)
Object (Java) ← bên kia network
```

```java
// Trong RMI, tự động xảy ra:
// Client
int result = remoteCalc.add(3, 4); 
// Bên trong: marshal(3), marshal(4) → gửi bytes → server unmarshal → invoke → marshal result → client unmarshal

// Manual marshalling với Serialization
// Marshal
ByteArrayOutputStream baos = new ByteArrayOutputStream();
ObjectOutputStream oos = new ObjectOutputStream(baos);
oos.writeObject(myObject); // marshal
byte[] bytes = baos.toByteArray();

// Demarshal
ByteArrayInputStream bais = new ByteArrayInputStream(bytes);
ObjectInputStream ois = new ObjectInputStream(bais);
MyClass obj = (MyClass) ois.readObject(); // demarshal
```

**RMI marshalling yêu cầu:**
- Objects phải implement `Serializable`
- Cùng class definition tồn tại ở cả hai phía (hoặc dùng dynamic class loading)

---

## Q116. What is the Applet ClassLoader?

**Applet ClassLoader** tải bytecode cho Applet từ web server, khác với Application ClassLoader tải từ classpath local.

**Đặc điểm:**
1. Download .class files qua HTTP
2. Cache các class đã download
3. Security restrictions: chỉ load class từ origin server
4. Namespace isolation: mỗi codebase có ClassLoader riêng

```java
// Mỗi Applet có ClassLoader riêng
// ClassLoader hierarchy:
Bootstrap CL → Extension CL → App CL → Applet CL (network)

// Applet không thể dùng class từ codebase khác
// (trừ khi Trusted/Signed)
```

**Java Web Start (JNLP) — thay thế Applet:**
```xml
<jnlp>
  <resources>
    <jar href="myapp.jar"/>
  </resources>
  <application-desc main-class="com.example.Main"/>
</jnlp>
```

---

## Q117. What is the difference between synchronized method and synchronized block?

**Synchronized method:**
```java
public class Counter {
    private int count = 0;
    
    // Lock on 'this' (instance)
    public synchronized void increment() {
        count++;
    }
    
    // Static synchronized — lock on Class object
    public static synchronized void staticIncrement() {
        staticCount++;
    }
}
```

**Synchronized block (fine-grained):**
```java
public class Counter {
    private int count = 0;
    private final Object lock = new Object(); // dedicated lock
    
    public void increment() {
        // Chỉ lock phần cần thiết
        synchronized(lock) {
            count++;
        }
        // Code này chạy không cần lock → hiệu năng tốt hơn
        logIncrement();
    }
    
    // Hai locks độc lập
    private final Object readLock = new Object();
    private final Object writeLock = new Object();
    
    public int read() {
        synchronized(readLock) { return count; }
    }
    public void write(int val) {
        synchronized(writeLock) { count = val; }
    }
}
```

| | Synchronized method | Synchronized block |
|--|--------------------|--------------------|
| Lock object | `this` (hoặc Class) | Bất kỳ object nào |
| Granularity | Toàn bộ method | Chỉ đoạn code cần |
| Flexibility | Ít | Nhiều hơn |
| Performance | Có thể lock quá nhiều | Tốt hơn nếu dùng đúng |

---

## Q118. What is Double Brace Initialization?

**Double Brace Initialization** là kỹ thuật kết hợp anonymous class creation và instance initializer.

```java
// Cú pháp
List<String> list = new ArrayList<String>() {{
    add("A");  // outer {} = anonymous subclass của ArrayList
    add("B");  // inner {} = instance initializer block
    add("C");
}};

Map<String, Integer> map = new HashMap<String, Integer>() {{
    put("one", 1);
    put("two", 2);
    put("three", 3);
}};
```

**Vấn đề với Double Brace Initialization:**
1. Tạo anonymous subclass → memory overhead
2. Anonymous class giữ reference đến outer class → memory leak!
3. Serialization issue (anonymous class có thể không serializable)
4. Không hoạt động với `final` class

**Alternatives tốt hơn (Java 9+):**
```java
// List.of() — immutable
List<String> list = List.of("A", "B", "C");

// Map.of() — immutable
Map<String, Integer> map = Map.of("one", 1, "two", 2);

// Mutable với stream
List<String> mutable = new ArrayList<>(List.of("A", "B", "C"));
mutable.add("D"); // có thể modify
```

---

## Q119. How to test private methods in Java?

**Cách 1: Reflection (trực tiếp nhất)**
```java
public class Calculator {
    private int add(int a, int b) { return a + b; }
}

@Test
public void testPrivateAdd() throws Exception {
    Calculator calc = new Calculator();
    Method addMethod = Calculator.class.getDeclaredMethod("add", int.class, int.class);
    addMethod.setAccessible(true); // bypass private
    
    int result = (int) addMethod.invoke(calc, 3, 4);
    assertEquals(7, result);
}
```

**Cách 2: Package-private (khuyến nghị cho TDD)**
```java
// Thay private → package-private (không modifier)
int add(int a, int b) { return a + b; } // có thể test từ cùng package

// Test class ở cùng package
```

**Cách 3: Test qua public method**
```java
// Nếu private method được gọi bởi public method,
// test thông qua public method
@Test
public void testPublicMethodThatUsesPrivate() {
    assertEquals(7, calc.compute(3, 4)); // compute() gọi add() internally
}
```

**Best practice:** Nếu cần test private method thường xuyên → có thể đó là dấu hiệu cần refactor thành class riêng.

---

## Q120. What is the Permanent Generation (Perm Gen)?

**Perm Gen (Permanent Generation)** là vùng heap đặc biệt (trước Java 8) chứa JVM class metadata.

**Trong Perm Gen:**
- Class definitions (bytecode)
- Method definitions
- Constant pool
- Static variables (Java 7 trở về)
- Interned Strings (Java 7 trở về)

```
Java 7 Heap:
┌─────────────────────────────────────────┐
│ Young Gen │    Old Gen    │  Perm Gen   │
│ Eden│S0│S1│               │class meta  │
└─────────────────────────────────────────┘
```

**Java 8: Perm Gen → Metaspace**
```
Java 8+ Heap:
┌─────────────────────────────┐
│ Young Gen │    Old Gen      │   (heap, -Xmx)
│ Eden│S0│S1│                 │
└─────────────────────────────┘

Native Memory (off-heap):
┌─────────────────────────────┐
│        Metaspace             │   (-XX:MaxMetaspaceSize)
│  class metadata, method def  │
└─────────────────────────────┘
```

**Tại sao thay đổi:**
- Perm Gen có kích thước cố định → `OutOfMemoryError: PermGen space`
- Metaspace dùng native memory → tự động grow (có giới hạn tùy chọn)

---

## Q121. What is the difference between Serial and Throughput (Parallel) GC?

| | Serial GC | Throughput/Parallel GC |
|--|-----------|------------------------|
| Threads | **1 GC thread** | **Nhiều GC threads** |
| Stop-the-world | Có | Có (nhưng ngắn hơn) |
| CPU | Single CPU | Multi-CPU |
| Thích hợp | Client, nhỏ | **Server, batch processing** |
| Flag | `-XX:+UseSerialGC` | `-XX:+UseParallelGC` |
| Memory | Nhỏ | Cần nhiều hơn |

```
Serial GC:
App thread ──── PAUSE ──── App thread
                 │
           GC (1 thread)

Parallel GC:
App thread ── PAUSE ── App thread
               │
        GC (N threads, parallel)
        GC (N threads, parallel)
        GC (N threads, parallel)
```

**Các GC khác:**
- **CMS (G1 predecessor):** Concurrent Mark Sweep — giảm pause
- **G1 GC** (Java 9 default): Balanced throughput + low pause
- **ZGC** (Java 15): Ultra-low pause (< 1ms)
- **Shenandoah:** Low pause, tương tự ZGC

---

## Q122. What are the thread states in Java?

```java
Thread.State states:
1. NEW          — vừa tạo, chưa start()
2. RUNNABLE     — đang chạy hoặc sẵn sàng chạy
3. BLOCKED      — chờ monitor lock (synchronized)
4. WAITING      — chờ vô thời hạn (wait(), join())
5. TIMED_WAITING — chờ có timeout (sleep(), wait(timeout))
6. TERMINATED   — đã kết thúc
```

```java
Thread t = new Thread(() -> {
    try {
        Thread.sleep(1000);
    } catch (InterruptedException e) {}
});

System.out.println(t.getState()); // NEW

t.start();
System.out.println(t.getState()); // RUNNABLE hoặc TIMED_WAITING

t.join();
System.out.println(t.getState()); // TERMINATED
```

```
NEW ──start()──→ RUNNABLE ←─────────────────────────
                    │          ↑ unblocked/notified/timeout
                    ├──→ BLOCKED (waiting for lock)
                    ├──→ WAITING (wait()/join())
                    ├──→ TIMED_WAITING (sleep()/wait(t))
                    └──→ TERMINATED
```

---

## Q123. What is the Remote Interface in RMI?

**Remote Interface** là interface mà remote object implement, định nghĩa các method có thể gọi từ xa.

**Yêu cầu:**
1. Phải `extend java.rmi.Remote`
2. Mỗi method phải khai báo `throws RemoteException`
3. Parameters và return types phải Serializable (hoặc Remote)

```java
import java.rmi.Remote;
import java.rmi.RemoteException;

// Định nghĩa Remote Interface
public interface BankService extends Remote {
    double getBalance(String accountId) throws RemoteException;
    void transfer(String from, String to, double amount) throws RemoteException;
    List<Transaction> getHistory(String accountId) throws RemoteException;
}

// Implementation
public class BankServiceImpl extends UnicastRemoteObject implements BankService {
    
    public BankServiceImpl() throws RemoteException {
        super(); // export object để nhận remote calls
    }
    
    @Override
    public double getBalance(String accountId) throws RemoteException {
        return database.getBalance(accountId);
    }
    
    @Override
    public void transfer(String from, String to, double amount) throws RemoteException {
        database.transfer(from, to, amount);
    }
    
    @Override
    public List<Transaction> getHistory(String accountId) throws RemoteException {
        return database.getHistory(accountId);
    }
}
```

---

## Q124. What is RMI?

**RMI (Remote Method Invocation)** là Java API cho phép object trong một JVM gọi method của object trong JVM khác — qua network hoặc trong cùng máy.

**Đặc điểm:**
- **Java-specific:** Chỉ Java-Java (khác CORBA là đa ngôn ngữ)
- **Transparent:** Code gọi remote object gần giống local object
- **Serialization:** Tự động marshal/unmarshal parameters
- **Dynamic class loading:** Client có thể download class từ server

**Kiến trúc:**
```
┌───────────────────┐         ┌───────────────────┐
│   Client JVM      │         │   Server JVM       │
│                   │         │                    │
│  Client App       │         │  Remote Object     │
│      ↓            │         │      ↑             │
│   Stub (proxy)    │←network→│  Skeleton          │
│      ↓            │         │                    │
│  Transport Layer  │         │  Transport Layer   │
└───────────────────┘         └───────────────────┘
         ↕                              ↕
    RMI Registry (naming service)
```

**Ứng dụng:**
- Distributed computing
- Client-server apps
- EJB (Enterprise JavaBeans) uses RMI
- Microservices (ngày nay dùng REST/gRPC thay thế)

```java
// Đơn giản hóa: RMI = "call method trên máy khác như local"
Calculator remote = (Calculator) Naming.lookup("rmi://server/Calculator");
int result = remote.add(2, 3); // code trông như local nhưng chạy qua network
```
