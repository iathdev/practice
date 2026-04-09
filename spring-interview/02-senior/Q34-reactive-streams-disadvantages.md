# Q34: What are the disadvantages of using Reactive Streams?
> **Dịch:** Nhược điểm của Reactive Streams là gì?

## Tra loi ngan gon
> Reactive Streams co nhieu nhuoc diem: **debug kho, learning curve cao, code phuc tap hon, ecosystem han che** (JDBC blocking), va **khong cai thien performance** cho CPU-bound tasks.

## Cach nho
```
Reactive = Xe F1
  Pro: Nhanh khi dua tren duong dua (high concurrency)
  Con: Kho lai, kho sua chua, khong hop di cho (CRUD don gian)
```

## Cac nhuoc diem chinh

### 1. Learning Curve CAO
```java
// Imperative (de hieu)
User user = userRepo.findById(id);
if (user != null) {
    user.setName("Thai");
    userRepo.save(user);
}

// Reactive (kho hieu hon nhieu)
userRepo.findById(id)
    .flatMap(user -> {
        user.setName("Thai");
        return userRepo.save(user);
    })
    .switchIfEmpty(Mono.error(new NotFoundException()))
    .subscribe();
```

### 2. Debug RAT KHO
```
// Stack trace cua Imperative: ro rang
java.lang.NullPointerException
    at UserService.findUser(UserService.java:25)
    at UserController.getUser(UserController.java:15)

// Stack trace cua Reactive: kho doc
java.lang.NullPointerException
    at reactor.core.publisher.Mono.lambda$flatMap$1
    at reactor.core.publisher.FluxFlatMap$FlatMapMain.onNext
    at reactor.core.publisher.Operators$MonoSubscriber.complete
    // ... 50 dong reactor internal code ...
```

### 3. Testing phuc tap hon
```java
// Imperative test
@Test
void testFind() {
    User user = userService.findById(1L);
    assertEquals("Thai", user.getName());
}

// Reactive test - can StepVerifier
@Test
void testFind() {
    StepVerifier.create(userService.findById("1"))
        .assertNext(user -> assertEquals("Thai", user.getName()))
        .verifyComplete();
}
```

### 4. Ecosystem han che
```
- JDBC la BLOCKING -> phai dung R2DBC (it driver hon)
- Nhieu thu vien chua ho tro reactive
- JPA/Hibernate khong reactive native
- Transaction management phuc tap hon
```

### 5. Khong giup gi cho CPU-bound
```
Reactive tot cho:   I/O bound (goi API, doc DB, doc file)
Reactive VO DUNG cho: CPU bound (tinh toan, ma hoa, nen file)
```

### 6. Code phuc tap, kho bao tri
```java
// 1 logic don gian viet bang reactive
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

// Tuong duong imperative:
User user = userRepo.findById(id);
List<Order> orders = orderRepo.findByUserId(user.getId());
if (orders.isEmpty()) return new EmptyResponse();
Payment payment = paymentService.process(orders);
notificationService.notify(user, payment);
```

## Diem quan trong nho phong van
1. **Debug kho** - stack trace dai va kho doc
2. **Learning curve cao** - can hieu Mono, Flux, operator
3. **Ecosystem han che** - JDBC blocking, nhieu lib chua ho tro
4. **Khong giup** CPU-bound tasks
5. Chi nen dung khi that su can **high concurrency** va **non-blocking I/O**
