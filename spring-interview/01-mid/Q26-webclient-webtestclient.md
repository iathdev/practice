# Q26: What is the use of WebClient and WebTestClient?
> **Dịch:** WebClient và WebTestClient dùng để làm gì?

## Trả lời ngắn gọn
> **WebClient** là HTTP client **non-blocking** (reactive) thay thế RestTemplate. **WebTestClient** là phiên bản test của WebClient, dùng để test reactive web endpoints mà không cần server thật.

## Cách nhớ
```
RestTemplate = Gọi điện thoại (đợi cho đến khi có kết quả)
WebClient    = Gửi tin nhắn  (gửi xong làm việc khác, báo khi có trả lời)
WebTestClient = Gửi tin nhắn thử (test không cần mạng thật)
```

## WebClient - HTTP Client

```java
// Tạo WebClient
WebClient client = WebClient.builder()
    .baseUrl("https://api.example.com")
    .defaultHeader("Authorization", "Bearer xxx")
    .build();

// GET request - trả về Mono (1 kết quả)
Mono<User> user = client.get()
    .uri("/users/{id}", 1)
    .retrieve()
    .bodyToMono(User.class);

// GET request - trả về Flux (nhiều kết quả)
Flux<User> users = client.get()
    .uri("/users")
    .retrieve()
    .bodyToFlux(User.class);

// POST request
Mono<User> created = client.post()
    .uri("/users")
    .bodyValue(new User("Thai", "thai@email.com"))
    .retrieve()
    .bodyToMono(User.class);

// Blocking (đợi kết quả - khi cần)
User result = client.get()
    .uri("/users/1")
    .retrieve()
    .bodyToMono(User.class)
    .block(); // Chờ kết quả (dùng trong non-reactive code)
```

## WebTestClient - Test

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class UserControllerTest {

    @Autowired
    private WebTestClient webTestClient;

    @Test
    void shouldGetUser() {
        webTestClient.get()
            .uri("/api/users/1")
            .exchange()
            .expectStatus().isOk()
            .expectBody(User.class)
            .value(user -> {
                assertEquals("Thai", user.getName());
            });
    }

    @Test
    void shouldCreateUser() {
        webTestClient.post()
            .uri("/api/users")
            .bodyValue(new User("Thai", "thai@email.com"))
            .exchange()
            .expectStatus().isCreated()
            .expectBody()
            .jsonPath("$.name").isEqualTo("Thai");
    }
}
```

## So sánh RestTemplate vs WebClient

| | RestTemplate | WebClient |
|--|:---:|:---:|
| Blocking | Có (mặc định) | Không (non-blocking) |
| Reactive | Không | Có |
| Performance | Thấp hơn | Cao hơn |
| Tương lai | **Deprecated** | Khuyên dùng |
| Spring 5+ | Vẫn dùng được | Mới và tốt hơn |

## Điểm quan trọng nhớ phỏng vấn
1. **WebClient** thay thế **RestTemplate** (deprecated)
2. WebClient là **non-blocking**, dùng `Mono`/`Flux`
3. **WebTestClient** test web endpoints không cần server thật
4. Có thể dùng `.block()` để chuyển về blocking khi cần
