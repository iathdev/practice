# Q52: What is reactive programming?
> **Dịch:** Lập trình phản ứng (Reactive Programming) là gì?

# Q53: What are the advantages of Spring MVC over Struts MVC?
> **Dịch:** Ưu điểm của Spring MVC so với Struts MVC là gì?

---

## Q52: Reactive Programming

### Trả lời ngắn gọn
> Reactive programming là mô hình lập trình **bất đồng bộ** (asynchronous), xử lý **data streams**, với cơ chế **backpressure** (kiểm soát tốc độ). Dựa trên Reactive Streams spec với 4 interface: Publisher, Subscriber, Subscription, Processor.

### Cách nhớ
```
Imperative  = Đọc báo giấy  (đọc từng trang, đợi in xong mới đọc)
Reactive    = Đọc tin tức online (tin mới tự động hiện, cuộn lên xem)
Backpressure = Nói "chậm lại!" khi tin nhiều quá
```

### 4 nguyên tắc của Reactive
```
1. RESPONSIVE   - Phản hồi nhanh
2. RESILIENT    - Chịu lỗi tốt
3. ELASTIC      - Co giãn (scale)
4. MESSAGE-DRIVEN - Giao tiếp qua message (async)
```

### Ví dụ với Project Reactor
```java
// Mono: 0 hoặc 1 kết quả
Mono<User> user = userRepo.findById("1");

// Flux: 0 đến N kết quả
Flux<User> users = userRepo.findAll();

// Chuỗi xử lý (pipeline)
Flux<String> names = userRepo.findAll()
    .filter(u -> u.getAge() > 18)        // lọc
    .map(User::getName)                   // chuyển đổi
    .sort()                               // sắp xếp
    .take(10);                            // lấy 10 kết quả đầu

// Kết hợp nhiều nguồn dữ liệu
Mono<OrderSummary> summary = Mono.zip(
    userService.findById(userId),
    orderService.findByUser(userId),
    paymentService.getBalance(userId)
).map(tuple -> new OrderSummary(tuple.getT1(), tuple.getT2(), tuple.getT3()));
```

---

## Q53: Spring MVC vs Struts MVC

### Trả lời ngắn gọn
> Spring MVC vượt trội Struts MVC ở nhiều mặt: **linh hoạt hơn, tích hợp tốt hơn, dễ test, không ràng buộc kế thừa class, hỗ trợ annotation**.

### So sánh

| | Spring MVC | Struts MVC |
|--|:---:|:---:|
| Controller | POJO (@Controller) | Phải extend ActionClass |
| Config | Annotation (ít XML) | Nhiều XML |
| Tích hợp | Tích hợp Spring ecosystem | Cần plugin |
| Test | Dễ (POJO + DI) | Khó (phụ thuộc Servlet) |
| View | Nhiều lựa chọn (Thymeleaf, JSP, JSON) | Chủ yếu JSP |
| Community | Rất lớn, active | Hầu như ngừng phát triển |
| Validation | Bean Validation tích hợp | XML-based |
| REST | @RestController | Khó khăn |
| Thread safety | Singleton bean (stateless) | Mỗi request 1 Action |

### Tại sao Spring MVC thắng?
```
1. Không cần kế thừa class cụ thể (POJO)
2. Annotation-driven (ít XML)
3. Tích hợp với Spring ecosystem (Security, Data, Boot)
4. Dễ test (MockMvc, DI)
5. Hỗ trợ REST native (@RestController)
6. Community lớn, cập nhật liên tục
```

## Điểm quan trọng nhớ phỏng vấn
1. Reactive = **async, non-blocking, backpressure** (Mono/Flux)
2. Spring MVC > Struts: **POJO, annotation, tích hợp, dễ test**
3. Struts đã **lỗi thời**, Spring MVC là **chuẩn công nghiệp**
4. Reactive chỉ nên dùng khi cần **high concurrency / streaming**
