# Kafka Real-World Problems — Cases lỗi & Solutions

---

## Case 1: Consumer Lag — Consumer xử lý chậm hơn Producer

### Triệu chứng

```
Topic: order-events (3 partitions)
Producer rate:  10,000 msg/s
Consumer rate:   2,000 msg/s
→ Lag tăng 8,000 msg/s → sau 1 giờ lag = 28.8 triệu messages!
```

```bash
# Check consumer lag
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group order-service

# Output:
# TOPIC        PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
# order-events 0          1000            50000           49000  ← lag lớn!
# order-events 1          1200            48000           46800
# order-events 2          900             51000           50100
```

### Nguyên nhân thường gặp

1. **Xử lý message quá chậm** (DB query, external API call)
2. **Ít partitions + ít consumers** (bottleneck parallelism)
3. **Consumer bị block** (deadlock, GC pause)
4. **max.poll.records quá lớn** → xử lý lâu → session timeout → rebalance → lag thêm

### Solutions

```java
// Solution 1: Tăng parallelism
// Tăng partitions (chỉ tăng, không giảm được!)
kafka-topics.sh --alter --topic order-events --partitions 12 \
    --bootstrap-server localhost:9092
// Rồi tăng consumers trong group lên 12

// Solution 2: Giảm max.poll.records + tăng session.timeout
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 100);      // giảm từ 500
props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 30000);   // 30s
props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 300000); // 5 phút

// Solution 3: Batch processing + async
for (ConsumerRecord<String, String> record : records) {
    // Đừng gọi DB từng record
    batchBuffer.add(record);
}
// Ghi DB 1 lần cho cả batch
orderRepository.saveAll(batchBuffer);
consumer.commitSync();

// Solution 4: Thread pool xử lý song song trong consumer
ExecutorService executor = Executors.newFixedThreadPool(10);
for (ConsumerRecord<String, String> record : records) {
    executor.submit(() -> processRecord(record));
}
// Chú ý: phải đợi tất cả tasks hoàn thành trước khi commit!
```

---

## Case 2: Rebalance Storm — Consumer liên tục rebalance

### Triệu chứng

```
[Consumer] Partitions revoked: [P0, P1, P2]
[Consumer] Partitions assigned: [P0, P2]
... 10 giây sau ...
[Consumer] Partitions revoked: [P0, P2]
[Consumer] Partitions assigned: [P1, P2]
... vòng lặp liên tục
```

Consumer bị kick → rebalance → consumer rejoin → rebalance → ...

### Nguyên nhân

1. **Xử lý quá lâu:** `poll()` interval > `max.poll.interval.ms` → Kafka nghĩ consumer chết
2. **GC pause dài:** JVM GC stop-the-world > `session.timeout.ms`
3. **Consumer instance liên tục restart** (OOM, crash loop)
4. **Network flapping** giữa consumer và broker

### Solutions

```java
// Solution 1: Tăng timeout + giảm batch size
props.put(ConsumerConfig.MAX_POLL_INTERVAL_MS_CONFIG, 600000);  // 10 phút
props.put(ConsumerConfig.MAX_POLL_RECORDS_CONFIG, 50);          // ít records/poll
props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 45000);     // 45s
props.put(ConsumerConfig.HEARTBEAT_INTERVAL_MS_CONFIG, 15000);  // 15s

// Quy tắc: session.timeout ≥ 3 × heartbeat.interval

// Solution 2: Cooperative rebalancing (Kafka 2.4+)
props.put(ConsumerConfig.PARTITION_ASSIGNMENT_STRATEGY_CONFIG,
    "org.apache.kafka.clients.consumer.CooperativeStickyAssignor");
// → Chỉ revoke partitions cần move, không stop-the-world

// Solution 3: Static group membership (Kafka 2.3+)
props.put(ConsumerConfig.GROUP_INSTANCE_ID_CONFIG, "consumer-host-1");
// → Consumer rejoin nhanh hơn, không trigger rebalance ngay
// → Kafka chờ session.timeout trước khi reassign
```

### Monitor rebalance

```java
consumer.subscribe(List.of("order-events"), new ConsumerRebalanceListener() {
    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        System.out.println("REVOKED: " + partitions);
        // Commit offset trước khi mất partition
        consumer.commitSync();
        metrics.counter("kafka.rebalance.revoked").increment();
    }

    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        System.out.println("ASSIGNED: " + partitions);
        metrics.counter("kafka.rebalance.assigned").increment();
    }
});
```

---

## Case 3: Duplicate Messages — Consumer xử lý trùng

### Tình huống

```
1. Consumer nhận message offset=100, xử lý thành công
2. Consumer ghi DB → OK
3. Consumer commit offset → FAIL (network error / rebalance)
4. Consumer restart → poll từ offset=100 lại!
5. → Xử lý trùng!
```

Kafka guarantee: **at-least-once** (mặc định), KHÔNG phải exactly-once.

### Solutions

**Solution 1: Idempotent consumer (khuyến nghị nhất)**

```java
// Dùng unique ID trong message để detect duplicate
for (ConsumerRecord<String, String> record : records) {
    OrderEvent event = deserialize(record.value());

    // Check idempotency key (eventId hoặc topic+partition+offset)
    String idempotencyKey = record.topic() + "-" + record.partition() + "-" + record.offset();
    // Hoặc dùng event.getEventId() nếu producer gán

    if (processedEventStore.exists(idempotencyKey)) {
        log.info("Duplicate detected, skipping: {}", idempotencyKey);
        continue;
    }

    // Xử lý + lưu idempotency key trong CÙNG transaction
    processEvent(event);
    processedEventStore.save(idempotencyKey);
}
consumer.commitSync();
```

**Solution 2: Database upsert (natural idempotency)**

```java
// Thay vì INSERT → dùng UPSERT
// Nếu process lại → ghi đè kết quả cũ (idempotent)

// SQL
"INSERT INTO orders (order_id, status, amount) VALUES (?, ?, ?) " +
"ON CONFLICT (order_id) DO UPDATE SET status = ?, amount = ?"

// JPA
@Transactional
public void processOrder(OrderEvent event) {
    Order order = orderRepo.findById(event.getOrderId())
        .orElse(new Order(event.getOrderId()));
    order.setStatus(event.getStatus());
    order.setAmount(event.getAmount());
    orderRepo.save(order); // insert hoặc update
}
```

**Solution 3: Outbox pattern + Kafka Transactions**

```java
// Producer side: Transactional producer
props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "order-tx-producer");

producer.initTransactions();
try {
    producer.beginTransaction();
    producer.send(new ProducerRecord<>("orders", key, value));
    producer.send(new ProducerRecord<>("audit-log", key, auditValue));
    producer.commitTransaction();
} catch (Exception e) {
    producer.abortTransaction();
}

// Consumer side: read committed only
props.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed");
```

---

## Case 4: Message Ordering bị vi phạm

### Tình huống

```
Producer gửi: Order_CREATED → Order_PAID → Order_SHIPPED (cùng orderId)

Consumer nhận:
  Partition 0: Order_PAID       ← SAI THỨ TỰ!
  Partition 1: Order_CREATED
  Partition 2: Order_SHIPPED

Nguyên nhân: key=null → round-robin → mỗi event vào partition khác!
```

### Solutions

**Solution 1: Key-based partitioning (quan trọng nhất)**

```java
// LUÔN đặt key = entity ID để đảm bảo ordering per entity
String orderId = order.getId();
ProducerRecord<String, String> record =
    new ProducerRecord<>("order-events", orderId, eventJson);
//                                      ↑ key = orderId
//                                      → cùng orderId luôn vào cùng partition
//                                      → ordering đảm bảo per orderId
producer.send(record);
```

**Solution 2: max.in.flight.requests = 1 (khi cần strict ordering)**

```java
// Ngăn reorder do retry
props.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 1);
// Hoặc dùng idempotent producer (cho phép lên 5 mà vẫn ordered)
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
```

**Solution 3: Consumer-side reordering (khi không control được producer)**

```java
// Buffer và sort theo timestamp/sequence number
PriorityQueue<OrderEvent> buffer = new PriorityQueue<>(
    Comparator.comparing(OrderEvent::getSequenceNumber)
);

for (ConsumerRecord<String, String> record : records) {
    OrderEvent event = deserialize(record.value());
    buffer.add(event);
}

// Process theo thứ tự
while (!buffer.isEmpty()) {
    OrderEvent event = buffer.poll();
    if (event.getSequenceNumber() == expectedSequence) {
        processEvent(event);
        expectedSequence++;
    } else {
        // Out of order — chờ thêm hoặc log warning
        log.warn("Out of order: expected={}, got={}",
            expectedSequence, event.getSequenceNumber());
    }
}
```

---

## Case 5: Poison Pill — Message corrupt làm consumer crash loop

### Tình huống

```
Consumer poll → deserialize message → EXCEPTION (corrupt data)
→ Consumer crash → restart → poll cùng message → crash lại!
→ Vòng lặp vô tận → consumer stuck mãi
```

### Solutions

**Solution 1: Dead Letter Queue (DLQ)**

```java
int maxRetries = 3;

for (ConsumerRecord<String, String> record : records) {
    try {
        processRecord(record);
    } catch (Exception e) {
        int retryCount = getRetryCount(record); // từ header hoặc store

        if (retryCount >= maxRetries) {
            // Gửi vào Dead Letter Queue
            sendToDLQ(record, e);
            log.error("Message sent to DLQ after {} retries: partition={}, offset={}",
                maxRetries, record.partition(), record.offset(), e);
        } else {
            // Retry: gửi lại vào retry topic với delay
            sendToRetryTopic(record, retryCount + 1);
        }
    }
}
// LUÔN commit — không bao giờ stuck
consumer.commitSync();
```

**DLQ Producer:**

```java
private void sendToDLQ(ConsumerRecord<String, String> original, Exception error) {
    ProducerRecord<String, String> dlqRecord =
        new ProducerRecord<>("order-events.DLQ", original.key(), original.value());

    // Attach metadata để debug
    dlqRecord.headers().add("original-topic", original.topic().getBytes());
    dlqRecord.headers().add("original-partition",
        String.valueOf(original.partition()).getBytes());
    dlqRecord.headers().add("original-offset",
        String.valueOf(original.offset()).getBytes());
    dlqRecord.headers().add("error-message",
        error.getMessage().getBytes());
    dlqRecord.headers().add("error-timestamp",
        Instant.now().toString().getBytes());

    dlqProducer.send(dlqRecord);
}
```

**Solution 2: Error handler với skip**

```java
// Spring Kafka — DefaultErrorHandler
@Bean
public DefaultErrorHandler errorHandler() {
    // Retry 3 lần, backoff 1s, sau đó skip + log
    return new DefaultErrorHandler(
        (record, exception) -> {
            log.error("Skipping message after retries: topic={}, partition={}, offset={}",
                record.topic(), record.partition(), record.offset(), exception);
            sendToDLQ(record, exception);
        },
        new FixedBackOff(1000L, 3L)  // 3 retries, 1s backoff
    );
}
```

---

## Case 6: Broker chết — Partition leader failover

### Tình huống

```
Broker 1 (leader P0, P1) ← CHẾT!
Broker 2 (follower P0, leader P2)
Broker 3 (follower P1, follower P2)

→ Kafka tự động elect leader mới:
   P0: Broker 2 (follower → leader)
   P1: Broker 3 (follower → leader)
```

### Vấn đề tiềm ẩn

**Data loss nếu config sai:**

```
# Config NGUY HIỂM — có thể mất data
acks=1                              # chỉ chờ leader ack
min.insync.replicas=1               # chỉ cần 1 replica
unclean.leader.election.enable=true # cho phép out-of-sync follower làm leader

# Kịch bản mất data:
# 1. Producer gửi msg → leader ack (acks=1)
# 2. Followers chưa kịp replicate
# 3. Leader chết → follower lên leader → MSG MẤT!
```

**Config AN TOÀN:**

```properties
# Producer
acks=all
retries=2147483647
max.in.flight.requests.per.connection=5
enable.idempotence=true

# Broker / Topic
min.insync.replicas=2            # ít nhất 2 replicas phải sync
replication.factor=3             # 3 copies
unclean.leader.election.enable=false  # KHÔNG cho out-of-sync lên leader

# Với config này:
# 1. Producer gửi msg → leader + ít nhất 1 follower ack
# 2. Leader chết → in-sync follower lên leader → KHÔNG MẤT DATA
# 3. Nếu cả 2 ISR chết → topic unavailable (thà unavailable hơn mất data)
```

### NotEnoughReplicasException

```
# Khi min.insync.replicas=2, replication.factor=3
# Nếu 2 brokers chết → chỉ còn 1 ISR → ÍT HƠN min.insync.replicas

Producer send → NotEnoughReplicasException!
# Topic tạm thời unavailable cho đến khi broker recovery
```

---

## Case 7: Consumer Group bị stuck — Committed offset vượt quá log

### Tình huống

```
Topic retention = 7 ngày
Consumer bị tắt 10 ngày
→ Committed offset = 5000, nhưng earliest offset hiện tại = 100000
→ Messages 5000-99999 đã bị xóa!

Consumer restart → OffsetOutOfRangeException hoặc auto.offset.reset kick in
```

### Solutions

```java
// auto.offset.reset config
props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
// → Nếu committed offset không hợp lệ → đọc từ đầu (còn tồn tại)

props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "latest");
// → Skip tất cả, chỉ đọc messages mới

// Detect vấn đề
consumer.subscribe(List.of("my-topic"), new ConsumerRebalanceListener() {
    @Override
    public void onPartitionsAssigned(Collection<TopicPartition> partitions) {
        for (TopicPartition tp : partitions) {
            long committed = consumer.committed(Set.of(tp)).get(tp) != null
                ? consumer.committed(Set.of(tp)).get(tp).offset() : -1;

            Map<TopicPartition, Long> beginOffsets =
                consumer.beginningOffsets(List.of(tp));
            long earliest = beginOffsets.get(tp);

            if (committed > 0 && committed < earliest) {
                log.warn("Offset reset detected! partition={}, committed={}, earliest={}. " +
                    "Data loss of {} messages!",
                    tp, committed, earliest, earliest - committed);
                alertingService.sendAlert("Kafka offset reset on " + tp);
            }
        }
    }

    @Override
    public void onPartitionsRevoked(Collection<TopicPartition> partitions) {
        consumer.commitSync();
    }
});
```

---

## Case 8: Producer timeout — Messages stuck trong buffer

### Tình huống

```
Producer gửi messages nhanh hơn network/broker xử lý
→ Buffer đầy → block hoặc throw BufferExhaustedException/TimeoutException

[WARN] Expiring 500 record(s) for topic-0:120000 ms has passed since batch creation
```

### Solutions

```java
// Tune producer buffer & timeout
props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 67108864);   // 64MB buffer
props.put(ProducerConfig.MAX_BLOCK_MS_CONFIG, 60000);       // block tối đa 60s khi buffer đầy
props.put(ProducerConfig.DELIVERY_TIMEOUT_MS_CONFIG, 120000); // 2 phút total delivery timeout
props.put(ProducerConfig.REQUEST_TIMEOUT_MS_CONFIG, 30000);  // 30s per request
props.put(ProducerConfig.RETRY_BACKOFF_MS_CONFIG, 100);      // 100ms giữa retries

// Monitoring: send callback
producer.send(record, (metadata, exception) -> {
    if (exception instanceof TimeoutException) {
        metrics.counter("kafka.producer.timeout").increment();
        log.error("Message delivery timed out: {}", exception.getMessage());
        // Fallback: lưu vào local queue / DB
        fallbackQueue.add(record);
    } else if (exception != null) {
        metrics.counter("kafka.producer.error").increment();
        log.error("Send failed: {}", exception.getMessage());
    } else {
        metrics.counter("kafka.producer.success").increment();
    }
});

// Tuning batch cho throughput cao
props.put(ProducerConfig.BATCH_SIZE_CONFIG, 65536);         // 64KB batch
props.put(ProducerConfig.LINGER_MS_CONFIG, 10);             // chờ 10ms gom batch
props.put(ProducerConfig.COMPRESSION_TYPE_CONFIG, "lz4");    // compress
```

---

## Case 9: Hot Partition — Một partition nhận traffic nhiều hơn

### Tình huống

```
Topic: user-events (3 partitions)

Key distribution:
  user_1 (VIP) → 50,000 events/day
  user_2       →    100 events/day
  user_3       →    100 events/day
  ...

hash("user_1") % 3 = partition 0
→ Partition 0: 50,000 events (HOT!)
→ Partition 1:  5,000 events
→ Partition 2:  5,000 events

Consumer 0: quá tải, lag
Consumer 1, 2: idle
```

### Solutions

**Solution 1: Composite key**

```java
// Thay vì key = userId (skewed)
// Dùng key = userId + suffix để phân tán
String key = userId + "-" + (event.hashCode() % 10); // spread thành 10 "sub-keys"
// Trade-off: mất ordering per userId

// Hoặc nếu cần ordering per userId nhưng muốn spread:
String key = userId + "-" + (eventType.hashCode() % 3);
// → Cùng userId + eventType vẫn ordered, nhưng spread hơn
```

**Solution 2: Custom partitioner**

```java
public class BalancedPartitioner implements Partitioner {

    private final Set<String> hotKeys = Set.of("user_1", "user_2"); // known hot keys

    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                        Object value, byte[] valueBytes, Cluster cluster) {
        int numPartitions = cluster.partitionCountForTopic(topic);

        if (key == null) {
            return ThreadLocalRandom.current().nextInt(numPartitions);
        }

        String keyStr = key.toString();
        if (hotKeys.contains(keyStr)) {
            // Hot keys → round-robin across all partitions
            return ThreadLocalRandom.current().nextInt(numPartitions);
        }

        // Normal keys → consistent hash
        return Math.abs(keyStr.hashCode()) % numPartitions;
    }

    @Override public void close() {}
    @Override public void configure(Map<String, ?> configs) {}
}

// Config
props.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, BalancedPartitioner.class.getName());
```

**Solution 3: Tăng partitions (nếu nguyên nhân chỉ là ít partitions)**

```bash
# Tăng từ 3 lên 12 → spread tốt hơn
kafka-topics.sh --alter --topic user-events --partitions 12 \
    --bootstrap-server localhost:9092

# LƯU Ý: Tăng partitions = thay đổi hash mapping!
# key X trước ở partition 0, sau có thể ở partition 7
# → Ordering bị break trong giai đoạn chuyển!
```

---

## Case 10: Schema Evolution gây lỗi deserialization

### Tình huống

```
Producer deploy V2 schema: thêm field "priority" (type: int, KHÔNG có default)
Schema Registry: compatibility=BACKWARD
→ Schema Registry REJECT! (thiếu default value cho new field)

Nhưng team bypass bằng cách set compatibility=NONE tạm thời...
→ Register thành công
→ Consumer V1 nhận data V2 → deserialization error → consumer crash!
```

### Prevention

```java
// 1. KHÔNG BAO GIỜ set compatibility=NONE trong production

// 2. Mọi field mới PHẢI có default value
{
    "name": "priority",
    "type": ["null", "int"],  // union type, nullable
    "default": null            // default = null
}

// 3. CI/CD kiểm tra compatibility trước khi deploy
// (xem script ở 02-schema-registry.md)

// 4. Consumer phải handle unknown fields gracefully
try {
    GenericRecord record = (GenericRecord) consumerRecord.value();
    processRecord(record);
} catch (SerializationException e) {
    log.error("Deserialization failed — possible schema mismatch: {}",
        e.getMessage());
    sendToDLQ(consumerRecord, e);
}
```

### Recovery khi đã xảy ra

```java
// Step 1: Rollback producer về schema V1

// Step 2: Consumer V1 cần skip messages V2 đã produce
// Dùng standalone consumer + seek để nhảy qua khoảng lỗi

// Step 3: Fix schema V2 (thêm default value)
{
    "name": "priority",
    "type": ["null", "int"],
    "default": null
}

// Step 4: Register V2 đúng cách
// Step 5: Replay messages bị skip (nếu cần) qua DLQ
```

---

## Case 11: Consumer commit offset trước khi xử lý xong

### Tình huống (at-most-once — MẤT DATA)

```java
// SAI — commit trước, xử lý sau
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));
    consumer.commitSync(); // commit ngay!

    for (ConsumerRecord<String, String> record : records) {
        processRecord(record); // crash ở đây → message MẤT vì đã commit!
    }
}
```

### Solution

```java
// ĐÚNG — xử lý xong rồi mới commit (at-least-once)
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));

    for (ConsumerRecord<String, String> record : records) {
        processRecord(record); // xử lý trước
    }

    if (!records.isEmpty()) {
        consumer.commitSync(); // commit sau
    }
}

// TỐT HƠN — commit per partition khi xử lý xong
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(1));

    for (TopicPartition partition : records.partitions()) {
        List<ConsumerRecord<String, String>> partitionRecords =
                records.records(partition);

        for (ConsumerRecord<String, String> record : partitionRecords) {
            processRecord(record);
        }

        // Commit offset cho partition này
        long lastOffset = partitionRecords.get(partitionRecords.size() - 1).offset();
        consumer.commitSync(Map.of(partition,
                new OffsetAndMetadata(lastOffset + 1)));
    }
}
```

---

## Case 12: Multi-DC (Data Center) Replication lỗi

### Tình huống

```
DC1 (primary) ──MirrorMaker 2──→ DC2 (DR)

MirrorMaker bị lag 2 giờ
DC1 chết → failover sang DC2
→ 2 giờ data gần nhất KHÔNG có ở DC2!
```

### Solutions

```properties
# MirrorMaker 2 tuning
# Tăng throughput
tasks.max=10
consumer.max.poll.records=1000
producer.batch.size=65536
producer.linger.ms=5
producer.compression.type=lz4

# Monitor MM2 lag
kafka-consumer-groups.sh --bootstrap-server dc2:9092 \
    --describe --group connect-mirror-source

# Alert khi lag > threshold
# Prometheus: kafka_consumer_group_lag > 10000
```

### Active-Active pattern

```
DC1 ←──────→ DC2
user-events.dc1  →  user-events.dc1 (replicated)
user-events.dc2  ←  user-events.dc2 (replicated)

Consumer đọc:
- Local topic + replicated topic
- Dedup bằng event ID
```

---

## Monitoring Checklist — Metrics quan trọng

### Producer metrics

```
kafka.producer.record-send-rate          — messages/s gửi
kafka.producer.record-error-rate         — errors/s
kafka.producer.request-latency-avg       — latency trung bình
kafka.producer.buffer-available-bytes     — buffer còn trống
kafka.producer.waiting-threads           — threads chờ buffer
```

### Consumer metrics

```
kafka.consumer.records-consumed-rate     — messages/s đọc
kafka.consumer.records-lag-max           — max lag (QUAN TRỌNG NHẤT!)
kafka.consumer.commit-rate               — commits/s
kafka.consumer.poll-rate                 — polls/s
kafka.consumer.rebalance-rate            — rebalances (phải thấp!)
```

### Broker metrics

```
kafka.server.UnderReplicatedPartitions   — partitions thiếu replica (phải = 0!)
kafka.server.IsrShrinksPerSec            — ISR shrink rate (phải thấp!)
kafka.server.ActiveControllerCount       — phải = 1 trong cluster
kafka.network.RequestMetrics.TotalTimeMs — request latency
kafka.log.LogFlushRateAndTimeMs          — disk flush time
```

### Alerting rules (Prometheus)

```yaml
# Consumer lag quá cao
- alert: KafkaConsumerLagHigh
  expr: kafka_consumer_group_lag > 10000
  for: 5m

# Under-replicated partitions
- alert: KafkaUnderReplicated
  expr: kafka_server_replicamanager_underreplicatedpartitions > 0
  for: 1m
  severity: critical

# No active controller
- alert: KafkaNoController
  expr: kafka_controller_activecontrollercount != 1
  for: 1m
  severity: critical

# Consumer rebalance rate cao
- alert: KafkaRebalanceStorm
  expr: rate(kafka_consumer_rebalance_total[5m]) > 0.5
  for: 5m
```
