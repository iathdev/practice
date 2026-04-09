# Q34: What are the disadvantages of using Reactive Streams?
> **Dịch:** Nhược điểm của Reactive Streams là gì?

## Trả lời ngắn gọn
> Reactive Streams có nhiều nhược điểm: **debug khó, learning curve cao, code phức tạp hơn, ecosystem hạn chế** (JDBC blocking), và **không cải thiện performance** cho CPU-bound tasks.

## Cách nhớ
```
Reactive = Xe F1
  Pro: Nhanh khi đua trên đường đua (high concurrency)
  Con: Khó lái, khó sửa chữa, không hợp đi chợ (CRUD đơn giản)
```

## Các nhược điểm chính

### 1. Learning Curve CAO
```java
// Imperative (dễ hiểu)
User user = userRepo.findById(id);
if (user != null) {
    user.setName("Thai");
    userRepo.save(user);
}

// Reactive (khó hiểu hơn nhiều)
userRepo.findById(id)
    .flatMap(user -> {
        user.setName("Thai");
        return userRepo.save(user);
    })
    .switchIfEmpty(Mono.error(new NotFoundException()))
    .subscribe();
```

### 2. Debug RẤT KHÓ
```
// Stack trace của Imperative: rõ ràng
java.lang.NullPointerException
    at UserService.findUser(UserService.java:25)
    at UserController.getUser(UserController.java:15)

// Stack trace của Reactive: khó đọc
java.lang.NullPointerException
    at reactor.core.publisher.Mono.lambda$flatMap$1
    at reactor.core.publisher.FluxFlatMap$FlatMapMain.onNext
    at reactor.core.publisher.Operators$MonoSubscriber.complete
    // ... 50 dòng reactor internal code ...
```

### 3. Testing phức tạp hơn
```java
// Imperative test
@Test
void testFind() {
    User user = userService.findById(1L);
    assertEquals("Thai", user.getName());
}

// Reactive test - cần StepVerifier
@Test
void testFind() {
    StepVerifier.create(userService.findById("1"))
        .assertNext(user -> assertEquals("Thai", user.getName()))
        .verifyComplete();
}
```

### 4. Ecosystem hạn chế
```
- JDBC là BLOCKING -> phải dùng R2DBC (ít driver hơn)
- Nhiều thư viện chưa hỗ trợ reactive
- JPA/Hibernate không reactive native
- Transaction management phức tạp hơn
```

### 5. Không giúp gì cho CPU-bound
```
Reactive tốt cho:   I/O bound (gọi API, đọc DB, đọc file)
Reactive VÔ DỤNG cho: CPU bound (tính toán, mã hóa, nén file)
```

### 6. Code phức tạp, khó bảo trì
```java
// 1 logic đơn giản viết bằng reactive
return userRepo.findById(id)
    .flatMap(user -> orderRepo.findByUserId(user.getId())
        .collectList()
        .flatMap(orders -> {
            if (orders.isEmpty()) {
                return Mono.error(new NoOrdersException());
            }
            return paymentService.process(orders)
                .flatMap(payment -> notificationService.notify(user, payment));
        }))
    .onErrorResume(NoOrdersException.class, 
        e -> Mono.just(new EmptyResponse()))
    .timeout(Duration.ofSeconds(5))
    .retryWhen(Retry.backoff(3, Duration.ofSeconds(1)));

// Tương đương imperative:
User user = userRepo.findById(id);
List<Order> orders = orderRepo.findByUserId(user.getId());
if (orders.isEmpty()) return new EmptyResponse();
Payment payment = paymentService.process(orders);
notificationService.notify(user, payment);
```

## Điểm quan trọng nhớ phỏng vấn
1. **Debug khó** - stack trace dài và khó đọc
2. **Learning curve cao** - cần hiểu Mono, Flux, operator
3. **Ecosystem hạn chế** - JDBC blocking, nhiều lib chưa hỗ trợ
4. **Không giúp** CPU-bound tasks
5. Chỉ nên dùng khi thật sự cần **high concurrency** và **non-blocking I/O**
