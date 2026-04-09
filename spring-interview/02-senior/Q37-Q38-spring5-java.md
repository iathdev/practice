# Q37: Is Spring 5 compatible with older versions of Java?
> **Dịch:** Spring 5 có tương thích với các phiên bản Java cũ không?

# Q38: How does Spring 5 integrate with JDK 9 modularity?
> **Dịch:** Spring 5 tích hợp với JDK 9 Modularity như thế nào?

## Q37: Spring 5 và Java versions

### Trả lời ngắn gọn
> **Không.** Spring 5 yêu cầu tối thiểu **Java 8**. Không hỗ trợ Java 6 hay 7 nữa.

### Bảng tương thích

| Spring version | Java tối thiểu | Java khuyên dùng |
|:-:|:-:|:-:|
| Spring 4.x | Java 6 | Java 7-8 |
| Spring 5.x | **Java 8** | Java 8-17 |
| Spring 6.x | **Java 17** | Java 17-21 |

### Tại sao cần Java 8+?
```java
// Spring 5 sử dụng nhiều tính năng Java 8:
// 1. Lambda
bean.ifPresent(b -> process(b));

// 2. Optional
public Optional<User> findById(Long id);

// 3. CompletableFuture
@Async
public CompletableFuture<User> findAsync(Long id);

// 4. Date/Time API
@DateTimeFormat(pattern = "yyyy-MM-dd")
private LocalDate birthday;

// 5. Default methods trong interface
```

---

## Q38: Spring 5 và JDK 9 Modularity

### Trả lời ngắn gọn
> Spring 5 hỗ trợ **JDK 9 module system** (JPMS) bằng cách cung cấp **Automatic-Module-Name** trong mỗi jar. Tuy nhiên, Spring **KHÔNG bắt buộc** sử dụng module system.

### Cách tích hợp
```java
// Mỗi Spring jar có Automatic-Module-Name trong MANIFEST.MF:
// spring-core        -> spring.core
// spring-context     -> spring.context
// spring-web         -> spring.web
// spring-webmvc      -> spring.webmvc

// module-info.java (nếu bạn dùng JPMS)
module com.myapp {
    requires spring.core;
    requires spring.context;
    requires spring.web;
    opens com.myapp to spring.core; // cho phép reflection
}
```

### Thực tế
- Hầu hết project **KHÔNG** dùng JPMS
- Spring vẫn chạy tốt **không cần** module-info.java
- Chỉ cần đảm bảo Java version tương thích

## Điểm quan trọng nhớ phỏng vấn
1. Spring 5 = **Java 8+**, Spring 6 = **Java 17+**
2. Spring 5 **hỗ trợ** JDK 9 modules nhưng **không bắt buộc**
3. Mỗi Spring jar có **Automatic-Module-Name**
4. Tính năng Java 8 (lambda, Optional, Stream) được dùng rộng rãi trong Spring 5
