# Java Interview Questions — Mid (Q33–Q94)

---

## Q33. What is the difference between fail-fast and fail-safe iterators?

**Fail-fast Iterator:**
- Phát hiện ngay khi collection bị sửa đổi trong lúc duyệt
- Ném `ConcurrentModificationException`
- Dùng `modCount` để theo dõi thay đổi
- Ví dụ: `ArrayList`, `HashMap`, `HashSet`

```java
List<String> list = new ArrayList<>(List.of("A", "B", "C"));
for (String s : list) {
    list.add("D"); // ConcurrentModificationException!
}
```

**Fail-safe Iterator:**
- Duyệt trên bản sao (snapshot) của collection
- Không ném exception khi collection bị sửa
- Có thể không phản ánh thay đổi mới nhất
- Ví dụ: `CopyOnWriteArrayList`, `ConcurrentHashMap`

```java
List<String> list = new CopyOnWriteArrayList<>(List.of("A", "B", "C"));
for (String s : list) {
    list.add("D"); // OK, không exception
}
```

| | Fail-fast | Fail-safe |
|--|-----------|-----------|
| Exception | ConcurrentModificationException | Không |
| Copy | Không copy | Copy collection |
| Performance | Nhanh hơn | Chậm hơn, tốn memory |
| Ví dụ | ArrayList, HashMap | CopyOnWriteArrayList, ConcurrentHashMap |

---

## Q34. What is the difference between `sleep()` and `wait()`?

| | `sleep()` | `wait()` |
|--|-----------|----------|
| Class | `Thread` | `Object` |
| Lock | **Giữ lock** | **Nhả lock** |
| Đánh thức | Tự thức sau timeout | Cần `notify()`/`notifyAll()` |
| Context | Bất kỳ đâu | Phải trong `synchronized` block |
| Exception | `InterruptedException` | `InterruptedException` |

```java
// sleep() — giữ lock, chờ thời gian
synchronized(lock) {
    Thread.sleep(1000); // vẫn giữ lock!
}

// wait() — nhả lock, chờ signal
synchronized(lock) {
    lock.wait(); // nhả lock, chờ notify
}
// ... thread khác:
synchronized(lock) {
    lock.notify(); // đánh thức thread đang wait
}
```

**Tóm tắt:** `sleep()` = nghỉ ngắn, giữ tài nguyên. `wait()` = chờ điều kiện, nhường tài nguyên.

---

## Q35. Can a class be declared as static?

- **Top-level class**: KHÔNG thể `static`
- **Nested class (inner class)**: CÓ thể `static` → gọi là **static nested class**

```java
public class Outer {
    // Static nested class
    public static class StaticNested {
        void show() { System.out.println("Static nested"); }
    }

    // Non-static inner class
    public class Inner {
        void show() { System.out.println("Inner"); }
    }
}

// Sử dụng
Outer.StaticNested sn = new Outer.StaticNested(); // không cần Outer instance
Outer.Inner inner = new Outer().new Inner();       // cần Outer instance
```

**Static nested class:**
- Không có tham chiếu đến Outer class instance
- Có thể có static members
- Thường dùng cho Builder pattern, helper classes

---

## Q36. What is the difference between `==` and `equals()`?

**`==` (Reference equality):**
- So sánh địa chỉ bộ nhớ (reference)
- Với primitive: so sánh giá trị

**`equals()` (Content equality):**
- So sánh nội dung (nếu được override)
- Default trong `Object`: giống `==`

```java
String s1 = new String("hello");
String s2 = new String("hello");

System.out.println(s1 == s2);       // false (khác reference)
System.out.println(s1.equals(s2));  // true  (cùng nội dung)

// String pool
String s3 = "hello";
String s4 = "hello";
System.out.println(s3 == s4);       // true (cùng pool reference)
```

**Lưu ý quan trọng:**
- Luôn dùng `equals()` để so sánh String và Objects
- Khi override `equals()`, phải override cả `hashCode()`
- `null.equals(x)` → NPE; dùng `Objects.equals(a, b)` để an toàn

---

## Q37. Is it possible to use enum with thread-safely?

**Có — Enum là thread-safe theo mặc định.**

Java đảm bảo:
1. Enum instances được khởi tạo một lần duy nhất bởi JVM
2. Class loading trong Java là thread-safe
3. Enum không thể bị tạo thêm hay sửa đổi

```java
public enum Status {
    ACTIVE, INACTIVE, PENDING;

    // Method cũng thread-safe nếu không có mutable state
    public boolean isActive() {
        return this == ACTIVE;
    }
}

// Singleton pattern dùng enum — thread-safe nhất
public enum Singleton {
    INSTANCE;

    public void doSomething() { ... }
}
```

**Lưu ý:** Nếu enum có mutable fields, các fields đó KHÔNG thread-safe:
```java
public enum Config {
    INSTANCE;
    private int count = 0; // mutable — NOT thread-safe!
    public void increment() { count++; } // race condition!
}
```

---

## Q38. What are the JSP implicit objects?

JSP cung cấp 9 implicit objects (không cần khai báo):

| Object | Type | Mô tả |
|--------|------|--------|
| `request` | `HttpServletRequest` | HTTP request |
| `response` | `HttpServletResponse` | HTTP response |
| `out` | `JspWriter` | Output stream |
| `session` | `HttpSession` | Session của user |
| `application` | `ServletContext` | Context của toàn app |
| `config` | `ServletConfig` | Cấu hình servlet |
| `pageContext` | `PageContext` | Page-level context |
| `page` | `Object` (this) | JSP page object |
| `exception` | `Throwable` | Chỉ có trong error pages |

```jsp
<!-- Sử dụng trong JSP -->
<% 
    String name = request.getParameter("name");
    session.setAttribute("user", name);
    out.println("Hello, " + name);
%>
```

---

## Q39. What is a PreparedStatement? Why use it over Statement?

`PreparedStatement` là precompiled SQL statement, tốt hơn `Statement` vì:

1. **Hiệu năng:** SQL được compile một lần, dùng nhiều lần
2. **Bảo mật:** Ngăn chặn **SQL Injection**
3. **Dễ đọc:** Tham số được đặt rõ ràng

```java
// Statement — nguy hiểm SQL injection!
String sql = "SELECT * FROM users WHERE name = '" + userName + "'";
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery(sql);

// PreparedStatement — an toàn
String sql = "SELECT * FROM users WHERE name = ?";
PreparedStatement ps = conn.prepareStatement(sql);
ps.setString(1, userName);
ResultSet rs = ps.executeQuery();

// SQL Injection attack với Statement:
// userName = "' OR '1'='1" → lấy toàn bộ users!
// PreparedStatement: tham số được escape, an toàn
```

**Các phương thức set:**
- `setString(index, value)`
- `setInt(index, value)`
- `setDouble(index, value)`
- `setDate(index, value)`
- `setNull(index, sqlType)`

---

## Q40. What is the difference between Array and ArrayList?

| | Array | ArrayList |
|--|-------|-----------|
| Size | Cố định | Động (resize tự động) |
| Type | Primitive + Object | Chỉ Object (wrapper) |
| Performance | Nhanh hơn | Chậm hơn (boxing/unboxing) |
| Generic | Không | Có |
| Methods | Ít (length) | Nhiều (add, remove, contains...) |
| Null | Có thể | Có thể |

```java
// Array
int[] arr = new int[5];
arr[0] = 1;
// arr[5] = 6; // ArrayIndexOutOfBoundsException

// ArrayList
ArrayList<Integer> list = new ArrayList<>();
list.add(1);
list.add(2);
list.remove(0); // xóa theo index
list.size();    // số phần tử
```

**Khi nào dùng Array:** Biết trước kích thước, hiệu năng quan trọng, primitive types  
**Khi nào dùng ArrayList:** Cần thêm/xóa động, cần nhiều utility methods

---

## Q41. What is the difference between StringBuffer and StringBuilder?

| | StringBuffer | StringBuilder |
|--|-------------|---------------|
| Thread-safe | **Có** (synchronized) | **Không** |
| Performance | Chậm hơn | **Nhanh hơn** |
| Introduced | Java 1.0 | Java 1.5 |
| Sử dụng | Multi-threaded | Single-threaded |

```java
// StringBuffer — thread-safe
StringBuffer sb = new StringBuffer("Hello");
sb.append(" World");
sb.insert(5, ",");
sb.reverse();
System.out.println(sb); // "dlroW ,olleH"

// StringBuilder — nhanh hơn, dùng trong single thread
StringBuilder sb2 = new StringBuilder("Hello");
sb2.append(" World");
System.out.println(sb2.toString());
```

**Quy tắc:** Luôn dùng `StringBuilder` trừ khi thực sự cần thread-safe.

---

## Q42. What is a Priority Queue?

`PriorityQueue` là queue có thứ tự ưu tiên, phần tử được lấy ra theo **thứ tự tự nhiên** (min-heap mặc định) hoặc theo `Comparator`.

```java
// Min-heap (mặc định)
PriorityQueue<Integer> pq = new PriorityQueue<>();
pq.add(5);
pq.add(1);
pq.add(3);
System.out.println(pq.poll()); // 1 (nhỏ nhất trước)
System.out.println(pq.poll()); // 3
System.out.println(pq.poll()); // 5

// Max-heap
PriorityQueue<Integer> maxPQ = new PriorityQueue<>(Collections.reverseOrder());
maxPQ.add(5); maxPQ.add(1); maxPQ.add(3);
System.out.println(maxPQ.poll()); // 5 (lớn nhất trước)

// Custom Comparator
PriorityQueue<String> pqStr = new PriorityQueue<>(
    Comparator.comparingInt(String::length)
);
pqStr.add("banana"); pqStr.add("apple"); pqStr.add("fig");
System.out.println(pqStr.poll()); // "fig" (ngắn nhất)
```

**Đặc điểm:**
- Không thread-safe (dùng `PriorityBlockingQueue` nếu cần)
- Không cho phép null
- Thêm/xóa: O(log n), xem phần tử đầu: O(1)

---

## Q43. What are the ways to create a thread in Java?

**Cách 1: Extends Thread**
```java
public class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Thread running: " + getName());
    }
}
MyThread t = new MyThread();
t.start();
```

**Cách 2: Implements Runnable (khuyến nghị)**
```java
public class MyRunnable implements Runnable {
    @Override
    public void run() {
        System.out.println("Runnable running");
    }
}
Thread t = new Thread(new MyRunnable());
t.start();

// Lambda (Java 8+)
Thread t2 = new Thread(() -> System.out.println("Lambda thread"));
t2.start();
```

**Cách 3: Implements Callable (có return value)**
```java
Callable<Integer> callable = () -> {
    return 42;
};
ExecutorService executor = Executors.newSingleThreadExecutor();
Future<Integer> future = executor.submit(callable);
System.out.println(future.get()); // 42
executor.shutdown();
```

**Cách 4: ExecutorService**
```java
ExecutorService pool = Executors.newFixedThreadPool(3);
pool.execute(() -> System.out.println("Task 1"));
pool.submit(() -> System.out.println("Task 2"));
pool.shutdown();
```

**Khuyến nghị:** Dùng `Runnable` + `ExecutorService` thay vì extends Thread.

---

## Q44. Why must we override `hashCode()` when overriding `equals()`?

**Hợp đồng (contract) của `hashCode()` và `equals()`:**
1. Nếu `a.equals(b)` == true, thì `a.hashCode()` == `b.hashCode()`
2. Ngược lại không cần thiết (collision cho phép)

**Nếu vi phạm contract:**
```java
public class Person {
    String name;
    
    @Override
    public boolean equals(Object o) {
        return ((Person) o).name.equals(this.name);
    }
    // KHÔNG override hashCode!
}

Person p1 = new Person("Alice");
Person p2 = new Person("Alice");

Map<Person, String> map = new HashMap<>();
map.put(p1, "Engineer");

System.out.println(p1.equals(p2)); // true
System.out.println(map.get(p2));   // null! — sai bucket!
```

**Giải thích:** HashMap dùng `hashCode()` để tìm bucket, rồi dùng `equals()` để tìm key. Nếu `hashCode()` khác nhau, tìm sai bucket, không bao giờ tìm thấy.

```java
// Đúng — override cả hai
@Override
public boolean equals(Object o) {
    if (!(o instanceof Person)) return false;
    return name.equals(((Person) o).name);
}

@Override
public int hashCode() {
    return Objects.hash(name);
}
```

---

## Q45. What is the `volatile` keyword?

`volatile` đảm bảo rằng mọi thread luôn đọc giá trị mới nhất của biến từ **main memory**, không từ CPU cache.

**Vấn đề không có volatile:**
```java
class SharedFlag {
    boolean running = true; // không volatile
    
    void stop() { running = false; }
    
    void run() {
        while (running) { // có thể đọc cached value mãi!
            // work
        }
    }
}
```

**Giải pháp với volatile:**
```java
class SharedFlag {
    volatile boolean running = true;
    
    void stop() { running = false; } // ghi vào main memory
    
    void run() {
        while (running) { // luôn đọc từ main memory
            // work
        }
    }
}
```

**Volatile KHÔNG đảm bảo atomicity:**
```java
volatile int count = 0;
count++; // KHÔNG thread-safe! = read + increment + write (3 ops)
// Dùng AtomicInteger thay thế
```

**Khi dùng volatile:** Biến chỉ được ghi bởi một thread, đọc bởi nhiều thread.

---

## Q46. What are varargs (`...`) in Java?

**Varargs** (variable arguments) cho phép method nhận số lượng tham số không xác định cùng kiểu.

```java
// Khai báo
public static int sum(int... numbers) {
    int total = 0;
    for (int n : numbers) total += n;
    return total;
}

// Gọi
System.out.println(sum(1, 2, 3));       // 6
System.out.println(sum(1, 2, 3, 4, 5)); // 15
System.out.println(sum());              // 0
System.out.println(sum(new int[]{1,2})); // dùng array cũng được
```

**Quy tắc:**
- Chỉ có một varargs per method
- Phải là tham số **cuối cùng**
- Bên trong method, là một array thông thường

```java
// Sai
public void bad(int... a, String... b) {} // lỗi compile

// Đúng
public void good(String prefix, int... numbers) {}
```

---

## Q47. What is the `transient` keyword?

`transient` đánh dấu field **không được serialize** khi object được ghi vào stream.

```java
public class User implements Serializable {
    private String username;
    private transient String password; // không serialize
    private transient int sessionId;   // không serialize
}

// Serialize
ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("user.dat"));
oos.writeObject(user);

// Deserialize
ObjectInputStream ois = new ObjectInputStream(new FileInputStream("user.dat"));
User restored = (User) ois.readObject();
System.out.println(restored.username); // "Alice"
System.out.println(restored.password); // null! (transient)
```

**Dùng transient khi:**
- Dữ liệu nhạy cảm (password, token)
- Dữ liệu có thể tính lại (derived fields)
- Tài nguyên hệ thống (connection, stream) không serialize được

---

## Q48. Java is pass-by-value or pass-by-reference?

**Java luôn là pass-by-value.**

Nhưng với objects, giá trị được pass là **reference (địa chỉ)** — không phải object.

```java
// Primitive — rõ ràng là pass-by-value
void changeInt(int x) {
    x = 100; // thay đổi copy local
}
int a = 5;
changeInt(a);
System.out.println(a); // vẫn là 5

// Object — pass reference BY VALUE
void changeName(Person p) {
    p.name = "Bob"; // thay đổi object qua reference — THÀNH CÔNG
}
void replaceObject(Person p) {
    p = new Person("Charlie"); // thay đổi local copy của reference — KHÔNG ảnh hưởng ngoài
}

Person person = new Person("Alice");
changeName(person);
System.out.println(person.name); // "Bob" — thay đổi qua reference

replaceObject(person);
System.out.println(person.name); // vẫn "Bob" — reference ngoài không đổi
```

**Tóm tắt:** Pass reference by value = có thể thay đổi nội dung object, nhưng không thể gán lại biến ngoài.

---

## Q49. What is a static initializer block?

**Static initializer block** chạy khi class được load, trước constructor.

```java
public class Config {
    static final Map<String, String> SETTINGS;
    static final int MAX_SIZE;
    
    // Static initializer
    static {
        SETTINGS = new HashMap<>();
        SETTINGS.put("host", "localhost");
        SETTINGS.put("port", "8080");
        MAX_SIZE = Integer.parseInt(System.getProperty("maxSize", "100"));
        System.out.println("Config class loaded!");
    }
    
    // Instance initializer (chạy trước constructor mỗi lần tạo object)
    {
        System.out.println("Instance created");
    }
    
    public Config() {
        System.out.println("Constructor called");
    }
}

// Output khi tạo new Config():
// Config class loaded!  (chỉ một lần)
// Instance created
// Constructor called
```

**Thứ tự khởi tạo:**
1. Static fields & static initializers (theo thứ tự trong class)
2. Instance fields & instance initializers (mỗi lần new)
3. Constructor

---

## Q50. Why use getters and setters (Encapsulation)?

```java
// Không dùng getter/setter — xấu
public class BankAccount {
    public double balance; // ai cũng có thể gán giá trị âm!
}
account.balance = -1000; // OK về compile, sai về logic

// Dùng getter/setter — tốt
public class BankAccount {
    private double balance;
    
    public double getBalance() {
        return balance;
    }
    
    public void setBalance(double balance) {
        if (balance < 0) throw new IllegalArgumentException("Balance cannot be negative");
        this.balance = balance; // validation
    }
    
    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        this.balance += amount;
    }
}
```

**Lợi ích:**
1. **Validation:** Kiểm tra dữ liệu trước khi set
2. **Encapsulation:** Ẩn implementation details
3. **Flexibility:** Có thể thay đổi internal implementation
4. **Read-only/Write-only:** Chỉ tạo getter hoặc setter
5. **Thread-safety:** Có thể thêm synchronization

---

## Q51. What are the access modifiers in Java?

| Modifier | Class | Package | Subclass | World |
|----------|-------|---------|----------|-------|
| `public` | Yes | Yes | Yes | Yes |
| `protected` | Yes | Yes | Yes | No |
| (default) | Yes | Yes | No | No |
| `private` | Yes | No | No | No |

```java
public class Example {
    public int pub = 1;       // mọi nơi
    protected int prot = 2;   // package + subclass
    int def = 3;              // chỉ package
    private int priv = 4;     // chỉ class này
    
    // Interface method — mặc định là public abstract
    // Class member — mặc định là package-private
}
```

**Lưu ý:**
- Constructor có thể là bất kỳ modifier nào
- `private` constructor → Singleton, utility class
- `protected` constructor → chỉ subclass có thể khởi tạo

---

## Q52. What are the differences between HashMap, LinkedHashMap, and TreeMap?

| | HashMap | LinkedHashMap | TreeMap |
|--|---------|---------------|---------|
| Thứ tự | Không | **Insertion order** | **Sorted (natural)** |
| Performance | O(1) | O(1) | O(log n) |
| Null key | 1 null key | 1 null key | **Không** |
| Thread-safe | Không | Không | Không |
| Implements | Map | Map | NavigableMap |

```java
// HashMap — không có thứ tự
Map<String, Integer> hm = new HashMap<>();
hm.put("banana", 2); hm.put("apple", 1); hm.put("cherry", 3);
System.out.println(hm); // {banana=2, cherry=3, apple=1} hoặc khác

// LinkedHashMap — giữ thứ tự thêm vào
Map<String, Integer> lhm = new LinkedHashMap<>();
lhm.put("banana", 2); lhm.put("apple", 1); lhm.put("cherry", 3);
System.out.println(lhm); // {banana=2, apple=1, cherry=3}

// TreeMap — sắp xếp theo key
Map<String, Integer> tm = new TreeMap<>();
tm.put("banana", 2); tm.put("apple", 1); tm.put("cherry", 3);
System.out.println(tm); // {apple=1, banana=2, cherry=3}
```

**Khi dùng:**
- `HashMap`: Cần tốc độ, không quan tâm thứ tự
- `LinkedHashMap`: LRU Cache, cần giữ thứ tự insert
- `TreeMap`: Cần sắp xếp theo key, range queries

---

## Q53. What are Annotations in Java?

**Annotations** là metadata được gắn vào code, không thay đổi logic chương trình nhưng cung cấp thông tin cho compiler, JVM, hoặc frameworks.

```java
// Built-in annotations
@Override          // báo compiler đây là override
@Deprecated        // đánh dấu cũ, không dùng nữa
@SuppressWarnings  // tắt compiler warning
@FunctionalInterface // đánh dấu functional interface

// Custom annotation
@Retention(RetentionPolicy.RUNTIME) // giữ đến runtime
@Target(ElementType.METHOD)         // chỉ dùng cho method
public @interface LogExecution {
    String value() default "INFO";
}

// Dùng annotation
@LogExecution("DEBUG")
public void processData() { ... }

// Đọc annotation bằng Reflection
Method method = MyClass.class.getMethod("processData");
LogExecution log = method.getAnnotation(LogExecution.class);
System.out.println(log.value()); // "DEBUG"
```

**Retention policies:**
- `SOURCE`: Chỉ trong source code (bị bỏ sau compile)
- `CLASS`: Trong .class file (mặc định)
- `RUNTIME`: Có thể đọc bằng Reflection lúc chạy

---

## Q54. What is a Classloader?

**Classloader** là component của JVM chịu trách nhiệm **load .class files** vào memory.

**Hệ thống phân cấp (Delegation Model):**
```
Bootstrap ClassLoader (JVM core — rt.jar, java.*)
    ↑
Extension ClassLoader (jre/lib/ext/*)
    ↑
Application ClassLoader (classpath — code của bạn)
    ↑
Custom ClassLoader (nếu cần)
```

**Quy trình load:**
1. Request load class
2. Kiểm tra cache (đã load chưa?)
3. Delegate lên parent
4. Nếu parent không tìm thấy, tự tìm

```java
// Xem classloader của một class
System.out.println(String.class.getClassLoader());    // null (Bootstrap)
System.out.println(MyClass.class.getClassLoader());  // AppClassLoader

// Custom ClassLoader
public class MyClassLoader extends ClassLoader {
    @Override
    protected Class<?> findClass(String name) throws ClassNotFoundException {
        byte[] bytes = loadClassData(name);
        return defineClass(name, bytes, 0, bytes.length);
    }
}
```

---

## Q55. Can we extend an Enum in Java?

**Không.** Enums KHÔNG thể được extend (kế thừa).

**Lý do:**
- Mỗi enum ngầm định `extends java.lang.Enum<E>`
- Java không hỗ trợ multiple inheritance
- Enum đã là `final` về mặt kế thừa

```java
enum Color { RED, GREEN, BLUE }
// enum ExtendedColor extends Color {} // Lỗi compile!

// Enum CÓ THỂ implement interface
interface Printable {
    void print();
}

enum Color implements Printable {
    RED, GREEN, BLUE;
    
    @Override
    public void print() {
        System.out.println("Color: " + name());
    }
}

// Enum có thể có abstract method
enum Operation {
    ADD {
        @Override
        public int apply(int a, int b) { return a + b; }
    },
    SUBTRACT {
        @Override
        public int apply(int a, int b) { return a - b; }
    };
    
    public abstract int apply(int a, int b);
}
```

---

## Q56. What is a JavaBean?

**JavaBean** là Java class tuân theo các quy ước (conventions):

1. **Serializable** — implements `Serializable`
2. **No-argument constructor** — có default constructor
3. **Private fields** — tất cả fields là private
4. **Getter/Setter** — public getter và setter cho mỗi field

```java
public class PersonBean implements Serializable {
    private static final long serialVersionUID = 1L;
    
    private String name;
    private int age;
    private boolean active;
    
    // No-arg constructor (bắt buộc)
    public PersonBean() {}
    
    // Getters
    public String getName() { return name; }
    public int getAge() { return age; }
    public boolean isActive() { return active; } // boolean dùng "is"
    
    // Setters
    public void setName(String name) { this.name = name; }
    public void setAge(int age) { this.age = age; }
    public void setActive(boolean active) { this.active = active; }
}
```

**JavaBean được dùng trong:** JSP (EL expressions), Spring, Hibernate, JavaFX, serialization frameworks.

---

## Q57. What is JIT (Just-In-Time) compilation?

**JIT Compiler** là component của JVM biên dịch bytecode → native machine code tại **runtime**, thay vì interpret từng dòng.

```
Source code (.java)
    ↓ javac (compile time)
Bytecode (.class)
    ↓ JVM (runtime)
Interpreter → JIT Compiler → Native code
```

**Quy trình JIT:**
1. JVM bắt đầu interpret bytecode
2. **Hotspot detection**: Phát hiện code chạy nhiều lần (hotspot)
3. JIT compile hotspot → native machine code
4. Lần sau gọi hotspot → chạy native code trực tiếp

**Lợi ích:**
- Lần đầu chạy: chậm (cần compile)
- Lần sau: nhanh hơn native (có thể tối ưu runtime)
- Tối ưu hóa: inlining, dead code elimination, loop unrolling

**Thực tế:** Đây là lý do Java ngày nay có hiệu năng gần bằng C++ trong nhiều use case.

---

## Q58. What are the advantages of JSP over Servlet?

| | JSP | Servlet |
|--|-----|---------|
| Dễ viết | Dễ (HTML + Java) | Khó (Java thuần) |
| Phân tách | View logic | Business logic |
| Compile | Tự động | Thủ công |
| Tag libraries | Có (JSTL) | Không |
| Designer-friendly | Có | Không |

```java
// Servlet — viết HTML rất khó
protected void doGet(HttpServletRequest req, HttpServletResponse res) {
    PrintWriter out = res.getWriter();
    out.println("<html><body>");
    out.println("<h1>Hello " + name + "</h1>");
    out.println("</body></html>");
}
```

```jsp
<!-- JSP — dễ viết, dễ đọc -->
<html>
<body>
    <h1>Hello ${name}</h1>
    <c:forEach items="${users}" var="user">
        <p>${user.name}</p>
    </c:forEach>
</body>
</html>
```

**Thực tế:** JSP compile thành Servlet khi deploy. JSP dùng cho View, Servlet dùng cho Controller.

---

## Q59. How to break out of nested loops in Java?

**Cách 1: Labeled break**
```java
outer:
for (int i = 0; i < 5; i++) {
    for (int j = 0; j < 5; j++) {
        if (i == 2 && j == 3) {
            break outer; // thoát cả hai vòng lặp
        }
        System.out.println(i + "," + j);
    }
}
System.out.println("Done"); // vẫn chạy
```

**Cách 2: Flag variable**
```java
boolean found = false;
for (int i = 0; i < 5 && !found; i++) {
    for (int j = 0; j < 5 && !found; j++) {
        if (i == 2 && j == 3) found = true;
    }
}
```

**Cách 3: Extract to method (Java 8+)**
```java
Optional<int[]> result = IntStream.range(0, 5)
    .boxed()
    .flatMap(i -> IntStream.range(0, 5).mapToObj(j -> new int[]{i, j}))
    .filter(pair -> pair[0] == 2 && pair[1] == 3)
    .findFirst();
```

**Khuyến nghị:** Dùng **labeled break** — rõ ràng và đơn giản nhất.

---

## Q60. What is process synchronization?

**Synchronization** đảm bảo chỉ một thread truy cập tài nguyên dùng chung tại một thời điểm.

```java
// Vấn đề không có synchronization
public class Counter {
    private int count = 0;
    
    public void increment() {
        count++; // KHÔNG thread-safe: read + add + write
    }
}
// Nhiều thread cùng increment → race condition!

// Giải pháp 1: synchronized method
public synchronized void increment() {
    count++;
}

// Giải pháp 2: synchronized block (fine-grained)
public void increment() {
    synchronized(this) {
        count++;
    }
}

// Giải pháp 3: AtomicInteger (tốt nhất cho counter)
private AtomicInteger count = new AtomicInteger(0);
public void increment() {
    count.incrementAndGet(); // atomic operation
}

// Giải pháp 4: ReentrantLock (linh hoạt nhất)
private final ReentrantLock lock = new ReentrantLock();
public void increment() {
    lock.lock();
    try { count++; }
    finally { lock.unlock(); }
}
```

---

## Q61. What is the difference between ClassNotFoundException and NoClassDefFoundError?

| | ClassNotFoundException | NoClassDefFoundError |
|--|----------------------|---------------------|
| Type | **Checked Exception** | **Error** |
| Khi nào | `Class.forName()`, `ClassLoader.loadClass()` | Class có lúc compile, mất lúc runtime |
| Cause | Class không tìm thấy trong classpath | Classpath thay đổi sau compile |
| Handle | try-catch | Khó xử lý |

```java
// ClassNotFoundException — có thể catch
try {
    Class<?> c = Class.forName("com.example.NonExistent");
} catch (ClassNotFoundException e) {
    System.out.println("Class not found: " + e.getMessage());
}

// NoClassDefFoundError — xảy ra khi:
// 1. Có MyClass.class lúc compile
// 2. Xóa MyClass.class trước khi chạy
// 3. JVM báo NoClassDefFoundError
MyClass obj = new MyClass(); // NoClassDefFoundError nếu .class bị xóa!
```

**Ví dụ thực tế:** Thiếu JAR trong dependency là nguyên nhân phổ biến của `NoClassDefFoundError`.

---

## Q62. Can we use non-static methods/fields in a static context?

**Không.** Static context không có `this` reference.

```java
public class Example {
    int instanceVar = 10;
    static int staticVar = 20;
    
    void instanceMethod() { System.out.println("instance"); }
    static void staticMethod() { System.out.println("static"); }
    
    static void doSomething() {
        System.out.println(staticVar);    // OK
        staticMethod();                   // OK
        
        // System.out.println(instanceVar); // Lỗi compile!
        // instanceMethod();               // Lỗi compile!
        
        // Giải pháp: tạo instance
        Example obj = new Example();
        System.out.println(obj.instanceVar); // OK qua object
        obj.instanceMethod();               // OK qua object
    }
}
```

**Lý do:** Static members thuộc về class, không thuộc về object. Không có object → không có instance members.

---

## Q63. What is the difference between `final`, `finally`, and `finalize()`?

**`final`** — keyword, ngăn thay đổi:
```java
final int x = 5;        // biến không thể gán lại
// x = 10;             // lỗi compile

final class A {}        // class không thể extend
// class B extends A{} // lỗi compile

final void method() {}  // method không thể override
```

**`finally`** — block trong try-catch, luôn chạy:
```java
try {
    riskyOperation();
} catch (Exception e) {
    handleError(e);
} finally {
    cleanup(); // luôn chạy, kể cả khi có exception
}
```

**`finalize()`** — method của Object, gọi trước GC:
```java
public class Resource {
    @Override
    protected void finalize() throws Throwable {
        try {
            closeConnection(); // dọn dẹp trước GC
        } finally {
            super.finalize();
        }
    }
}
// Lưu ý: finalize() không được đảm bảo gọi, không dùng cho critical cleanup
// Dùng try-with-resources thay thế
```

---

## Q64. What are restrictions on Applets?

Applet chạy trong sandbox với nhiều hạn chế bảo mật:

| Applet KHÔNG thể | Lý do |
|------------------|-------|
| Đọc/ghi file local | Bảo vệ filesystem |
| Kết nối đến server khác (chỉ được gọi về origin) | Ngăn data exfiltration |
| Chạy native code | Sandbox |
| Lấy thông tin hệ thống nhạy cảm | Privacy |
| Tạo network connection tùy ý | Security |
| Load native libraries | Sandbox |

**Trusted Applet** (có certificate) có thể vượt sandbox với permission của user.

**Lưu ý:** Applet đã bị deprecated từ Java 9 và bị remove trong Java 17. Không còn được hỗ trợ bởi các trình duyệt hiện đại.

---

## Q65. What is a Stub in RMI?

**Stub** là client-side proxy object đại diện cho remote object. Khi client gọi method trên stub, stub:
1. Serialize tham số (marshalling)
2. Gửi request qua network đến server
3. Nhận kết quả (deserialization)
4. Trả về cho client

```
Client Code
    ↓ gọi method
Stub (proxy, local)
    ↓ marshal + network call
Skeleton (server side)
    ↓ unmarshal + invoke
Remote Object Implementation
```

```java
// Interface
public interface Calculator extends Remote {
    int add(int a, int b) throws RemoteException;
}

// Client gọi qua stub
Calculator calc = (Calculator) Naming.lookup("rmi://localhost/Calculator");
int result = calc.add(3, 4); // thực ra gọi qua network!
```

**Skeleton:** Đối tác server-side của Stub, nhận request và gọi actual method.

---

## Q66. What is the difference between GenericServlet and HttpServlet?

| | GenericServlet | HttpServlet |
|--|----------------|-------------|
| Protocol | Bất kỳ (protocol-independent) | Chỉ HTTP |
| Abstract method | `service()` | `doGet()`, `doPost()`, etc. |
| Package | `javax.servlet` | `javax.servlet.http` |
| HTTP methods | Không hỗ trợ | GET, POST, PUT, DELETE... |
| Dùng khi | Custom protocol | Web applications |

```java
// GenericServlet
public class MyServlet extends GenericServlet {
    @Override
    public void service(ServletRequest req, ServletResponse res) {
        // xử lý request
    }
}

// HttpServlet (phổ biến hơn)
public class MyHttpServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse res) {
        // xử lý GET request
    }
    
    @Override
    protected void doPost(HttpServletRequest req, HttpServletResponse res) {
        // xử lý POST request
    }
}
```

---

## Q67. What are Scriptlets in JSP?

**Scriptlets** là Java code nhúng trực tiếp trong JSP giữa `<% %>`.

```jsp
<!-- Scriptlet -->
<%
    String name = request.getParameter("name");
    int count = 0;
    for (int i = 0; i < 5; i++) {
        count += i;
    }
    out.println("Sum: " + count);
%>

<!-- Expression (in ra giá trị) -->
<p>Hello, <%= name %></p>

<!-- Declaration (khai báo method/field của Servlet class) -->
<%!
    private int counter = 0;
    public int getCounter() { return ++counter; }
%>
```

**Vấn đề với Scriptlets:**
- Trộn lẫn Java code với HTML → khó maintain
- Khó test, khó debug
- Vi phạm separation of concerns

**Khuyến nghị:** Dùng **EL (Expression Language)** và **JSTL** thay thế:
```jsp
<!-- Tốt hơn với EL/JSTL -->
<p>Hello, ${param.name}</p>
<c:forEach items="${users}" var="user">
    <p>${user.name}</p>
</c:forEach>
```

---

## Q68. What is the lifecycle of an Applet?

```
init()      → Khởi tạo (gọi một lần khi load)
    ↓
start()     → Bắt đầu chạy (gọi sau init() và sau mỗi lần resume)
    ↓
paint()     → Vẽ giao diện (gọi bất cứ khi nào cần render)
    ↓
stop()      → Tạm dừng (khi user rời trang)
    ↓
destroy()   → Dọn dẹp (gọi trước khi unload)
```

```java
public class MyApplet extends Applet {
    @Override
    public void init() {
        System.out.println("init - khởi tạo");
    }
    
    @Override
    public void start() {
        System.out.println("start - bắt đầu");
    }
    
    @Override
    public void paint(Graphics g) {
        g.drawString("Hello Applet!", 50, 50);
    }
    
    @Override
    public void stop() {
        System.out.println("stop - dừng lại");
    }
    
    @Override
    public void destroy() {
        System.out.println("destroy - hủy");
    }
}
```

---

## Q69. What is the lifecycle of a Servlet?

```
Class loading → init() → service() [nhiều lần] → destroy()
```

```java
public class MyServlet extends HttpServlet {
    @Override
    public void init(ServletConfig config) throws ServletException {
        super.init(config);
        // Khởi tạo: load config, tạo connection pool
        // Gọi một lần khi servlet được load
    }
    
    @Override
    protected void service(HttpServletRequest req, HttpServletResponse res)
            throws ServletException, IOException {
        // Gọi mỗi HTTP request
        // Phân phối đến doGet(), doPost(), etc.
        super.service(req, res); // gọi doGet/doPost tự động
    }
    
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse res) {
        // Xử lý GET request
    }
    
    @Override
    public void destroy() {
        // Dọn dẹp: đóng connection, lưu state
        // Gọi một lần trước khi servlet bị unload
    }
}
```

**Lưu ý:** Servlet là singleton — một instance, nhiều thread. Fields của Servlet phải thread-safe.

---

## Q70. Which Swing methods are thread-safe?

Swing **không phải thread-safe** — hầu hết Swing operations phải chạy trên **Event Dispatch Thread (EDT)**.

**Các method thread-safe của Swing:**
- `repaint()` — request repaint từ bất kỳ thread
- `revalidate()` — request layout từ bất kỳ thread  
- `JTextComponent.setText()` — thread-safe
- `JTextArea.insert()` và `append()` — thread-safe

**Chạy code trên EDT:**
```java
// Sai — cập nhật UI từ background thread
new Thread(() -> {
    label.setText("Done"); // Nguy hiểm!
}).start();

// Đúng — dùng SwingUtilities.invokeLater
new Thread(() -> {
    String result = doLongTask();
    SwingUtilities.invokeLater(() -> {
        label.setText(result); // An toàn trên EDT
    });
}).start();

// invokeAndWait — block cho đến khi EDT xử lý xong
SwingUtilities.invokeAndWait(() -> {
    label.setText("Done");
});
```

**SwingWorker:** Pattern tốt nhất cho long-running tasks trong Swing.

---

## Q71. What is a deadlock? How to prevent it?

**Deadlock** xảy ra khi 2+ threads chờ nhau giải phóng lock → vòng chờ vô tận.

```java
// Deadlock điển hình
Object lock1 = new Object();
Object lock2 = new Object();

Thread t1 = new Thread(() -> {
    synchronized(lock1) {
        Thread.sleep(100);
        synchronized(lock2) { // chờ t2 nhả lock2
            System.out.println("T1 done");
        }
    }
});

Thread t2 = new Thread(() -> {
    synchronized(lock2) {
        Thread.sleep(100);
        synchronized(lock1) { // chờ t1 nhả lock1
            System.out.println("T2 done");
        }
    }
});
// t1 giữ lock1, chờ lock2
// t2 giữ lock2, chờ lock1 → DEADLOCK!
```

**Cách phòng tránh:**
1. **Thứ tự lock nhất quán:** Luôn lấy lock theo thứ tự cố định
2. **Timeout:** Dùng `tryLock(timeout)` thay vì `lock()`
3. **Tránh nested locks:** Đừng giữ lock khi chờ lock khác
4. **Lock fewer objects:** Giảm số lock cần thiết

```java
// Giải pháp: thống nhất thứ tự lấy lock
Thread t1 = new Thread(() -> {
    synchronized(lock1) { synchronized(lock2) { } }
});
Thread t2 = new Thread(() -> {
    synchronized(lock1) { synchronized(lock2) { } } // cùng thứ tự!
});
```

---

## Q72. What is the difference between ordered and unordered arrays?

| | Ordered Array | Unordered Array |
|--|---------------|-----------------|
| Sắp xếp | Có | Không |
| Tìm kiếm | Binary Search O(log n) | Linear Search O(n) |
| Thêm vào | O(n) — tìm vị trí + shift | O(1) — thêm cuối |
| Xóa | O(n) | O(n) — tìm + shift |
| Dùng khi | Tìm kiếm nhiều | Thêm xóa nhiều |

```java
// Unordered — thêm nhanh O(1)
int[] arr = {5, 2, 8, 1, 9};
// Tìm: phải duyệt tuần tự O(n)

// Ordered — tìm nhanh O(log n)
int[] sorted = {1, 2, 5, 8, 9};
int idx = Arrays.binarySearch(sorted, 5); // O(log n)

// Java Collections
List<Integer> list = new ArrayList<>();       // unordered
List<Integer> sortedList = new ArrayList<>();
Collections.sort(sortedList);                 // sắp xếp → binary search được
```

---

## Q73. What is the difference between doGet() and doPost()?

| | doGet() | doPost() |
|--|---------|----------|
| Data location | URL query string | Request body |
| Bookmark-able | Có | Không |
| Cached | Có thể | Không |
| Size limit | URL limit (~2KB) | Unlimited |
| Security | Data visible in URL | Ẩn trong body |
| Idempotent | Có (đọc dữ liệu) | Không (ghi dữ liệu) |
| Ví dụ | Tìm kiếm, filter | Login, form submit |

```java
@Override
protected void doGet(HttpServletRequest req, HttpServletResponse res) {
    String search = req.getParameter("q"); // ?q=java
    // Phù hợp cho: tìm kiếm, lấy dữ liệu
}

@Override
protected void doPost(HttpServletRequest req, HttpServletResponse res) {
    String password = req.getParameter("password"); // trong body
    // Phù hợp cho: đăng nhập, tạo/sửa dữ liệu
}
```

---

## Q74. Explain the Java Heap structure.

**Java Heap** được chia thành các vùng:

```
Java Heap
├── Young Generation (Eden + Survivor S0, S1)
│   ├── Eden Space    — Objects mới tạo ở đây
│   ├── Survivor 0    — Sống sót sau minor GC
│   └── Survivor 1    — Sống sót sau minor GC
└── Old Generation (Tenured)
    └── Objects sống lâu (sau nhiều lần GC)

Non-Heap (Metaspace từ Java 8)
└── Class metadata, method bytecode
```

**Quy trình GC:**
1. Object tạo trong **Eden**
2. **Minor GC:** Eden đầy → sống sót qua Survivor S0/S1
3. Sau nhiều Minor GC, object chuyển sang **Old Gen**
4. **Major/Full GC:** Dọn Old Gen (chậm hơn)

```java
// Kiểm tra heap size
Runtime rt = Runtime.getRuntime();
long totalMem = rt.totalMemory();
long freeMem = rt.freeMemory();
long usedMem = totalMem - freeMem;

// JVM flags
// -Xms512m: heap khởi đầu 512MB
// -Xmx2g:  heap tối đa 2GB
// -Xmn256m: young gen 256MB
```

---

## Q75. What is the difference between Iterator and ListIterator?

| | Iterator | ListIterator |
|--|----------|--------------|
| Hướng | Forward only | **Bidirectional** |
| Áp dụng cho | Mọi Collection | **Chỉ List** |
| Add element | Không | Có (`add()`) |
| Set element | Không | Có (`set()`) |
| Index | Không | Có (`nextIndex()`, `previousIndex()`) |

```java
List<String> list = new ArrayList<>(List.of("A", "B", "C"));

// Iterator — chỉ đi về trước
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    System.out.print(it.next() + " "); // A B C
}

// ListIterator — hai chiều
ListIterator<String> lit = list.listIterator(list.size()); // bắt đầu từ cuối
while (lit.hasPrevious()) {
    System.out.print(lit.previous() + " "); // C B A
}

// ListIterator — add và set
ListIterator<String> lit2 = list.listIterator();
while (lit2.hasNext()) {
    String s = lit2.next();
    if (s.equals("B")) {
        lit2.set("X"); // thay B bằng X
        lit2.add("Y"); // thêm Y sau X
    }
}
// list: [A, X, Y, C]
```

---

## Q76. What is the difference between `finally` and `finalize()`?

**`finally`** — block trong try-catch, **luôn chạy:**
```java
Connection conn = null;
try {
    conn = getConnection();
    executeQuery();
} catch (SQLException e) {
    log(e);
} finally {
    if (conn != null) conn.close(); // luôn đóng connection
}
```

**`finalize()`** — method của Object, **gọi TRƯỚC khi GC:**
```java
@Override
protected void finalize() throws Throwable {
    try {
        cleanup();
    } finally {
        super.finalize();
    }
}
```

| | finally | finalize() |
|--|---------|-----------|
| Loại | Keyword (block) | Method của Object |
| Đảm bảo chạy | **Có** (trừ `System.exit()`) | **Không** (GC không đảm bảo) |
| Thời điểm | Ngay sau try-catch | Trước khi GC |
| Khuyến nghị | Dùng cho cleanup | Đã deprecated Java 9 |

---

## Q77. What is the role of a JDBC Driver?

**JDBC Driver** là bridge giữa Java application và database cụ thể.

**4 loại JDBC Driver:**

| Type | Tên | Mô tả |
|------|-----|--------|
| 1 | JDBC-ODBC Bridge | Dùng ODBC driver (deprecated) |
| 2 | Native-API | Dùng native DB client library |
| 3 | Network Protocol | Middleware server |
| 4 | **Thin Driver** | Pure Java, TCP/IP trực tiếp (phổ biến nhất) |

```java
// Type 4 Driver (MySQL)
// 1. Load driver (Java 6+ tự động)
Class.forName("com.mysql.cj.jdbc.Driver");

// 2. Kết nối
Connection conn = DriverManager.getConnection(
    "jdbc:mysql://localhost:3306/mydb",
    "username",
    "password"
);

// 3. Query
PreparedStatement ps = conn.prepareStatement("SELECT * FROM users");
ResultSet rs = ps.executeQuery();
while (rs.next()) {
    System.out.println(rs.getString("name"));
}

// 4. Đóng
rs.close(); ps.close(); conn.close();
```

---

## Q78. What is the difference between `throw` and `throws`?

**`throw`** — ném exception ra (statement):
```java
public void validateAge(int age) {
    if (age < 0) {
        throw new IllegalArgumentException("Age cannot be negative: " + age);
    }
    if (age > 150) {
        throw new IllegalArgumentException("Unrealistic age: " + age);
    }
}

// Dùng throw để ném
throw new RuntimeException("Something went wrong");
```

**`throws`** — khai báo method có thể ném exception (declaration):
```java
public void readFile(String path) throws IOException, FileNotFoundException {
    // method này có thể ném IOException
    FileReader reader = new FileReader(path); // throws FileNotFoundException
    reader.read();                            // throws IOException
}

// Caller phải handle
try {
    readFile("data.txt");
} catch (IOException e) {
    e.printStackTrace();
}
```

| | `throw` | `throws` |
|--|---------|----------|
| Vị trí | Method body | Method signature |
| Số lượng | Một exception | Nhiều exception (dấu phẩy) |
| Loại | Checked + Unchecked | Checked + Unchecked |

---

## Q79. What is the difference between Comparable and Comparator?

| | Comparable | Comparator |
|--|------------|------------|
| Package | `java.lang` | `java.util` |
| Method | `compareTo(T o)` | `compare(T o1, T o2)` |
| Định nghĩa trong | Chính class | Class ngoài / Lambda |
| Thứ tự | **Natural ordering** | **Custom ordering** |
| Thay đổi | Phải sửa class | Không cần sửa class |

```java
// Comparable — natural ordering
public class Person implements Comparable<Person> {
    String name;
    int age;
    
    @Override
    public int compareTo(Person other) {
        return this.name.compareTo(other.name); // sắp theo tên
    }
}

List<Person> people = new ArrayList<>();
Collections.sort(people); // dùng compareTo

// Comparator — custom ordering
Comparator<Person> byAge = Comparator.comparingInt(p -> p.age);
Comparator<Person> byNameThenAge = Comparator.comparing((Person p) -> p.name)
                                             .thenComparingInt(p -> p.age);

people.sort(byAge);         // sắp theo tuổi
people.sort(byNameThenAge); // sắp theo tên, rồi tuổi
```

**Khi nào dùng:**
- `Comparable`: Class có thứ tự tự nhiên (String, Integer, Date)
- `Comparator`: Cần nhiều cách sắp xếp, hoặc không thể sửa class

---

## Q80. What restrictions apply to untrusted Applets?

Untrusted applets (chạy không có certificate) bị giới hạn:

1. **File system:** Không đọc/ghi file local
2. **Network:** Chỉ kết nối về **origin server** (same-origin policy)
3. **System info:** Không đọc nhiều system properties
4. **Native code:** Không chạy native code/libraries
5. **Clipboard:** Không truy cập clipboard
6. **Threads:** Không tạo ThreadGroup ở root level
7. **Reflection:** Bị hạn chế

**Trusted Applets:** Có digital signature, user chấp nhận → có thêm quyền.

**Ghi nhớ:** Applet đã lỗi thời — deprecated Java 9, removed Java 17.

---

## Q81. When to use ArrayList vs LinkedList?

| | ArrayList | LinkedList |
|--|-----------|------------|
| Internal | Dynamic array | Doubly linked list |
| Random access | O(1) | O(n) |
| Insert/delete đầu | O(n) — shift | **O(1)** |
| Insert/delete giữa | O(n) | O(n) — traverse |
| Insert/delete cuối | O(1) amortized | O(1) |
| Memory | Compact | Overhead mỗi node |
| Cache-friendly | Có | Không |

```java
// ArrayList tốt cho:
List<String> al = new ArrayList<>();
al.get(100);           // O(1) — random access nhanh
al.add("end");         // O(1) amortized — thêm cuối

// LinkedList tốt cho:
LinkedList<String> ll = new LinkedList<>();
ll.addFirst("head");   // O(1) — thêm đầu
ll.removeFirst();      // O(1) — xóa đầu
// Dùng như Queue/Deque
Deque<String> deque = new LinkedList<>();
deque.push("item");
deque.pop();
```

**Thực tế:** ArrayList thường tốt hơn trong hầu hết cases. LinkedList chỉ tốt hơn khi thêm/xóa ở đầu list rất nhiều.

---

## Q82. What is the difference between Applet and Servlet?

| | Applet | Servlet |
|--|--------|---------|
| Chạy ở | **Client** (browser) | **Server** |
| Giao diện | Có (AWT/Swing) | Không (HTML output) |
| Network | Hạn chế | Toàn quyền |
| Java cần | Trên client | Trên server |
| Trạng thái | Stateful | Stateless (theo request) |
| Deprecated | Java 9, removed 17 | Vẫn dùng |

```
Applet: Browser → download .class → chạy trong JVM của browser
Servlet: Browser → HTTP request → Server (JVM) → HTML response
```

---

## Q83. What are the steps to use RMI?

**Remote Method Invocation (RMI) — 6 bước:**

```java
// Bước 1: Tạo Remote Interface
public interface Calculator extends Remote {
    int add(int a, int b) throws RemoteException;
}

// Bước 2: Implement Remote Interface
public class CalculatorImpl extends UnicastRemoteObject implements Calculator {
    public CalculatorImpl() throws RemoteException { super(); }
    
    @Override
    public int add(int a, int b) throws RemoteException {
        return a + b;
    }
}

// Bước 3: Tạo và Register Stub
// (Java 5+: tự động, không cần rmic)

// Bước 4: Start RMI Registry và bind object
CalculatorImpl calc = new CalculatorImpl();
Registry registry = LocateRegistry.createRegistry(1099);
registry.bind("Calculator", calc);
// hoặc: Naming.rebind("rmi://localhost:1099/Calculator", calc);

// Bước 5: Client lookup và gọi
Registry registry = LocateRegistry.getRegistry("localhost", 1099);
Calculator calc = (Calculator) registry.lookup("Calculator");
// hoặc: Calculator calc = (Calculator) Naming.lookup("rmi://localhost/Calculator");

// Bước 6: Gọi remote method
int result = calc.add(3, 4); // gọi qua network!
System.out.println("3 + 4 = " + result);
```

---

## Q84. Are Collection classes Cloneable and Serializable?

**Cloneable:**
- Hầu hết Collection implementations đều implement `Cloneable`
- `ArrayList`, `LinkedList`, `HashMap`, `HashSet`, etc.
- Nhưng **interfaces** (`List`, `Map`, `Set`) KHÔNG extend `Cloneable`

**Serializable:**
- Hầu hết Collection implementations đều implement `Serializable`
- Cho phép serialize collection và các elements (nếu elements cũng Serializable)

```java
// Clone ArrayList (shallow copy)
ArrayList<String> original = new ArrayList<>(List.of("A", "B", "C"));
ArrayList<String> clone = (ArrayList<String>) original.clone();

// Serialize ArrayList
ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("list.dat"));
oos.writeObject(original); // works vì ArrayList implements Serializable

// Lưu ý: clone là SHALLOW copy
ArrayList<Person> persons = new ArrayList<>();
persons.add(new Person("Alice"));
ArrayList<Person> cloned = (ArrayList<Person>) persons.clone();
cloned.get(0).name = "Bob"; // thay đổi ảnh hưởng đến original!
```

---

## Q85. Can GC claim an object with a non-null reference?

**Có thể, trong trường hợp đặc biệt:**

**WeakReference:**
```java
// WeakReference — GC có thể collect bất kỳ lúc
Person person = new Person("Alice");
WeakReference<Person> weakRef = new WeakReference<>(person);

person = null; // xóa strong reference
System.gc();   // gợi ý GC

Person p = weakRef.get(); // null nếu đã bị collect!
```

**SoftReference:**
```java
// SoftReference — GC collect khi low memory
SoftReference<byte[]> cache = new SoftReference<>(new byte[1024 * 1024]);
byte[] data = cache.get(); // null nếu GC đã collect
```

**Thông thường:** Object có strong reference thì KHÔNG bị GC.

```java
Person p = new Person(); // strong reference → không bị GC
p = null; // giờ eligible for GC
```

**Tóm tắt:**
- Strong reference → không bị GC
- Weak reference → bị GC khi không có strong ref
- Soft reference → bị GC khi memory thấp

---

## Q86. Can a constructor be overloaded? What is a copy constructor?

**Constructor Overloading:**
```java
public class Rectangle {
    int width, height;
    
    // No-arg constructor
    public Rectangle() {
        this(1, 1); // gọi constructor khác
    }
    
    // One-arg constructor
    public Rectangle(int size) {
        this(size, size); // hình vuông
    }
    
    // Full constructor
    public Rectangle(int width, int height) {
        this.width = width;
        this.height = height;
    }
    
    // Copy constructor
    public Rectangle(Rectangle other) {
        this(other.width, other.height);
    }
}

// Sử dụng
Rectangle r1 = new Rectangle();      // 1x1
Rectangle r2 = new Rectangle(5);     // 5x5
Rectangle r3 = new Rectangle(3, 4);  // 3x4
Rectangle r4 = new Rectangle(r3);    // copy của r3
```

**Copy constructor** tạo deep copy, khác với `clone()` (shallow copy mặc định).

---

## Q87. What is the difference between Enumeration and Iterator?

| | Enumeration | Iterator |
|--|-------------|----------|
| Introduced | Java 1.0 | Java 1.2 |
| Methods | `hasMoreElements()`, `nextElement()` | `hasNext()`, `next()`, `remove()` |
| Remove | **Không thể** | **Có thể** |
| Fail-fast | Không | Có |
| Dùng với | Legacy (Vector, Hashtable) | Mọi Collection |

```java
// Enumeration (legacy)
Vector<String> v = new Vector<>(List.of("A", "B", "C"));
Enumeration<String> e = v.elements();
while (e.hasMoreElements()) {
    System.out.println(e.nextElement());
}

// Iterator (modern)
List<String> list = new ArrayList<>(List.of("A", "B", "C"));
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    if (s.equals("B")) it.remove(); // remove được!
}
```

**Khuyến nghị:** Dùng `Iterator` cho code mới. `Enumeration` chỉ để làm việc với legacy code.

---

## Q88. What is SSI (Server Side Include)?

**SSI (Server Side Include)** là directive nhúng nội dung động vào HTML tại server trước khi gửi về client.

```html
<!-- HTML với SSI directive -->
<!--#include virtual="/header.html" -->
<main>
    <h1>Content</h1>
    <!--#include virtual="/footer.html" -->
</main>

<!-- Include output của script -->
<!--#exec cgi="/cgi-bin/counter.cgi" -->

<!-- Echo variable -->
<!--#echo var="DATE_LOCAL" -->
```

**Các directive phổ biến:**
- `#include` — nhúng file
- `#exec` — chạy CGI/shell command
- `#echo` — hiển thị server variable
- `#set` — đặt biến
- `#if/#elif/#else/#endif` — điều kiện

**Hạn chế:** SSI ít được dùng ngày nay, thay bởi JSP, PHP, template engines.

---

## Q89. Why doesn't Java support multiple inheritance?

Java không hỗ trợ **multiple inheritance với class** để tránh **Diamond Problem**:

```
        Animal
       /      \
    Dog       Cat
       \      /
       DogCat? ← Which speak() to use?
```

```java
class Animal { void speak() { System.out.println("..."); } }
class Dog extends Animal { void speak() { System.out.println("Woof"); } }
class Cat extends Animal { void speak() { System.out.println("Meow"); } }
// class DogCat extends Dog, Cat {} // Lỗi! Ambiguous!
```

**Java giải quyết bằng Interface:**
```java
interface Flyable { 
    default void fly() { System.out.println("Flying"); }
}
interface Swimmable { 
    default void swim() { System.out.println("Swimming"); }
}

// Class có thể implement nhiều interface
class Duck extends Animal implements Flyable, Swimmable {
    // Nếu cả hai interface có cùng default method, phải override
}
```

**Từ Java 8:** Interface có `default` method → "pseudo multiple inheritance" nhưng phải giải quyết conflict thủ công.

---

## Q90. What is the difference between Event Listener and Event Adapter?

**Event Listener:** Interface với nhiều abstract methods:
```java
// Phải implement TẤT CẢ methods
class MyMouseListener implements MouseListener {
    @Override public void mouseClicked(MouseEvent e) { /* handle */ }
    @Override public void mousePressed(MouseEvent e) {} // phải có dù không dùng
    @Override public void mouseReleased(MouseEvent e) {}
    @Override public void mouseEntered(MouseEvent e) {}
    @Override public void mouseExited(MouseEvent e) {}
}
```

**Event Adapter:** Abstract class với empty implementations:
```java
// Chỉ override những method cần
class MyMouseAdapter extends MouseAdapter {
    @Override
    public void mouseClicked(MouseEvent e) {
        System.out.println("Clicked at " + e.getX() + "," + e.getY());
    }
    // Không cần implement các method khác!
}

button.addMouseListener(new MyMouseAdapter());
```

**Khi nào dùng:**
- **Listener:** Khi class đã extends class khác, hoặc cần implement nhiều interface
- **Adapter:** Khi chỉ cần vài event handlers, code gọn hơn

---

## Q91. What is a Security Manager in Applet?

**SecurityManager** kiểm soát các operation nhạy cảm trong Java, đặc biệt cho Applets:

```java
// SecurityManager kiểm tra permission trước khi:
// - Đọc/ghi file
// - Mở network connection
// - Chạy subprocess
// - Thoát JVM (System.exit)
// - Load native library

// Kiểm tra permission thủ công
SecurityManager sm = System.getSecurityManager();
if (sm != null) {
    sm.checkRead("/etc/passwd"); // ném SecurityException nếu không được phép
}

// Custom SecurityManager (không khuyến nghị cho production)
System.setSecurityManager(new SecurityManager() {
    @Override
    public void checkRead(String file) {
        if (file.contains("secret")) {
            throw new SecurityException("Access denied: " + file);
        }
    }
});
```

**Lưu ý:** SecurityManager đã bị **deprecated trong Java 17** và sẽ bị remove.

---

## Q92. What is the Applet load lifecycle?

Khi browser load applet:

```
1. Browser gặp <applet> tag trong HTML
2. Download .class file (hoặc .jar) từ server
3. Tạo instance của Applet class
4. Gọi init() — khởi tạo UI, resources
5. Gọi start() — bắt đầu animation, threads
6. Gọi paint() — vẽ giao diện
--- User tương tác ---
7. Gọi stop() khi user rời trang hoặc minimize
8. Gọi start() khi user quay lại
--- User đóng trang ---
9. Gọi stop()
10. Gọi destroy() — cleanup
```

```java
public class MyApplet extends JApplet {
    private Timer animationTimer;
    
    @Override
    public void init() {
        // Chỉ gọi 1 lần — khởi tạo
        setLayout(new BorderLayout());
        add(new JButton("Click me"), BorderLayout.CENTER);
    }
    
    @Override
    public void start() {
        // Gọi mỗi khi hiển thị
        animationTimer = new Timer(100, e -> repaint());
        animationTimer.start();
    }
    
    @Override
    public void stop() {
        // Gọi khi ẩn đi
        if (animationTimer != null) animationTimer.stop();
    }
    
    @Override
    public void destroy() {
        // Dọn dẹp cuối cùng
    }
}
```

---

## Q93. What is the purpose of `finalize()`?

`finalize()` cho phép object thực hiện cleanup trước khi bị **Garbage Collected**.

```java
public class DatabaseConnection {
    private Connection conn;
    
    public DatabaseConnection() {
        conn = DriverManager.getConnection("...");
    }
    
    @Override
    protected void finalize() throws Throwable {
        try {
            if (conn != null && !conn.isClosed()) {
                conn.close(); // đóng connection trước GC
                System.out.println("Connection closed in finalize");
            }
        } finally {
            super.finalize(); // luôn gọi super
        }
    }
}
```

**Vấn đề với finalize():**
- **Không đảm bảo** khi nào được gọi
- **Không đảm bảo** có được gọi không (nếu JVM tắt)
- Làm chậm GC (object cần 2 GC cycles)
- Có thể gây memory leak
- **Deprecated từ Java 9**

**Giải pháp tốt hơn:**
```java
// try-with-resources (Java 7+) — tốt nhất
try (DatabaseConnection conn = new DatabaseConnection()) {
    // dùng conn
} // tự động đóng

// Implements AutoCloseable
public class DatabaseConnection implements AutoCloseable {
    @Override
    public void close() {
        conn.close();
    }
}
```

---

## Q94. Summary: Mid-Level Java Concepts

Tổng kết các concepts quan trọng ở level Mid:

| Concept | Key Point |
|---------|-----------|
| Fail-fast/safe | modCount vs snapshot |
| sleep vs wait | giữ lock vs nhả lock |
| volatile | visibility, không atomicity |
| transient | không serialize |
| Deadlock | circular lock dependency |
| HashMap vs LinkedHashMap vs TreeMap | unordered / insertion / sorted |
| Comparable vs Comparator | natural / custom ordering |
| ArrayList vs LinkedList | random access / frequent insert-delete |
| StringBuilder vs StringBuffer | single vs multi-thread |
| throw vs throws | ném vs khai báo |
| final/finally/finalize | const/cleanup/GC hook |
| ClassNotFoundException vs NoClassDefFoundError | missing class vs missing at runtime |
| Iterator vs ListIterator | forward only / bidirectional |
| Enum vs Iterator | legacy vs modern |
| doGet vs doPost | URL / body |
