# Q54: Can we use both Web MVC and WebFlux in the same application?
> **Dịch:** Có thể dùng cả Web MVC và WebFlux trong cùng một ứng dụng không?

## Tra loi ngan gon
> **Khong the** dung ca hai cung luc trong 1 application. Spring Boot se chon **1 trong 2** dua tren classpath. Tuy nhien, ban co the dung **WebClient** (reactive HTTP client) trong ung dung MVC.

## Cach nho
```
MVC + WebFlux = 2 he thong lai xe khac nhau
  Khong the vua lai so san vua lai so tu dong cung luc
  Nhung co the dung GPS (WebClient) tren ca 2 loai xe
```

## Chi tiet

### Spring Boot tu dong chon
```
Co spring-webmvc       -> Chay MVC (Tomcat)
Co spring-webflux      -> Chay WebFlux (Netty)
Co CA HAI              -> Mac dinh chon MVC
```

### Dung WebClient trong MVC app (HAY DUNG)
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

        // Dung WebClient trong MVC - goi external API
        User user = webClient.get()
            .uri("/users/{userId}", order.getUserId())
            .retrieve()
            .bodyToMono(User.class)
            .block(); // block() vi dang trong MVC context

        return new OrderDto(order, user);
    }
}
```

### Microservices: Tach rieng
```
User Service    -> Spring MVC (CRUD don gian)
Gateway Service -> Spring WebFlux (routing, non-blocking)
Chat Service    -> Spring WebFlux (WebSocket, streaming)
```

## Diem quan trong nho phong van
1. **KHONG** dung ca MVC va WebFlux trong 1 app
2. Co ca 2 dependency -> Spring Boot **chon MVC**
3. Co the dung **WebClient** trong MVC app (thay RestTemplate)
4. Microservices: moi service **chon rieng** MVC hoac WebFlux
