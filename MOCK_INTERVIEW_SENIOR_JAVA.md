# Mock Interview Session

**Senior Java Backend Developer - 60 phút**
Ứng viên: Nguyễn Phú Quân | 5+ năm kinh nghiệm

---

## Thông tin buổi phỏng vấn

- **Vị trí:** Senior Java Backend Developer
- **Công ty:** Tech Company (Fintech/E-commerce)
- **Interviewer:** Technical Lead + Engineering Manager
- **Thời lượng:** 60 phút
- **Format:** Technical (40%) + System Design (30%) + Behavioral (30%)

---

## PART 1: Giới thiệu bản thân (5 phút)

**Interviewer (Tech Lead):** Chào Quân, cảm ơn bạn đã tham gia buổi phỏng vấn hôm nay. Trước tiên, bạn có thể giới thiệu về bản thân và highlight những kinh nghiệm relevant nhất với vị trí Senior Java Backend Developer không?

**Candidate (Quân):** Dạ chào anh/chị, em là Nguyễn Phú Quân, hiện có hơn 5 năm kinh nghiệm trong lĩnh vực Web Development, chủ yếu tập trung vào Backend với Java và Spring Boot.

**Hiện tại em đang làm tại GHTK** - một trong những công ty logistics lớn nhất Việt Nam. Em tham gia phát triển **GAPP Platform** - nền tảng phục vụ đội ngũ shipper với hàng triệu users. Tech stack chính là:

- Java Spring Boot cho microservices
- Kafka cho message queue và event-driven architecture
- Kubernetes cho orchestration
- Redis, MySQL cho caching và database

**Trước đó em làm tại VCCORP** - phát triển AI Chatbot platform với NLP, và cũng có kinh nghiệm với React Native, Flutter cho mobile development.

Điểm mạnh của em là khả năng làm việc với hệ thống high-traffic, microservices architecture, và kinh nghiệm end-to-end từ backend đến mobile.

> **Tips giới thiệu bản thân:**
> - Giữ trong 2-3 phút, không quá dài
> - Highlight kinh nghiệm relevant với vị trí
> - Đề cập số liệu cụ thể (millions users, high traffic)
> - Kết thúc bằng điểm mạnh của bạn

---

## PART 2: Technical Deep Dive (25 phút)

### Q1: HashMap vs ConcurrentHashMap

**Interviewer:** Bạn có thể giải thích sự khác biệt giữa `HashMap` và `ConcurrentHashMap` không? Khi nào nên dùng cái nào trong microservices?

**Candidate (Quân):**

**HashMap:**
- Không thread-safe
- Cho phép 1 null key và nhiều null values
- Performance tốt hơn trong single-threaded environment

**ConcurrentHashMap:**
- Thread-safe với segment-level locking (Java 7) hoặc CAS operations (Java 8+)
- Không cho phép null key/value
- Cho phép concurrent reads và writes mà không lock toàn bộ map

**Trong microservices tại GHTK:**
- Em dùng `ConcurrentHashMap` cho local cache trong memory khi cần thread-safety
- Tuy nhiên, với distributed systems, em prefer dùng **Redis** cho shared cache giữa các instances
- `HashMap` vẫn OK nếu bean là stateless hoặc data là request-scoped

```java
// Thread-safe cache with ConcurrentHashMap
private final Map<String, UserCache> cache = new ConcurrentHashMap<>();

// Atomic operations
cache.computeIfAbsent(userId, id -> fetchFromDB(id));
cache.putIfAbsent(userId, newUser);
```

---

### Q2: @Transactional trong Spring Boot

**Interviewer:** Trong Spring Boot, giải thích về `@Transactional`. Bạn đã gặp những issues gì với transactions trong thực tế?

**Candidate (Quân):**

`@Transactional` dùng AOP proxy để wrap method trong transaction boundary.

**Các issues em đã gặp:**

**1. Self-invocation problem:**

```java
@Service
public class OrderService {

    public void processOrder(Order order) {
        this.saveOrder(order); // Transaction không work!
    }

    @Transactional
    public void saveOrder(Order order) {
        orderRepository.save(order);
    }
}
```

Lý do: Gọi internal method bypass proxy. Giải pháp: Inject self hoặc tách ra service khác.

**2. Checked Exception không rollback:**

```java
@Transactional // Default chỉ rollback RuntimeException
public void process() throws IOException {
    // IOException sẽ KHÔNG rollback!
}

// Fix:
@Transactional(rollbackFor = Exception.class)
```

**3. Long transaction với Kafka:**

Tại GHTK, em gặp case transaction quá dài khi publish Kafka message trong transaction. Giải pháp: Dùng **Transactional Outbox Pattern** - save event vào DB trước, rồi có background job publish lên Kafka.

---

### Q3: Kafka - Exactly-once Semantics và Message Ordering

**Interviewer:** Bạn đề cập đến Kafka ở GHTK. Giải thích cách bạn đảm bảo **exactly-once semantics** và xử lý **message ordering**?

**Candidate (Quân):**

**Exactly-once semantics:**

Thực tế rất khó đạt true exactly-once. Em implement **at-least-once + idempotency**:

```java
// 1. Idempotent Consumer với unique message ID
@KafkaListener(topics = "order-events")
public void handleOrder(OrderEvent event) {
    String messageId = event.getMessageId();

    // Check đã xử lý chưa (Redis hoặc DB)
    if (processedMessages.contains(messageId)) {
        log.info("Duplicate message, skipping: {}", messageId);
        return;
    }

    // Process và mark as processed
    processOrder(event);
    processedMessages.add(messageId, TTL_24H);
}
```

**Message Ordering:**
- Kafka đảm bảo ordering **trong cùng partition**
- Em dùng **partition key** = orderId hoặc userId để các events của cùng entity vào cùng partition

```java
// Producer: Partition by orderId
kafkaTemplate.send("order-events", order.getId(), event);

// Tất cả events của order-123 sẽ vào cùng partition
// → Đảm bảo thứ tự: CREATED → CONFIRMED → SHIPPED
```

**Lưu ý:** Nếu tăng partitions, cần rebalance và có thể mất ordering tạm thời. Em design system để tolerate short-term reordering.

---

> **Key Points được đánh giá:**
> - Hiểu deep về Java concurrency
> - Kinh nghiệm thực tế với Spring Transaction pitfalls
> - Kafka best practices trong production
> - Có solution cho real-world problems

---

### Q4: MySQL Query Optimization

**Interviewer:** Với MySQL, làm thế nào bạn optimize query cho table có hàng triệu records? Giải thích về indexing strategy.

**Candidate (Quân):**

**Quy trình optimize của em:**

**1. Analyze với EXPLAIN:**

```sql
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = 123
  AND status = 'PENDING'
  AND created_at > '2024-01-01'
ORDER BY created_at DESC
LIMIT 20;
```

**2. Composite Index theo query pattern:**

```sql
-- Covering index cho query trên
CREATE INDEX idx_orders_user_status_created
ON orders(user_id, status, created_at DESC);

-- Lưu ý thứ tự columns theo:
-- 1. Equality conditions (user_id, status)
-- 2. Range/Sort conditions (created_at)
```

**3. Pagination với Cursor thay vì OFFSET:**

```sql
-- Slow với large offset
SELECT * FROM orders LIMIT 1000000, 20;

-- Cursor-based pagination
SELECT * FROM orders
WHERE id > :lastSeenId
ORDER BY id
LIMIT 20;
```

**4. Partitioning cho historical data:**

Tại GHTK, bảng orders được partition theo tháng. Query chỉ scan partition cần thiết.

**5. Read replicas:** Tách read queries sang replica, giảm load cho master.

---

### Q5: Redis Cache Consistency và Cache Stampede

**Interviewer:** Bạn dùng Redis tại GHTK. Làm sao đảm bảo **cache consistency** với database? Xử lý **cache stampede** như thế nào?

**Candidate (Quân):**

**Cache Consistency Strategies:**

**1. Cache-Aside (em dùng chủ yếu):**

```java
public User getUser(Long id) {
    // 1. Check cache
    User user = redisTemplate.opsForValue().get("user:" + id);
    if (user != null) return user;

    // 2. Cache miss → query DB
    user = userRepository.findById(id).orElseThrow();

    // 3. Populate cache với TTL
    redisTemplate.opsForValue().set("user:" + id, user, Duration.ofMinutes(30));
    return user;
}

@Transactional
public void updateUser(User user) {
    userRepository.save(user);
    // Invalidate cache sau khi write
    redisTemplate.delete("user:" + user.getId());
}
```

**2. Xử lý Cache Stampede:**

```java
// Distributed lock với Redisson
public User getUserWithLock(Long id) {
    String key = "user:" + id;
    User user = redisTemplate.opsForValue().get(key);
    if (user != null) return user;

    RLock lock = redisson.getLock("lock:" + key);
    try {
        if (lock.tryLock(5, 10, TimeUnit.SECONDS)) {
            // Double-check after acquiring lock
            user = redisTemplate.opsForValue().get(key);
            if (user != null) return user;

            user = userRepository.findById(id).orElseThrow();
            redisTemplate.opsForValue().set(key, user, Duration.ofMinutes(30));
        }
    } finally {
        lock.unlock();
    }
    return user;
}
```

**3. Probabilistic Early Expiration:** Refresh cache trước khi hết hạn để tránh stampede.

---

## PART 3: System Design (15 phút)

### Q6: Thiết kế Order Management System

**Interviewer:** Thiết kế hệ thống **Order Management** cho một e-commerce platform với 10 triệu orders/ngày. Focus vào high availability và consistency.

**Candidate (Quân):**

**Requirements clarification:**
- 10M orders/day ≈ ~115 orders/second (peak có thể 500-1000 TPS)
- Order lifecycle: Created → Paid → Shipped → Delivered
- Strong consistency cho payment, eventual consistency cho notifications

**High-Level Architecture:**

```
┌─────────────┐     ┌──────────────┐     ┌─────────────────┐
│   Clients   │────▶│ API Gateway  │────▶│  Load Balancer  │
└─────────────┘     │  (Kong/AWS)  │     └─────────────────┘
                    └──────────────┘              │
                                          ┌──────┴──────┐
                                          ▼             ▼
                                   ┌───────────┐ ┌───────────┐
                                   │  Order    │ │  Order    │
                                   │ Service 1 │ │ Service N │
                                   └─────┬─────┘ └─────┬─────┘
                                         │             │
                    ┌────────────────────┼─────────────┼────────────────┐
                    ▼                    ▼             ▼                ▼
              ┌──────────┐        ┌──────────┐  ┌──────────┐     ┌──────────┐
              │  Redis   │        │  Kafka   │  │  MySQL   │     │  MySQL   │
              │ (Cache)  │        │ Cluster  │  │  Master  │────▶│ Replicas │
              └──────────┘        └──────────┘  └──────────┘     └──────────┘
```

**Key Design Decisions:**

**1. Database Sharding by user_id:**

```
// Shard key = user_id % num_shards
// User's orders luôn ở cùng shard → query hiệu quả
// Trade-off: Cross-shard queries khó hơn
```

**2. Event-Driven với Kafka:**

```
Order Created → Kafka → Payment Service
                     → Inventory Service
                     → Notification Service
```

**3. Distributed Transaction với Saga Pattern:**

```
// Choreography-based Saga
1. Order Service: Create order (PENDING)
2. Payment Service: Process payment
   - Success → Publish PaymentCompleted
   - Fail → Publish PaymentFailed
3. Inventory Service: Reserve stock
4. Order Service: Update status (CONFIRMED/CANCELLED)
```

**4. Idempotency cho API:**

```
POST /orders
Headers: Idempotency-Key: abc-123

// Server check key trong Redis trước khi create
```

> **Follow-up questions có thể hỏi:**
> - Làm sao handle failure khi một service trong saga fail?
> - Database replication lag ảnh hưởng thế nào?
> - Monitoring và alerting strategy?

---

## PART 4: Kinh nghiệm dự án (10 phút)

### Q7: Challenge lớn nhất tại GHTK

**Interviewer:** Kể về một challenge lớn nhất bạn gặp tại GHTK và cách bạn giải quyết?

**Candidate (Quân):**

**Challenge:** Hệ thống tracking đơn hàng real-time cho shipper gặp bottleneck khi scale đến 500K active shippers đồng thời.

**Symptoms:**
- API response time tăng từ 100ms lên 2-3 seconds
- Database CPU 90%+
- Timeout errors tăng vọt vào giờ cao điểm

**Root Cause Analysis:**
- N+1 query problem khi load shipper + orders + locations
- Không có caching layer
- Single database instance

**Solution em implement:**
1. **Query Optimization:** Refactor thành JOIN queries, thêm composite indexes
2. **Redis Caching:** Cache shipper data với TTL 5 phút, location data với TTL 30 giây
3. **Read Replicas:** Route read queries sang 2 replicas
4. **Connection Pooling:** Tune HikariCP parameters

**Results:**
- Response time giảm xuống <100ms
- Database CPU giảm xuống 30%
- System handle được 1M+ concurrent users

---

### Q8: Authentication/Authorization với Keycloak

**Interviewer:** Bạn có kinh nghiệm với Keycloak. Explain cách bạn implement authentication/authorization trong microservices?

**Candidate (Quân):**

**Architecture với Keycloak:**

```
┌─────────┐    ┌──────────┐    ┌─────────────┐
│ Client  │───▶│ Keycloak │───▶│ JWT Token   │
└─────────┘    └──────────┘    └─────────────┘
                                      │
                    ┌─────────────────┼─────────────────┐
                    ▼                 ▼                 ▼
              ┌──────────┐     ┌──────────┐      ┌──────────┐
              │ Service A│     │ Service B│      │ Service C│
              │(validate │     │(validate │      │(validate │
              │  JWT)    │     │  JWT)    │      │  JWT)    │
              └──────────┘     └──────────┘      └──────────┘
```

**Implementation:**

```java
// Spring Security config với Keycloak
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) {
        return http
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(
                    jwtAuthConverter())))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/admin/**")
                    .hasRole("ADMIN")
                .requestMatchers("/api/**")
                    .authenticated())
            .build();
    }
}

// Extract roles from Keycloak JWT
private JwtAuthenticationConverter jwtAuthConverter() {
    JwtGrantedAuthoritiesConverter converter =
        new JwtGrantedAuthoritiesConverter();
    converter.setAuthoritiesClaimName("realm_access.roles");
    converter.setAuthorityPrefix("ROLE_");

    JwtAuthenticationConverter jwtConverter =
        new JwtAuthenticationConverter();
    jwtConverter.setJwtGrantedAuthoritiesConverter(converter);
    return jwtConverter;
}
```

**Service-to-Service Auth:** Dùng client credentials flow với service account.

---

## PART 5: Behavioral Questions (5 phút)

### Q9: Xử lý bất đồng với technical decision

**Interviewer:** Khi bạn không đồng ý với technical decision của senior/lead, bạn xử lý như thế nào?

**Candidate (Quân):**

Em tiếp cận theo framework sau:

**1. Seek to understand first:**

Hỏi để hiểu context và constraints mà em có thể chưa biết. Có thể họ có thông tin em không có.

**2. Data-driven discussion:**

Nếu vẫn không đồng ý, em chuẩn bị data/evidence cụ thể. Ví dụ: benchmark results, case studies, hoặc proof-of-concept.

**3. Propose alternatives:**

Không chỉ critique, mà propose solution thay thế với pros/cons rõ ràng.

**4. Disagree and commit:**

Nếu cuối cùng decision khác với ý kiến em, em vẫn commit 100% để thực hiện. Có thể document concerns để revisit sau.

**Ví dụ thực tế:** Tại GHTK, em từng không đồng ý với choice của lead về một message queue solution. Em đã làm POC so sánh performance, present cho team, và cuối cùng team đã adopt em's recommendation.

---

### Q10: Career development và expectations

**Interviewer:** Bạn expect gì ở vị trí Senior Developer? 3-5 năm tới bạn muốn phát triển như thế nào?

**Candidate (Quân):**

**Expectation ở vị trí Senior:**
- Ownership end-to-end cho features/systems
- Tham gia technical decisions và architecture
- Mentor junior developers
- Opportunity để work với challenging problems tại scale

**Career direction trong 3-5 năm:**

Em muốn phát triển theo hướng **Technical Leadership**:
- **Short-term (1-2 năm):** Deep expertise trong distributed systems, become go-to person cho complex technical problems
- **Medium-term (3-5 năm):** Tech Lead hoặc Staff Engineer role - drive architecture decisions, establish engineering practices
- Em cũng interested trong system design và scalability challenges

Em tin rằng [Company name] với scale và technical challenges sẽ là môi trường tốt để em grow theo hướng này.

---

## PART 6: Câu hỏi từ ứng viên (5 phút)

**Interviewer:** Bạn có câu hỏi gì cho chúng tôi không?

**Candidate (Quân):**

Em có một số câu hỏi:

1. **Tech stack & Architecture:** Team đang dùng tech stack gì? Có challenges nào đặc biệt về scalability không?
2. **Team structure:** Team có bao nhiêu người? Workflow development như thế nào?
3. **Growth opportunities:** Company có support learning như thế nào? Conference, training budget?
4. **Engineering culture:** Anh/chị có thể share về engineering culture và what makes a successful engineer here?
5. **Next steps:** Timeline cho hiring process như thế nào?

> **Good questions to ask:**
> - Questions về technical challenges và growth
> - Questions về team dynamics
> - Questions về engineering culture
> - Avoid: salary questions trong technical round

---

## Interview Assessment Summary

### Đánh giá tổng quan

| Criteria | Score | Notes |
|---|---|---|
| Java Core Knowledge | Strong | Good understanding of concurrency, collections |
| Spring Framework | Strong | Deep knowledge of transactions, real-world experience |
| Distributed Systems | Strong | Kafka, Redis, microservices patterns |
| Database | Strong | MySQL optimization, indexing strategies |
| System Design | Good | Can design at scale, knows trade-offs |
| Communication | Strong | Clear, structured answers |
| Cultural Fit | Strong | Good attitude, growth mindset |

> **Key Takeaways để chuẩn bị:**
> - Review Java concurrency và collections internal working
> - Chuẩn bị STAR stories cho behavioral questions
> - Practice system design với whiteboard
> - Chuẩn bị questions to ask interviewer
> - Review projects và có thể deep dive vào technical details

---

*Mock Interview - Senior Java Backend Developer*
*Chúc Quân phỏng vấn thành công!*
