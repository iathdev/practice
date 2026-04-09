# Standalone Consumer & Seek Offset

---

## 1. Consumer Group vs Standalone Consumer

### Consumer Group (thông thường)

```java
// Consumer Group — Kafka tự quản lý partition assignment
consumer.subscribe(List.of("user-events"));
// → Kafka assign partitions tự động
// → Rebalance khi consumer join/leave
// → Offset committed cho group
```

### Standalone Consumer (assign trực tiếp)

```java
// Standalone — TỰ MÌNH quản lý partition assignment
TopicPartition p0 = new TopicPartition("user-events", 0);
TopicPartition p1 = new TopicPartition("user-events", 1);
consumer.assign(List.of(p0, p1));
// → KHÔNG có rebalance
// → KHÔNG thuộc consumer group (group.id vẫn cần cho offset commit)
// → Mình tự quyết partition nào đọc
```

| | Consumer Group | Standalone |
|--|---------------|------------|
| Assignment | Kafka tự động | Tự assign |
| Rebalance | Có | **Không** |
| Scale | Tự động | Tự quản lý |
| Offset commit | Group-managed | Tự quản lý (hoặc group) |
| Use case | Production normal | Debug, replay, tool |

---

## 2. Standalone Consumer — Full Example

```java
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.TopicPartition;
import org.apache.kafka.common.serialization.StringDeserializer;

import java.time.Duration;
import java.util.*;

public class StandaloneConsumer {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        // group.id vẫn cần nếu muốn commit offset
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "standalone-debug-tool");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {

            // Lấy danh sách partitions
            List<TopicPartition> partitions = consumer.partitionsFor("user-events")
                    .stream()
                    .map(info -> new TopicPartition(info.topic(), info.partition()))
                    .toList();

            System.out.println("Available partitions: " + partitions);

            // Assign specific partitions (không subscribe!)
            consumer.assign(partitions);

            // Poll loop
            int totalMessages = 0;
            while (totalMessages < 100) { // đọc tối đa 100 messages
                ConsumerRecords<String, String> records =
                        consumer.poll(Duration.ofMillis(1000));

                if (records.isEmpty()) break; // không còn data

                for (ConsumerRecord<String, String> record : records) {
                    System.out.printf("P%d | offset=%d | key=%s | value=%s%n",
                            record.partition(), record.offset(),
                            record.key(), record.value());
                    totalMessages++;
                }
            }

            System.out.println("Total messages read: " + totalMessages);
        }
    }
}
```

---

## 3. Seek Offset — Điều khiển vị trí đọc

### seek() — Nhảy đến offset cụ thể

```java
TopicPartition partition0 = new TopicPartition("user-events", 0);
consumer.assign(List.of(partition0));

// Nhảy đến offset 100
consumer.seek(partition0, 100);

// Bây giờ poll() sẽ bắt đầu từ offset 100
ConsumerRecords<String, String> records = consumer.poll(Duration.ofSeconds(5));
```

### seekToBeginning() — Đọc lại từ đầu

```java
consumer.assign(List.of(partition0, partition1, partition2));
consumer.seekToBeginning(List.of(partition0, partition1, partition2));

// Poll sẽ đọc từ offset 0 cho tất cả partitions
```

### seekToEnd() — Nhảy đến cuối

```java
consumer.assign(List.of(partition0));
consumer.seekToEnd(List.of(partition0));

// Chỉ đọc messages MỚI từ đây trở đi
```

### offsetsForTimes() — Seek theo timestamp

```java
// Tìm offset tại thời điểm cụ thể
long targetTimestamp = Instant.parse("2025-01-15T10:00:00Z").toEpochMilli();

Map<TopicPartition, Long> timestampToSearch = new HashMap<>();
for (TopicPartition tp : partitions) {
    timestampToSearch.put(tp, targetTimestamp);
}

// Kafka trả về offset tương ứng với timestamp
Map<TopicPartition, OffsetAndTimestamp> result =
        consumer.offsetsForTimes(timestampToSearch);

// Seek đến các offset đó
for (Map.Entry<TopicPartition, OffsetAndTimestamp> entry : result.entrySet()) {
    if (entry.getValue() != null) {
        consumer.seek(entry.getKey(), entry.getValue().offset());
        System.out.printf("Partition %d → seek to offset %d (timestamp %d)%n",
                entry.getKey().partition(),
                entry.getValue().offset(),
                entry.getValue().timestamp());
    }
}
```

---

## 4. Use Cases thực tế

### Case 1: Replay events sau khi fix bug

**Tình huống:** Consumer có bug xử lý sai 1000 messages từ 2h trước.

```java
public class EventReplayer {

    public static void replayFromTimestamp(String topic, Instant fromTime) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "replay-tool-" + System.currentTimeMillis());
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
            // Lấy tất cả partitions
            List<TopicPartition> partitions = consumer.partitionsFor(topic)
                    .stream()
                    .map(p -> new TopicPartition(p.topic(), p.partition()))
                    .toList();

            consumer.assign(partitions);

            // Seek theo timestamp
            Map<TopicPartition, Long> timestamps = new HashMap<>();
            partitions.forEach(tp -> timestamps.put(tp, fromTime.toEpochMilli()));

            Map<TopicPartition, OffsetAndTimestamp> offsets =
                    consumer.offsetsForTimes(timestamps);

            offsets.forEach((tp, offsetAndTime) -> {
                if (offsetAndTime != null) {
                    consumer.seek(tp, offsetAndTime.offset());
                    System.out.printf("Partition %d: replay from offset %d%n",
                            tp.partition(), offsetAndTime.offset());
                }
            });

            // Replay
            Instant endTime = Instant.now(); // chỉ replay đến hiện tại
            boolean hasMore = true;
            while (hasMore) {
                ConsumerRecords<String, String> records =
                        consumer.poll(Duration.ofSeconds(2));

                if (records.isEmpty()) {
                    hasMore = false;
                    continue;
                }

                for (ConsumerRecord<String, String> record : records) {
                    if (record.timestamp() > endTime.toEpochMilli()) {
                        hasMore = false;
                        break;
                    }
                    reprocessMessage(record); // logic xử lý lại
                }
            }
        }
    }

    private static void reprocessMessage(ConsumerRecord<String, String> record) {
        System.out.printf("Reprocessing: P%d offset=%d key=%s%n",
                record.partition(), record.offset(), record.key());
        // ... xử lý lại business logic
    }
}
```

### Case 2: Debug tool — xem messages trong khoảng offset

```java
public class KafkaDebugTool {

    /**
     * Đọc messages trong khoảng [fromOffset, toOffset] từ một partition.
     */
    public static List<ConsumerRecord<String, String>> readRange(
            String topic, int partition, long fromOffset, long toOffset) {

        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        List<ConsumerRecord<String, String>> result = new ArrayList<>();

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
            TopicPartition tp = new TopicPartition(topic, partition);
            consumer.assign(List.of(tp));
            consumer.seek(tp, fromOffset);

            while (true) {
                ConsumerRecords<String, String> records =
                        consumer.poll(Duration.ofSeconds(2));

                if (records.isEmpty()) break;

                for (ConsumerRecord<String, String> record : records) {
                    if (record.offset() > toOffset) {
                        return result; // đủ rồi
                    }
                    result.add(record);
                }
            }
        }

        return result;
    }

    /**
     * Tìm message theo key trong topic.
     */
    public static List<ConsumerRecord<String, String>> findByKey(
            String topic, String targetKey, int maxResults) {

        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        List<ConsumerRecord<String, String>> found = new ArrayList<>();

        try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
            List<TopicPartition> partitions = consumer.partitionsFor(topic)
                    .stream()
                    .map(p -> new TopicPartition(p.topic(), p.partition()))
                    .toList();

            consumer.assign(partitions);
            consumer.seekToBeginning(partitions);

            int emptyPolls = 0;
            while (found.size() < maxResults && emptyPolls < 3) {
                ConsumerRecords<String, String> records =
                        consumer.poll(Duration.ofSeconds(2));

                if (records.isEmpty()) {
                    emptyPolls++;
                    continue;
                }
                emptyPolls = 0;

                for (ConsumerRecord<String, String> record : records) {
                    if (targetKey.equals(record.key())) {
                        found.add(record);
                        if (found.size() >= maxResults) break;
                    }
                }
            }
        }

        return found;
    }
}
```

### Case 3: Offset gap detector

```java
/**
 * Phát hiện gaps trong offset (messages bị skip/mất).
 */
public static Map<Integer, List<Long>> detectGaps(String topic) {
    Properties props = new Properties();
    props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
    props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
    props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
    props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

    Map<Integer, List<Long>> gaps = new HashMap<>();

    try (KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props)) {
        List<TopicPartition> partitions = consumer.partitionsFor(topic)
                .stream()
                .map(p -> new TopicPartition(p.topic(), p.partition()))
                .toList();

        consumer.assign(partitions);
        consumer.seekToBeginning(partitions);

        Map<Integer, Long> lastOffset = new HashMap<>();

        int emptyPolls = 0;
        while (emptyPolls < 3) {
            ConsumerRecords<String, String> records =
                    consumer.poll(Duration.ofSeconds(2));

            if (records.isEmpty()) { emptyPolls++; continue; }
            emptyPolls = 0;

            for (ConsumerRecord<String, String> record : records) {
                int p = record.partition();
                long offset = record.offset();

                Long prev = lastOffset.get(p);
                if (prev != null && offset != prev + 1) {
                    // Gap detected!
                    gaps.computeIfAbsent(p, k -> new ArrayList<>())
                            .add(prev + 1); // missing offset start
                    System.out.printf("GAP: partition=%d, missing offsets %d to %d%n",
                            p, prev + 1, offset - 1);
                }
                lastOffset.put(p, offset);
            }
        }
    }

    return gaps;
}
```

---

## 5. Standalone Consumer trong Go

```go
package main

import (
	"context"
	"fmt"
	"log"
	"time"

	"github.com/segmentio/kafka-go"
)

func main() {
	// Standalone reader — assign partition trực tiếp
	reader := kafka.NewReader(kafka.ReaderConfig{
		Brokers:   []string{"localhost:9092"},
		Topic:     "user-events",
		Partition: 0,           // specific partition
		MinBytes:  1,
		MaxBytes:  10e6,        // 10MB
	})
	defer reader.Close()

	// Seek to specific offset
	reader.SetOffset(100)
	fmt.Println("Seeking to offset 100")

	// Hoặc seek by timestamp
	// reader.SetOffsetAt(context.Background(), time.Now().Add(-2*time.Hour))

	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	for {
		msg, err := reader.ReadMessage(ctx)
		if err != nil {
			if err == context.DeadlineExceeded {
				break
			}
			log.Printf("read error: %v", err)
			break
		}

		fmt.Printf("P%d | offset=%d | key=%s | value=%s | time=%s\n",
			msg.Partition, msg.Offset,
			string(msg.Key), string(msg.Value),
			msg.Time.Format(time.RFC3339))
	}
}

// Seek theo timestamp
func seekByTime(reader *kafka.Reader, targetTime time.Time) error {
	return reader.SetOffsetAt(context.Background(), targetTime)
}

// Replay từ đầu
func replayFromBeginning(reader *kafka.Reader) {
	reader.SetOffset(kafka.FirstOffset)
}

// Nhảy đến cuối
func seekToEnd(reader *kafka.Reader) {
	reader.SetOffset(kafka.LastOffset)
}
```

---

## 6. Kết hợp Standalone Consumer + Schema Registry

```java
/**
 * Tool đọc Avro messages từ một partition cụ thể với offset range.
 * Hữu ích khi debug production issues.
 */
public class AvroDebugReader {

    public static void readAvroRange(String topic, int partition,
                                     long fromOffset, long toOffset) {

        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, KafkaAvroDeserializer.class);
        props.put(KafkaAvroDeserializerConfig.SCHEMA_REGISTRY_URL_CONFIG, "http://localhost:8081");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false);

        try (KafkaConsumer<String, GenericRecord> consumer = new KafkaConsumer<>(props)) {
            TopicPartition tp = new TopicPartition(topic, partition);
            consumer.assign(List.of(tp));
            consumer.seek(tp, fromOffset);

            while (true) {
                ConsumerRecords<String, GenericRecord> records =
                        consumer.poll(Duration.ofSeconds(2));

                if (records.isEmpty()) break;

                for (ConsumerRecord<String, GenericRecord> record : records) {
                    if (record.offset() > toOffset) return;

                    GenericRecord avroRecord = record.value();
                    System.out.printf("offset=%d | schema_version=%s%n",
                            record.offset(), avroRecord.getSchema().getFullName());

                    // In tất cả fields
                    avroRecord.getSchema().getFields().forEach(field -> {
                        System.out.printf("  %s = %s%n",
                                field.name(), avroRecord.get(field.name()));
                    });
                    System.out.println("---");
                }
            }
        }
    }
}
```
