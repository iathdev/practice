# Q58: What are the Mono and Flux types?
> **Dịch:** Mono và Flux là gì?

## Trả lời ngắn gọn
> **Mono** và **Flux** là 2 kiểu dữ liệu cốt lõi của Project Reactor (thư viện reactive của Spring WebFlux). **Mono** chứa **0 hoặc 1** phần tử. **Flux** chứa **0 đến N** phần tử.

## Cách nhớ
```
Mono = Hộp quà 1 món  (0 hoặc 1 item)
Flux = Băng chuyền sushi (0 đến N items, chạy liên tục)
```

## So sánh

| | Mono<T> | Flux<T> |
|--|:---:|:---:|
| Số phần tử | 0 hoặc 1 | 0 đến N |
| Tương tự | Optional, CompletableFuture | List, Stream |
| Use case | findById, save | findAll, search |

## Ví dụ Mono

```java
// Tạo Mono
Mono<String> empty = Mono.empty();
Mono<String> mono = Mono.just("Hello");
Mono<User> user = Mono.fromCallable(() -> userRepo.findById(1));

// Chuỗi xử lý
Mono<String> result = userRepo.findById("1")       // Mono<User>
    .map(User::getName)                             // Mono<String>
    .defaultIfEmpty("Unknown")                      // Giá trị mặc định
    .doOnSuccess(name -> log.info("Found: " + name)); // Side effect

// Xử lý lỗi
Mono<User> safe = userRepo.findById("1")
    .switchIfEmpty(Mono.error(new NotFoundException()))
    .onErrorReturn(new User("default"));

// Kết hợp 2 Mono
Mono<OrderDetail> detail = Mono.zip(
    userService.findById(userId),
    orderService.findById(orderId)
).map(tuple -> new OrderDetail(tuple.getT1(), tuple.getT2()));
```

## Ví dụ Flux

```java
// Tạo Flux
Flux<Integer> numbers = Flux.just(1, 2, 3, 4, 5);
Flux<Integer> range = Flux.range(1, 10);
Flux<String> fromList = Flux.fromIterable(List.of("a", "b", "c"));

// Chuỗi xử lý
Flux<UserDto> users = userRepo.findAll()          // Flux<User>
    .filter(u -> u.isActive())                    // Lọc
    .map(u -> new UserDto(u.getName()))            // Chuyển đổi
    .sort(Comparator.comparing(UserDto::getName))  // Sắp xếp
    .take(10);                                     // Lấy 10 kết quả

// Streaming (Server-Sent Events)
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<Long> stream() {
    return Flux.interval(Duration.ofSeconds(1)); // Gửi 1 số mỗi giây
}

// Kết hợp Mono + Flux
Flux<OrderDto> userOrders = userRepo.findById(userId)     // Mono<User>
    .flatMapMany(user -> orderRepo.findByUserId(user.getId())) // Flux<Order>
    .map(order -> new OrderDto(order));
```

## Operators quan trọng

```java
// map: chuyển đổi 1-1
mono.map(user -> user.getName());

// flatMap: chuyển đổi 1-1 (async, trả về Mono/Flux)
mono.flatMap(user -> orderService.findByUser(user.getId()));

// flatMapMany: Mono -> Flux
mono.flatMapMany(user -> orderRepo.findByUserId(user.getId()));

// zip: kết hợp nhiều nguồn
Mono.zip(monoA, monoB, monoC).map(tuple -> ...);

// switchIfEmpty: xử lý khi rỗng
mono.switchIfEmpty(Mono.error(new NotFoundException()));

// retry: thử lại khi lỗi
mono.retryWhen(Retry.backoff(3, Duration.ofSeconds(1)));

// timeout: giới hạn thời gian
mono.timeout(Duration.ofSeconds(5));
```

## Điểm quan trọng nhớ phỏng vấn
1. **Mono** = 0/1 phần tử, **Flux** = 0/N phần tử
2. Cả hai đều **lazy** - chỉ chạy khi có **subscriber**
3. `flatMap` cho async, `map` cho sync transformation
4. `Mono.zip()` để gọi **nhiều API song song**
5. Dùng `.block()` khi cần chuyển về **blocking** (chỉ trong MVC context)
