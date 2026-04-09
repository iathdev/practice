# Q37: Is Spring 5 compatible with older versions of Java?
> **Dịch:** Spring 5 có tương thích với các phiên bản Java cũ không?

# Q38: How does Spring 5 integrate with JDK 9 modularity?
> **Dịch:** Spring 5 tích hợp với JDK 9 Modularity như thế nào?

## Q37: Spring 5 va Java versions

### Tra loi ngan gon
> **Khong.** Spring 5 yeu cau toi thieu **Java 8**. Khong ho tro Java 6 hay 7 nua.

### Bang tuong thich

| Spring version | Java toi thieu | Java khuyen dung |
|:-:|:-:|:-:|
| Spring 4.x | Java 6 | Java 7-8 |
| Spring 5.x | **Java 8** | Java 8-17 |
| Spring 6.x | **Java 17** | Java 17-21 |

### Tai sao can Java 8+?
```java
// Spring 5 su dung nhieu tinh nang Java 8:
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

## Q38: Spring 5 va JDK 9 Modularity

### Tra loi ngan gon
> Spring 5 ho tro **JDK 9 module system** (JPMS) bang cach cung cap **Automatic-Module-Name** trong moi jar. Tuy nhien, Spring **KHONG bat buoc** su dung module system.

### Cach tich hop
```java
// Moi Spring jar co Automatic-Module-Name trong MANIFEST.MF:
// spring-core        -> spring.core
// spring-context     -> spring.context
// spring-web         -> spring.web
// spring-webmvc      -> spring.webmvc

// module-info.java (neu ban dung JPMS)
module com.myapp {
    requires spring.core;
    requires spring.context;
    requires spring.web;
    opens com.myapp to spring.core; // cho phep reflection
}
```

### Thuc te
- Hau het project **KHONG** dung JPMS
- Spring van chay tot **khong can** module-info.java
- Chi can dam bao Java version tuong thich

## Diem quan trong nho phong van
1. Spring 5 = **Java 8+**, Spring 6 = **Java 17+**
2. Spring 5 **ho tro** JDK 9 modules nhung **khong bat buoc**
3. Moi Spring jar co **Automatic-Module-Name**
4. Tinh nang Java 8 (lambda, Optional, Stream) duoc dung rong rai trong Spring 5
