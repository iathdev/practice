# Q26: What is the use of WebClient and WebTestClient?
> **Dịch:** WebClient và WebTestClient dùng để làm gì?

## Tra loi ngan gon
> **WebClient** la HTTP client **non-blocking** (reactive) thay the RestTemplate. **WebTestClient** la phien ban test cua WebClient, dung de test reactive web endpoints ma khong can server that.

## Cach nho
```
RestTemplate = Goi dien thoai (doi cho den khi co ket qua)
WebClient    = Gui tin nhan  (gui xong lam viec khac, bao khi co tra loi)
WebTestClient = Gui tin nhan thu (test khong can mang that)
```

## WebClient - HTTP Client

```java
// Tao WebClient
WebClient client = WebClient.builder()
    .baseUrl("https://api.example.com")
    .defaultHeader("Authorization", "Bearer xxx")
    .build();

// GET request - tra ve Mono (1 ket qua)
Mono<User> user = client.get()
    .uri("/users/{id}", 1)
    .retrieve()
    .bodyToMono(User.class);

// GET request - tra ve Flux (nhieu ket qua)
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

// Blocking (doi ket qua - khi can)
User result = client.get()
    .uri("/users/1")
    .retrieve()
    .bodyToMono(User.class)
    .block(); // Cho ket qua (dung trong non-reactive code)
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

## So sanh RestTemplate vs WebClient

| | RestTemplate | WebClient |
|--|:---:|:---:|
| Blocking | Co (mac dinh) | Khong (non-blocking) |
| Reactive | Khong | Co |
| Performance | Thap hon | Cao hon |
| Tuong lai | **Deprecated** | Khuyen dung |
| Spring 5+ | Van dung duoc | Moi va tot hon |

## Diem quan trong nho phong van
1. **WebClient** thay the **RestTemplate** (deprecated)
2. WebClient la **non-blocking**, dung `Mono`/`Flux`
3. **WebTestClient** test web endpoints khong can server that
4. Co the dung `.block()` de chuyen ve blocking khi can
