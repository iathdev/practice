# Kafka Fundamentals

---

## 1. Kafka là gì?

**Apache Kafka** là distributed event streaming platform, dùng để publish/subscribe messages với throughput cao, fault-tolerant, và durable.

```
Producer A ──→ ┌─────────────────────────┐ ──→ Consumer Group 1
Producer B ──→ │     Kafka Cluster        │ ──→ Consumer Group 2
Producer C ──→ │  Broker 1 │ Broker 2 │ B3│ ──→ Consumer Group 3
               └─────────────────────────┘
                        ↕
                   ZooKeeper / KRaft
```

**Use cases:**
- Event-driven microservices
- Log aggregation
- Stream processing (real-time analytics)
- Change Data Capture (CDC)
- Message queue thay thế RabbitMQ cho high throughput

---

## 2. Core Concepts

### Topic & Partition

```
Topic: "user-events"
├── Partition 0: [msg0, msg1, msg4, msg7, ...]  → offset tăng dần
├── Partition 1: [msg2, msg3, msg5, msg8, ...]
└── Partition 2: [msg6, msg9, msg10, ...]
```

- **Topic:** Logical channel cho messages (tương tự table trong DB)
- **Partition:** Đơn vị parallelism, mỗi partition là ordered log
- **Offset:** Vị trí message trong partition (immutable, tăng dần)

### Message (Record)

```
Record {
    key:       byte[]     // quyết định partition (null → round-robin)
    value:     byte[]     // nội dung message
    headers:   Header[]   // metadata (tracing, version...)
    timestamp: long       // event time hoặc ingestion time
    offset:    long       // do Kafka gán (không set được)
    partition: int        // do Kafka gán (hoặc producer chọn)
}
```

### Broker & Cluster

```
Kafka Cluster
├── Broker 0 (controller)
│   ├── Topic A - Partition 0 (leader)
│   ├── Topic A - Partition 1 (follower)
│   └── Topic B - Partition 0 (follower)
├── Broker 1
│   ├── Topic A - Partition 0 (follower)
│   ├── Topic A - Partition 1 (leader)
│   └── Topic B - Partition 0 (leader)
└── Broker 2
    ├── Topic A - Partition 0 (follower)
    ├── Topic A - Partition 1 (follower)
    └── Topic B - Partition 0 (follower)
```

- **Broker:** Một Kafka server instance
- **Leader:** Partition nhận reads/writes
- **Follower:** Replica, sync từ leader, take over khi leader chết
- **Replication factor:** Số copies (thường = 3)

### Consumer Group

```
Topic: orders (3 partitions)

Consumer Group "order-service"
├── Consumer 1 ← Partition 0
├── Consumer 2 ← Partition 1
└── Consumer 3 ← Partition 2

Consumer Group "analytics"
├── Consumer A ← Partition 0, 1
└── Consumer B ← Partition 2
```

- Mỗi partition chỉ được **một consumer** trong group đọc
- Thêm consumer > số partition → consumer thừa idle
- Mỗi **consumer group** đọc independently (nhận tất cả messages)

---

## 3. Producer

### Acks (Acknowledgment)

| acks | Mô tả | Durability | Performance |
|------|--------|-----------|-------------|
| `0` | Không chờ ack | Thấp nhất (mất data được) | **Nhanh nhất** |
| `1` | Chờ leader ack | Trung bình | Trung bình |
| `all` (-1) | Chờ **tất cả ISR** ack | **Cao nhất** | Chậm nhất |

```
acks=0:  Producer → Broker (fire and forget)
acks=1:  Producer → Broker (leader) → ack
acks=all: Producer → Broker (leader) → replicate to ISR → ack
```

### Partitioning Strategy

```
1. Key = null       → Round-robin (hoặc sticky partition)
2. Key != null      → hash(key) % numPartitions
3. Custom partitioner → bạn quyết định

// Ví dụ: key = userId → cùng user luôn vào cùng partition
// → đảm bảo ordering per user
```

### Idempotent Producer

```
enable.idempotence = true
→ Kafka gán Producer ID + Sequence Number
→ Broker detect và loại bỏ duplicate
→ Exactly-once semantics (within single partition)
```

### Batching & Compression

```
batch.size = 16384          // 16KB — gom messages trước khi gửi
linger.ms = 5               // chờ tối đa 5ms để gom batch
compression.type = snappy   // nén batch (none/gzip/snappy/lz4/zstd)

// Trade-off: linger.ms cao → throughput cao, latency cao hơn
```

---

## 4. Consumer

### Commit Offset

```
Partition 0: [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
                            ↑ committed offset = 5
                            → next poll bắt đầu từ 5

// Auto commit
enable.auto.commit = true
auto.commit.interval.ms = 5000  // commit mỗi 5s

// Manual commit (khuyến nghị cho production)
enable.auto.commit = false
consumer.commitSync()   // block cho đến khi commit thành công
consumer.commitAsync()  // không block, callback
```

### Consumer Rebalance

Khi consumer join/leave group → Kafka rebalance partitions.

```
Trước rebalance:
  Consumer 1 ← P0, P1
  Consumer 2 ← P2

Consumer 3 joins →

Sau rebalance:
  Consumer 1 ← P0
  Consumer 2 ← P1
  Consumer 3 ← P2
```

**Rebalance strategies:**
- **Eager:** Revoke tất cả, gán lại (stop-the-world)
- **Cooperative (Incremental):** Chỉ revoke partitions cần di chuyển (ít disruption)

### auto.offset.reset

Khi consumer mới (chưa có committed offset):

| Giá trị | Hành vi |
|---------|---------|
| `earliest` | Đọc từ đầu (offset 0) |
| `latest` | Chỉ đọc messages mới (default) |
| `none` | Ném exception nếu không có offset |

---

## 5. Retention & Compaction

### Time-based retention

```
retention.ms = 604800000  // 7 ngày (default)
// Sau 7 ngày → segment bị xóa
```

### Size-based retention

```
retention.bytes = 1073741824  // 1GB per partition
// Khi vượt 1GB → segment cũ nhất bị xóa
```

### Log Compaction

```
cleanup.policy = compact

// Giữ lại message MỚI NHẤT cho mỗi key
// Trước compaction:
key=A: v1, key=B: v1, key=A: v2, key=C: v1, key=B: v2

// Sau compaction:
key=A: v2, key=C: v1, key=B: v2
// (chỉ giữ giá trị mới nhất per key)

// Tombstone: value = null → xóa key sau delete.retention.ms
```

**Use case:** CDC, state store, configuration topics.

---

## 6. Exactly-Once Semantics (EOS)

### Idempotent Producer (within partition)
```
enable.idempotence = true
// → Producer ID + Sequence → dedup tự động
```

### Transactional Producer (cross partitions)
```
// Gửi messages vào nhiều partitions trong 1 transaction
// Tất cả thành công hoặc tất cả fail
transactional.id = "my-transactional-producer"
```

### Consumer: read_committed
```
isolation.level = read_committed
// Chỉ đọc messages đã committed (trong transaction)
// Messages chưa commit hoặc aborted → bị skip
```

---

## 7. Kafka vs Alternatives

| | Kafka | RabbitMQ | AWS SQS |
|--|-------|----------|---------|
| Model | Log-based | Queue + Exchange | Queue |
| Ordering | Per-partition | Per-queue | Best-effort (FIFO riêng) |
| Replay | **Có** (seek offset) | Không | Không |
| Throughput | **Rất cao** (millions/s) | Trung bình | Trung bình |
| Retention | Configurable | Ack → delete | 14 ngày max |
| Consumer model | Pull | Push | Pull |
| Complexity | Cao | Trung bình | Thấp (managed) |
| Dùng khi | Event streaming, high volume | Task queue, routing phức tạp | Simple queue, serverless |
