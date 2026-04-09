# Q58: What are the Mono and Flux types?
> **Dịch:** Mono và Flux là gì?

## Tra loi ngan gon
> **Mono** va **Flux** la 2 kieu du lieu cot loi cua Project Reactor (thu vien reactive cua Spring WebFlux). **Mono** chua **0 hoac 1** phan tu. **Flux** chua **0 den N** phan tu.

## Cach nho
```
Mono = Hop qua 1 mon  (0 hoac 1 item)
Flux = Bang chuyen sushi (0 den N items, chay lien tuc)
```

## So sanh

| | Mono<T> | Flux<T> |
|--|:---:|:---:|
| So phan tu | 0 hoac 1 | 0 den N |
| Tuong tu | Optional, CompletableFuture | List, Stream |
| Use case | findById, save | findAll, search |

## Vi du Mono

```java
// Tao Mono
Mono<String> empty = Mono.empty();
Mono<String> mono = Mono.just("Hello");
Mono<User> user = Mono.fromCallable(() -> userRepo.findById(1));

// Chuoi xu ly
Mono<String> result = userRepo.findById("1")       // Mono<User>
    .map(User::getName)                             // Mono<String>
    .defaultIfEmpty("Unknown")                      // Gia tri mac dinh
    .doOnSuccess(name -> log.info("Found: " + name)); // Side effect

// Xu ly loi
Mono<User> safe = userRepo.findById("1")
    .switchIfEmpty(Mono.error(new NotFoundException()))
    .onErrorReturn(new User("default"));

// Ket hop 2 Mono
Mono<OrderDetail> detail = Mono.zip(
    userService.findById(userId),
    orderService.findById(orderId)
).map(tuple -> new OrderDetail(tuple.getT1(), tuple.getT2()));
```

## Vi du Flux

```java
// Tao Flux
Flux<Integer> numbers = Flux.just(1, 2, 3, 4, 5);
Flux<Integer> range = Flux.range(1, 10);
Flux<String> fromList = Flux.fromIterable(List.of("a", "b", "c"));

// Chuoi xu ly
Flux<UserDto> users = userRepo.findAll()          // Flux<User>
    .filter(u -> u.isActive())                    // Loc
    .map(u -> new UserDto(u.getName()))            // Chuyen doi
    .sort(Comparator.comparing(UserDto::getName))  // Sap xep
    .take(10);                                     // Lay 10 ket qua

// Streaming (Server-Sent Events)
@GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
public Flux<Long> stream() {
    return Flux.interval(Duration.ofSeconds(1)); // Gui 1 so moi giay
}

// Ket hop Mono + Flux
Flux<OrderDto> userOrders = userRepo.findById(userId)     // Mono<User>
    .flatMapMany(user -> orderRepo.findByUserId(user.getId())) // Flux<Order>
    .map(order -> new OrderDto(order));
```

## Operators quan trong

```java
// map: chuyen doi 1-1
mono.map(user -> user.getName());

// flatMap: chuyen doi 1-1 (async, tra ve Mono/Flux)
mono.flatMap(user -> orderService.findByUser(user.getId()));

// flatMapMany: Mono -> Flux
mono.flatMapMany(user -> orderRepo.findByUserId(user.getId()));

// zip: ket hop nhieu nguon
Mono.zip(monoA, monoB, monoC).map(tuple -> ...);

// switchIfEmpty: xu ly khi rong
mono.switchIfEmpty(Mono.error(new NotFoundException()));

// retry: thu lai khi loi
mono.retryWhen(Retry.backoff(3, Duration.ofSeconds(1)));

// timeout: gioi han thoi gian
mono.timeout(Duration.ofSeconds(5));
```

## Diem quan trong nho phong van
1. **Mono** = 0/1 phan tu, **Flux** = 0/N phan tu
2. Ca hai deu **lazy** - chi chay khi co **subscriber**
3. `flatMap` cho async, `map` cho sync transformation
4. `Mono.zip()` de goi **nhieu API song song**
5. Dung `.block()` khi can chuyen ve **blocking** (chi trong MVC context)
