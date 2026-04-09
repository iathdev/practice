# Java Interview Questions — Junior (Q1–Q32)

---

## Q1. What is an Iterator?
> **Dịch:** Iterator là gì?

`Iterator` là một interface trong `java.util` cung cấp các phương thức để duyệt qua bất kỳ `Collection` nào.

**Các phương thức chính:**
- `hasNext()` — trả về `true` nếu còn phần tử
- `next()` — trả về phần tử tiếp theo
- `remove()` — xóa phần tử hiện tại khỏi collection

```java
List<String> list = new ArrayList<>(List.of("A", "B", "C"));
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    String s = it.next();
    if (s.equals("B")) it.remove(); // safe removal during iteration
}
```

**Lưu ý:** Dùng Iterator khi cần xóa phần tử trong lúc duyệt (dùng for-each sẽ gây `ConcurrentModificationException`).

---

## Q2. What are the Data Types supported by Java? What is Autoboxing and Unboxing?
> **Dịch:** Java hỗ trợ những kiểu dữ liệu nào? Autoboxing và Unboxing là gì?

**Primitive types (8 loại):**

| Type    | Size    | Default |
|---------|---------|---------|
| byte    | 8 bit   | 0       |
| short   | 16 bit  | 0       |
| int     | 32 bit  | 0       |
| long    | 64 bit  | 0L      |
| float   | 32 bit  | 0.0f    |
| double  | 64 bit  | 0.0d    |
| char    | 16 bit  | '\u0000'|
| boolean | 1 bit   | false   |

**Reference types:** class, interface, array, enum

**Autoboxing:** Tự động chuyển primitive → Wrapper class  
```java
int i = 5;
Integer boxed = i; // autoboxing
```

**Unboxing:** Tự động chuyển Wrapper class → primitive  
```java
Integer boxed = 10;
int i = boxed; // unboxing
```

---

## Q3. What are the basic interfaces of Java Collections Framework?
> **Dịch:** Các interface cơ bản của Java Collections Framework là gì?

```
Collection
├── List       — ordered, allows duplicates (ArrayList, LinkedList)
├── Set        — no duplicates (HashSet, TreeSet, LinkedHashSet)
└── Queue      — FIFO (LinkedList, PriorityQueue)

Map            — key-value pairs (HashMap, TreeMap, LinkedHashMap)
```

**Chi tiết:**
- `List` — có thứ tự, cho phép trùng lặp
- `Set` — không cho phép trùng lặp
- `Queue` — FIFO, dùng cho hàng đợi
- `Deque` — double-ended queue
- `Map` — ánh xạ key → value, key không trùng

---

## Q4. How does HashMap work in Java?
> **Dịch:** HashMap hoạt động như thế nào trong Java?

`HashMap` sử dụng **hash table** nội bộ (mảng các bucket).

**Quá trình PUT:**
1. Tính `hashCode()` của key
2. Tính index: `index = hash & (capacity - 1)`
3. Nếu bucket trống → lưu entry
4. Nếu collision → dùng **chaining** (LinkedList, Java 8+ dùng TreeNode khi ≥ 8 entries)

**Quá trình GET:**
1. Tính hash của key
2. Tìm bucket
3. So sánh bằng `equals()` để tìm đúng entry

**Đặc điểm:**
- Cho phép `null` key (một lần) và `null` value
- Không thread-safe
- Load factor mặc định: 0.75, capacity mặc định: 16

---

## Q5. What is a Java Applet?
> **Dịch:** Java Applet là gì?

`Applet` là chương trình Java nhỏ chạy bên trong trình duyệt web (thông qua JVM plugin).

**Đặc điểm:**
- Extend `java.applet.Applet`
- Chạy trong sandbox (bị giới hạn quyền truy cập)
- **Đã bị deprecated** và bị xóa từ Java 11

**Vòng đời:** `init()` → `start()` → `paint()` → `stop()` → `destroy()`

> Ngày nay đã bị thay thế hoàn toàn bởi web technologies (HTML5, JavaScript).

---

## Q6. What are pass by reference and pass by value?
> **Dịch:** Truyền tham chiếu và truyền giá trị là gì?

**Pass by value:** Truyền bản sao của giá trị — thay đổi trong method không ảnh hưởng biến gốc.

**Pass by reference:** Truyền địa chỉ bộ nhớ — thay đổi trong method ảnh hưởng biến gốc.

**Java luôn dùng pass by value:**
```java
// Với primitive: hoàn toàn độc lập
void change(int x) { x = 100; }
int a = 5;
change(a); // a vẫn = 5

// Với object: truyền bản sao của tham chiếu (reference copy)
void modify(List<String> list) {
    list.add("X");      // ảnh hưởng object gốc (cùng địa chỉ)
    list = new ArrayList<>(); // KHÔNG ảnh hưởng tham chiếu gốc
}
```

---

## Q7. What is the difference between processes and threads?
> **Dịch:** Sự khác nhau giữa Process và Thread là gì?

| Tiêu chí         | Process                        | Thread                          |
|------------------|--------------------------------|---------------------------------|
| Định nghĩa       | Chương trình đang chạy         | Đơn vị thực thi trong process   |
| Bộ nhớ           | Riêng biệt (isolated)          | Chia sẻ trong cùng process      |
| Giao tiếp        | IPC (socket, pipe...)          | Shared memory trực tiếp         |
| Tạo/hủy          | Chi phí cao                    | Chi phí thấp hơn                |
| Ảnh hưởng crash  | Không ảnh hưởng process khác   | Có thể crash cả process         |

---

## Q8. When does an Object become eligible for Garbage Collection?
> **Dịch:** Khi nào một Object đủ điều kiện để Garbage Collection thu hồi?

Một object trở thành eligible cho GC khi **không còn reference nào** trỏ đến nó từ reachable code.

**Các trường hợp:**
```java
// 1. Reference bị set null
Object obj = new Object();
obj = null;

// 2. Reference ra ngoài scope
{
    Object obj = new Object();
} // obj eligible sau đây

// 3. Reference bị ghi đè
Object obj = new Object();
obj = new Object(); // object cũ eligible

// 4. Island of isolation (objects tham chiếu nhau nhưng không reachable từ root)
```

---

## Q9. What is the difference between Exception and Error in Java?
> **Dịch:** Sự khác nhau giữa Exception và Error trong Java là gì?

```
Throwable
├── Error        — JVM-level problems, không nên catch
│   ├── OutOfMemoryError
│   ├── StackOverflowError
│   └── VirtualMachineError
└── Exception    — application-level problems
    ├── Checked Exception   — phải xử lý (IOException, SQLException)
    └── RuntimeException    — không bắt buộc (NullPointerException, IllegalArgumentException)
```

**Error:** Vấn đề nghiêm trọng của JVM, không thể recover được → không nên catch.  
**Exception:** Vấn đề của ứng dụng, có thể xử lý.

---

## Q10. What is the importance of `finally` block in exception handling?
> **Dịch:** Tầm quan trọng của khối `finally` trong xử lý ngoại lệ là gì?

`finally` luôn được thực thi dù có exception hay không, dùng để **giải phóng tài nguyên**.

```java
Connection conn = null;
try {
    conn = getConnection();
    // ... xử lý
} catch (SQLException e) {
    e.printStackTrace();
} finally {
    if (conn != null) conn.close(); // luôn chạy
}
```

**Khi nào finally KHÔNG chạy:**
- `System.exit()` được gọi
- JVM crash
- Thread bị kill

> Từ Java 7, nên dùng **try-with-resources** thay thế.

---

## Q11. What does `static` keyword mean? Can you override `private` or `static` method?
> **Dịch:** Từ khóa `static` nghĩa là gì? Có thể override method `private` hoặc `static` không?

**`static` có nghĩa là:**
- Thuộc về class, không thuộc về instance
- Dùng chung cho tất cả objects
- Có thể truy cập mà không cần tạo object

```java
class Counter {
    static int count = 0;    // class variable
    static void reset() { count = 0; } // class method
}
Counter.reset(); // gọi không cần new
```

**Override:**
- `private` method: **Không thể override** (không visible với subclass)
- `static` method: **Không thể override** — chỉ có thể **hide** (method hiding)

```java
class Parent { static void foo() { System.out.println("Parent"); } }
class Child extends Parent { static void foo() { System.out.println("Child"); } }
Parent p = new Child();
p.foo(); // in "Parent" — không phải polymorphism
```

---

## Q12. What is JDBC?
> **Dịch:** JDBC là gì?

**JDBC (Java Database Connectivity)** là API chuẩn của Java để kết nối và tương tác với database.

**Các bước sử dụng JDBC:**
```java
// 1. Load driver
Class.forName("com.mysql.cj.jdbc.Driver");

// 2. Tạo connection
Connection conn = DriverManager.getConnection(url, user, password);

// 3. Tạo statement
PreparedStatement ps = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
ps.setInt(1, 1);

// 4. Thực thi query
ResultSet rs = ps.executeQuery();

// 5. Xử lý kết quả
while (rs.next()) {
    System.out.println(rs.getString("name"));
}

// 6. Đóng resources
rs.close(); ps.close(); conn.close();
```

---

## Q13. What is the purpose of Garbage Collection in Java, and when is it used?
> **Dịch:** Mục đích của Garbage Collection trong Java là gì và khi nào được sử dụng?

**Mục đích:** Tự động thu hồi bộ nhớ của các object không còn được sử dụng → tránh memory leak.

**Khi nào GC chạy:**
- Khi JVM thấy heap sắp đầy
- Khi gọi `System.gc()` (chỉ là suggestion, không đảm bảo)
- Theo schedule của GC algorithm

**Các thuật toán GC:** Serial GC, Parallel GC, G1 GC (default Java 9+), ZGC, Shenandoah

---

## Q14. How does Garbage Collection prevent a Java application from going out of memory?
> **Dịch:** Garbage Collection ngăn ứng dụng Java bị tràn bộ nhớ như thế nào?

GC **không đảm bảo** tránh hoàn toàn `OutOfMemoryError`. GC chỉ thu hồi memory của các object **không còn reachable**.

Nếu app liên tục tạo object và giữ reference → GC không thể thu hồi → vẫn OOM.

**Để tránh OOM:**
- Giải phóng references khi không dùng (`obj = null`)
- Dùng try-with-resources
- Dùng weak/soft references cho cache
- Tăng heap size: `-Xmx512m`

---

## Q15. Explain Serialization and Deserialization.
> **Dịch:** Giải thích Serialization và Deserialization.

**Serialization:** Chuyển object → stream of bytes (để lưu file, gửi qua network).  
**Deserialization:** Chuyển stream of bytes → object.

```java
// Serializable class
class User implements Serializable {
    private static final long serialVersionUID = 1L;
    String name;
    transient String password; // không serialize
}

// Serialize
ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("user.dat"));
oos.writeObject(user);

// Deserialize
ObjectInputStream ois = new ObjectInputStream(new FileInputStream("user.dat"));
User user = (User) ois.readObject();
```

---

## Q16. Explain the architecture of a Servlet.
> **Dịch:** Giải thích kiến trúc của Servlet.

```
Client (Browser)
    ↓ HTTP Request
Web Server (Tomcat)
    ↓
Servlet Container
    ↓ tạo Request/Response objects
Servlet (doGet/doPost)
    ↓ xử lý logic
    ↓ HTTP Response
Client
```

**Servlet Container** quản lý vòng đời: `init()` → `service()` → `destroy()`

**`service()` điều phối đến:** `doGet()`, `doPost()`, `doPut()`, `doDelete()`...

---

## Q17. What is reflection and why is it useful?
> **Dịch:** Reflection là gì và tại sao hữu ích?

**Reflection** cho phép inspect và modify behavior của class/method/field tại runtime.

```java
Class<?> clazz = Class.forName("com.example.MyClass");
Object instance = clazz.getDeclaredConstructor().newInstance();

Method method = clazz.getDeclaredMethod("privateMethod");
method.setAccessible(true);
method.invoke(instance);
```

**Use cases:**
- Frameworks (Spring, Hibernate) — DI, ORM mapping
- Testing (Mockito) — mock private methods
- Serialization libraries (Jackson, Gson)
- Plugin systems

**Nhược điểm:** Chậm hơn, bypass access control, khó debug.

---

## Q18. What are JSP Actions?
> **Dịch:** JSP Actions là gì?

JSP Actions là các XML tags đặc biệt dùng để control Servlet engine và reuse JavaBeans.

**Các actions phổ biến:**
```jsp
<jsp:include page="header.jsp" />       <!-- include file -->
<jsp:forward page="result.jsp" />       <!-- forward request -->
<jsp:useBean id="user" class="User" scope="session" />
<jsp:setProperty name="user" property="name" value="John" />
<jsp:getProperty name="user" property="name" />
<jsp:param name="key" value="val" />
```

---

## Q19. What are Declarations?

Trong JSP, **Declarations** dùng để khai báo biến và method ở mức class (không phải trong `_jspService()`).

```jsp
<%!
    int count = 0;           // instance variable của Servlet class

    String greet(String name) {
        return "Hello, " + name;
    }
%>
```

**Khác với Scriptlets (`<% %>`):** Scriptlets khai báo biến local trong method `_jspService()`.

---

## Q20. What's the difference between `sendRedirect` and `forward`?

| Tiêu chí         | `sendRedirect`                      | `forward`                           |
|------------------|-------------------------------------|-------------------------------------|
| Thực hiện        | Client-side (2 request)             | Server-side (1 request)             |
| URL thay đổi     | Có                                  | Không                               |
| Request/Response | Tạo mới                             | Giữ nguyên                          |
| Dùng khi         | Redirect sang URL khác/external     | Forward trong cùng app              |
| Tốc độ           | Chậm hơn                            | Nhanh hơn                           |

```java
// sendRedirect
response.sendRedirect("https://example.com");

// forward
RequestDispatcher rd = request.getRequestDispatcher("/result.jsp");
rd.forward(request, response);
```

---

## Q21. What is the purpose of `Class.forName` method?

`Class.forName(String className)` nạp dynamically một class vào JVM tại runtime và trả về `Class` object.

```java
// Nạp JDBC driver (cách cũ)
Class.forName("com.mysql.cj.jdbc.Driver");

// Dùng với reflection
Class<?> clazz = Class.forName("com.example.MyPlugin");
Object instance = clazz.getDeclaredConstructor().newInstance();
```

**Thực chất:** Gọi ClassLoader để load, link và initialize class đó.

---

## Q22. What is the design pattern that Java uses for all Swing components?

Java Swing dùng **Model-View-Controller (MVC)** pattern.

- **Model:** Dữ liệu (ví dụ: `DefaultTableModel`, `DefaultListModel`)
- **View:** Hiển thị UI (ví dụ: `JTable`, `JList`)
- **Controller:** Xử lý sự kiện (ví dụ: `ActionListener`)

```java
// JButton là View
JButton btn = new JButton("Click");

// ActionListener là Controller
btn.addActionListener(e -> System.out.println("Clicked!"));
```

> Thực ra Swing dùng biến thể gọi là **Separable Model Architecture**.

---

## Q23. How are JSP requests handled?

1. Client gửi request đến `.jsp` file
2. **JSP Container** kiểm tra xem JSP đã được compile chưa
3. Nếu chưa (hoặc đã thay đổi): compile JSP → **Servlet** (`.java` → `.class`)
4. Servlet được load và khởi tạo (`init()`)
5. `service()` / `_jspService()` xử lý request
6. Response trả về client

**Lần sau:** Servlet đã compiled sẵn → bỏ qua bước compile → nhanh hơn.

---

## Q24. What is Function Overriding and Overloading in Java?

**Overloading** (compile-time polymorphism): Cùng tên method, khác signature trong cùng class.
```java
int add(int a, int b) { return a + b; }
double add(double a, double b) { return a + b; }
```

**Overriding** (runtime polymorphism): Subclass định nghĩa lại method của superclass.
```java
class Animal { void sound() { System.out.println("..."); } }
class Dog extends Animal {
    @Override
    void sound() { System.out.println("Woof"); }
}
```

| Tiêu chí      | Overloading       | Overriding           |
|---------------|-------------------|----------------------|
| Class         | Cùng class        | Khác class (kế thừa) |
| Return type   | Có thể khác       | Phải tương thích     |
| Polymorphism  | Compile-time      | Runtime              |

---

## Q25. What do you know about Big-O notation?

**Big-O** mô tả độ phức tạp thời gian/không gian của thuật toán theo kích thước input.

**Các độ phức tạp phổ biến:**

| Notation    | Ví dụ                              |
|-------------|------------------------------------|
| O(1)        | HashMap get/put, Array index access|
| O(log n)    | Binary search, TreeMap get/put     |
| O(n)        | Linear search, ArrayList iteration |
| O(n log n)  | Merge sort, Arrays.sort()          |
| O(n²)       | Bubble sort, nested loops          |

**Theo data structure:**

| Structure       | Access | Search | Insert | Delete |
|-----------------|--------|--------|--------|--------|
| ArrayList       | O(1)   | O(n)   | O(n)   | O(n)   |
| LinkedList      | O(n)   | O(n)   | O(1)   | O(1)   |
| HashMap         | O(1)   | O(1)   | O(1)   | O(1)   |
| TreeMap         | O(log n)| O(log n)| O(log n)| O(log n)|

---

## Q26. What are Expressions?

Trong JSP, **Expressions** dùng để in giá trị ra output stream.

```jsp
<%= expression %>
<!-- tương đương: out.print(expression) -->

<%= user.getName() %>
<%= new java.util.Date() %>
<%= 2 + 2 %>
```

**Lưu ý:** Không có dấu `;` ở cuối. Từ JSP 2.0, nên dùng **EL (Expression Language)** thay thế: `${user.name}`.

---

## Q27. What is the difference between Interface and Abstract class?

| Tiêu chí              | Interface                         | Abstract class                    |
|-----------------------|-----------------------------------|-----------------------------------|
| Keyword               | `interface`                       | `abstract class`                  |
| Multiple inheritance  | Có (implements nhiều)             | Không (extends một)               |
| Fields                | Chỉ `public static final`         | Mọi loại                          |
| Methods               | `default`, `static`, abstract     | Abstract và concrete              |
| Constructor           | Không có                          | Có                                |
| Access modifiers      | Public mặc định                   | Tùy ý                             |
| Dùng khi              | Định nghĩa contract/capability    | Chia sẻ code chung + có template  |

```java
interface Flyable { void fly(); }
abstract class Vehicle {
    abstract void move();
    void refuel() { System.out.println("Refueling"); } // concrete method
}
```

---

## Q28. What will happen to the Exception object after exception handling?

Sau khi exception được catch và xử lý, **Exception object trở thành unreachable** (nếu không còn reference) và sẽ được **Garbage Collected** như các object thông thường khác.

```java
try {
    throw new RuntimeException("error");
} catch (RuntimeException e) {
    // e là reference đến exception object
    log(e); // xử lý
} // sau đây, e out of scope → eligible for GC
```

---

## Q29. Explain what is Binary Search

**Binary Search** tìm kiếm phần tử trong **mảng đã sắp xếp** bằng cách liên tục chia đôi khoảng tìm kiếm.

**Thuật toán:**
1. So sánh target với phần tử giữa
2. Nếu bằng → tìm thấy
3. Nếu target < mid → tìm ở nửa trái
4. Nếu target > mid → tìm ở nửa phải
5. Lặp lại cho đến khi tìm thấy hoặc hết

```java
int binarySearch(int[] arr, int target) {
    int left = 0, right = arr.length - 1;
    while (left <= right) {
        int mid = left + (right - left) / 2;
        if (arr[mid] == target) return mid;
        if (arr[mid] < target) left = mid + 1;
        else right = mid - 1;
    }
    return -1;
}
```

**Độ phức tạp:** O(log n) — hiệu quả hơn linear search O(n) rất nhiều.

---

## Q30. What are Directives?

JSP Directives cung cấp thông tin toàn cục về toàn bộ JSP page cho Container.

**3 loại directive:**
```jsp
<%@ page language="java" contentType="text/html; charset=UTF-8"
         import="java.util.*" errorPage="error.jsp" %>

<%@ include file="header.jsp" %>

<%@ taglib uri="http://java.sun.com/jsp/jstl/core" prefix="c" %>
```

- `page` — cấu hình page (import, session, errorPage...)
- `include` — nhúng file tĩnh tại compile time
- `taglib` — khai báo custom tag library

---

## Q31. What differences exist between HashMap and Hashtable?

| Tiêu chí         | HashMap               | Hashtable              |
|------------------|-----------------------|------------------------|
| Thread-safe      | Không                 | Có (synchronized)      |
| Null key/value   | Cho phép              | Không cho phép         |
| Performance      | Nhanh hơn             | Chậm hơn (overhead)    |
| Iterator         | Fail-fast             | Fail-safe (Enumeration)|
| Kế thừa          | AbstractMap           | Dictionary (legacy)    |
| Sử dụng          | Khuyên dùng           | Legacy, không khuyên   |

> Nếu cần thread-safe, dùng `ConcurrentHashMap` thay `Hashtable`.

---

## Q32. What does `System.gc()` and `Runtime.gc()` methods do?

Cả hai đều **gợi ý** JVM thực hiện Garbage Collection, nhưng **không đảm bảo** GC sẽ chạy ngay.

```java
System.gc();                    // gọi Runtime.getRuntime().gc()
Runtime.getRuntime().gc();      // tương đương
```

**Thực tế:** JVM có thể bỏ qua request này. Trong production, không nên gọi thủ công vì:
- GC pause ảnh hưởng performance
- JVM biết tốt hơn khi nào nên GC
- Có thể gây "stop-the-world" không cần thiết
