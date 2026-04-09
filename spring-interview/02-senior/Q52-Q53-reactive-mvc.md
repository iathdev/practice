# Q52: What is reactive programming?
> **Dịch:** Lập trình phản ứng (Reactive Programming) là gì?

# Q53: What are the advantages of Spring MVC over Struts MVC?
> **Dịch:** Ưu điểm của Spring MVC so với Struts MVC là gì?

---

## Q52: Reactive Programming

### Tra loi ngan gon
> Reactive programming la mo hinh lap trinh **bat dong bo** (asynchronous), xu ly **data streams**, voi co che **backpressure** (kiem soat toc do). Dua tren Reactive Streams spec voi 4 interface: Publisher, Subscriber, Subscription, Processor.

### Cach nho
```
Imperative  = Doc bao giay  (doc tung trang, doi in xong moi doc)
Reactive    = Doc tin tuc online (tin moi tu dong hien, cuon len xem)
Backpressure = Noi "cham lai!" khi tin nhieu qua
```

### 4 nguyen tac cua Reactive
```
1. RESPONSIVE   - Phan hoi nhanh
2. RESILIENT    - Chiu loi tot
3. ELASTIC      - Co gian (scale)
4. MESSAGE-DRIVEN - Giao tiep qua message (async)
```

### Vi du voi Project Reactor
```java
// Mono: 0 hoac 1 ket qua
Mono<User> user = userRepo.findById("1");

// Flux: 0 den N ket qua
Flux<User> users = userRepo.findAll();

// Chuoi xu ly (pipeline)
Flux<String> names = userRepo.findAll()
    .filter(u -> u.getAge() > 18)        // loc
    .map(User::getName)                   // chuyen doi
    .sort()                               // sap xep
    .take(10);                            // lay 10 ket qua dau

// Ket hop nhieu nguon du lieu
Mono<OrderSummary> summary = Mono.zip(
    userService.findById(userId),
    orderService.findByUser(userId),
    paymentService.getBalance(userId)
).map(tuple -> new OrderSummary(tuple.getT1(), tuple.getT2(), tuple.getT3()));
```

---

## Q53: Spring MVC vs Struts MVC

### Tra loi ngan gon
> Spring MVC vuot troi Struts MVC o nhieu mat: **linh hoat hon, tich hop tot hon, de test, khong rang buoc ke thua class, ho tro annotation**.

### So sanh

| | Spring MVC | Struts MVC |
|--|:---:|:---:|
| Controller | POJO (@Controller) | Phai extend ActionClass |
| Config | Annotation (ít XML) | Nhieu XML |
| Tich hop | Tich hop Spring ecosystem | Can plugin |
| Test | De (POJO + DI) | Kho (phu thuoc Servlet) |
| View | Nhieu lua chon (Thymeleaf, JSP, JSON) | Chu yeu JSP |
| Community | Rat lon, active | Hau nhu ngung phat trien |
| Validation | Bean Validation tich hop | XML-based |
| REST | @RestController | Kho khan |
| Thread safety | Singleton bean (stateless) | Moi request 1 Action |

### Tai sao Spring MVC thang?
```
1. Khong can ke thua class cu the (POJO)
2. Annotation-driven (it XML)
3. Tich hop voi Spring ecosystem (Security, Data, Boot)
4. De test (MockMvc, DI)
5. Ho tro REST native (@RestController)
6. Community lon, cap nhat lien tuc
```

## Diem quan trong nho phong van
1. Reactive = **async, non-blocking, backpressure** (Mono/Flux)
2. Spring MVC > Struts: **POJO, annotation, tich hop, de test**
3. Struts da **loi thoi**, Spring MVC la **chuan cong nghiep**
4. Reactive chi nen dung khi can **high concurrency / streaming**
