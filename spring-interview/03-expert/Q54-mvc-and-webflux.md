# Q54: Can we use both Web MVC and WebFlux in the same application?
> **Dịch:** Có thể dùng cả Web MVC và WebFlux trong cùng một ứng dụng không?

## Trả lời ngắn gọn
> **Không thể** dùng cả hai cùng lúc trong 1 application. Spring Boot sẽ chọn **1 trong 2** dựa trên classpath. Tuy nhiên, bạn có thể dùng **WebClient** (reactive HTTP client) trong ứng dụng MVC.

## Cách nhớ
```
MVC + WebFlux = 2 hệ thống lái xe khác nhau
  Không thể vừa lái số sàn vừa lái số tự động cùng lúc
  Nhưng có thể dùng GPS (WebClient) trên cả 2 loại xe
```

## Chi tiết

### Spring Boot tự động chọn
```
Có spring-webmvc       -> Chạy MVC (Tomcat)
Có spring-webflux      -> Chạy WebFlux (Netty)
Có CẢ HAI              -> Mặc định chọn MVC
```

### Dùng WebClient trong MVC app (HAY DÙNG)
```java
// application - Spring MVC (blocking)
@RestController
public class OrderController {

    private final WebClient webClient; // Reactive HTTP client

    public OrderController(WebClient.Builder builder) {
        this.webClient = builder.baseUrl("http://user-service").build();
    }

    @GetMapping("/orders/{id}")
    public OrderDto getOrder(@PathVariable Long id) {
        Order order = orderRepo.findById(id).orElseThrow();

        // Dùng WebClient trong MVC - gọi external API
        User user = webClient.get()
            .uri("/users/{userId}", order.getUserId())
            .retrieve()
            .bodyToMono(User.class)
            .block(); // block() vì đang trong MVC context

        return new OrderDto(order, user);
    }
}
```

### Microservices: Tách riêng
```
User Service    -> Spring MVC (CRUD đơn giản)
Gateway Service -> Spring WebFlux (routing, non-blocking)
Chat Service    -> Spring WebFlux (WebSocket, streaming)
```

## Điểm quan trọng nhớ phỏng vấn
1. **KHÔNG** dùng cả MVC và WebFlux trong 1 app
2. Có cả 2 dependency -> Spring Boot **chọn MVC**
3. Có thể dùng **WebClient** trong MVC app (thay RestTemplate)
4. Microservices: mỗi service **chọn riêng** MVC hoặc WebFlux
