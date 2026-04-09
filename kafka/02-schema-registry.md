# Schema Registry & Data Evolution

---

## 1. Vấn đề không có Schema Registry

```
Producer (v1): {"userId": 123, "name": "Alice"}
     ↓ deploy v2
Producer (v2): {"user_id": 123, "fullName": "Alice Nguyen", "age": 30}
     ↓
Consumer (vẫn đọc v1 format) → CRASH! field "name" không tồn tại!
```

**Vấn đề:**
- Producer thay đổi schema → Consumer không biết → parse lỗi
- Không ai kiểm soát schema evolution
- Khó rollback
- Debug khó khi data corrupt

---

## 2. Schema Registry là gì?

**Confluent Schema Registry** là centralized service lưu trữ và quản lý schemas (Avro, Protobuf, JSON Schema) cho Kafka messages.

```
Producer                      Schema Registry                Consumer
   │                               │                            │
   ├─ 1. Register schema ────────→ │                            │
   │  ← schema_id=1 ──────────────┤                            │
   │                               │                            │
   ├─ 2. Send [schema_id=1|data] ─────────→ Kafka ──────────→  │
   │                               │                            │
   │                               │  3. Fetch schema (id=1) ←─┤
   │                               ├─ schema definition ──────→ │
   │                               │                            │
   │                               │  4. Deserialize with schema│
```

**Lưu ý quan trọng:** Schema Registry KHÔNG nằm trong Kafka. Nó là service riêng (HTTP REST API) chạy bên cạnh Kafka cluster.

---

## 3. Serialization Formats

### Avro (phổ biến nhất với Kafka)

```json
{
  "type": "record",
  "name": "User",
  "namespace": "com.example",
  "fields": [
    {"name": "id", "type": "long"},
    {"name": "name", "type": "string"},
    {"name": "email", "type": ["null", "string"], "default": null},
    {"name": "age", "type": "int", "default": 0},
    {"name": "tags", "type": {"type": "array", "items": "string"}, "default": []}
  ]
}
```

**Ưu điểm Avro:**
- Compact binary format
- Schema evolution mạnh (backward/forward compatible)
- Schema đi kèm data (self-describing)
- Dynamic typing (không cần generate code, nhưng có thể)

### Protobuf

```protobuf
syntax = "proto3";
message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
}
```

### JSON Schema

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "id": {"type": "integer"},
    "name": {"type": "string"},
    "email": {"type": "string"}
  },
  "required": ["id", "name"]
}
```

| | Avro | Protobuf | JSON Schema |
|--|------|----------|-------------|
| Format | Binary | Binary | Text (JSON) |
| Size | Nhỏ | Nhỏ | Lớn |
| Schema evolution | Mạnh | Mạnh | Yếu hơn |
| Code gen | Optional | Bắt buộc | Không |
| Human-readable data | Không | Không | Có |
| Kafka ecosystem | **Tốt nhất** | Tốt | OK |

---

## 4. Compatibility Types

Schema Registry kiểm tra compatibility mỗi khi register schema mới.

### BACKWARD (default)

**Consumer mới đọc được data cũ.**

```
Schema v1: {id, name}
Schema v2: {id, name, email(default=null)}  ← thêm field có default

Consumer v2 đọc data v1 → OK (email = null)
Consumer v2 đọc data v2 → OK
```

**Cho phép:**
- Thêm field có default value
- Xóa field không có default

**Deploy order:** Consumer trước, Producer sau.

### FORWARD

**Consumer cũ đọc được data mới.**

```
Schema v1: {id, name, email}
Schema v2: {id, name}  ← xóa field

Consumer v1 đọc data v2 → OK (email = default)
Consumer v1 đọc data v1 → OK
```

**Cho phép:**
- Xóa field có default value
- Thêm field không có default

**Deploy order:** Producer trước, Consumer sau.

### FULL

**Cả BACKWARD lẫn FORWARD.** An toàn nhất.

```
Chỉ cho phép:
- Thêm field có default value
- Xóa field có default value
```

### NONE

Không kiểm tra — nguy hiểm, chỉ dùng cho dev.

### BACKWARD_TRANSITIVE / FORWARD_TRANSITIVE / FULL_TRANSITIVE

Kiểm tra với **tất cả** versions trước đó, không chỉ version ngay trước.

```
BACKWARD:             v3 compatible với v2
BACKWARD_TRANSITIVE:  v3 compatible với v2 VÀ v1 VÀ tất cả versions
```

---

## 5. Schema Registry API

```bash
# Xem tất cả subjects
curl http://localhost:8081/subjects

# Register schema (tên subject = topic-value hoặc topic-key)
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{
    "schema": "{\"type\":\"record\",\"name\":\"User\",\"fields\":[{\"name\":\"id\",\"type\":\"long\"},{\"name\":\"name\",\"type\":\"string\"}]}"
  }' \
  http://localhost:8081/subjects/user-events-value/versions

# Response: {"id": 1}

# Xem schema by ID
curl http://localhost:8081/schemas/ids/1

# Xem tất cả versions của subject
curl http://localhost:8081/subjects/user-events-value/versions

# Xem schema by version
curl http://localhost:8081/subjects/user-events-value/versions/1

# Check compatibility trước khi register
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "..."}' \
  http://localhost:8081/compatibility/subjects/user-events-value/versions/latest

# Set compatibility level
curl -X PUT -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"compatibility": "FULL"}' \
  http://localhost:8081/config/user-events-value

# Global config
curl http://localhost:8081/config
```

---

## 6. Java — Producer & Consumer với Schema Registry

### Dependencies (Maven)

```xml
<dependencies>
    <dependency>
        <groupId>org.apache.kafka</groupId>
        <artifactId>kafka-clients</artifactId>
        <version>3.7.0</version>
    </dependency>
    <dependency>
        <groupId>io.confluent</groupId>
        <artifactId>kafka-avro-serializer</artifactId>
        <version>7.6.0</version>
    </dependency>
    <dependency>
        <groupId>org.apache.avro</groupId>
        <artifactId>avro</artifactId>
        <version>1.11.3</version>
    </dependency>
</dependencies>

<repositories>
    <repository>
        <id>confluent</id>
        <url>https://packages.confluent.io/maven/</url>
    </repository>
</repositories>
```

### Avro Schema file

```json
// src/main/avro/User.avsc
{
  "type": "record",
  "name": "User",
  "namespace": "com.example.avro",
  "fields": [
    {"name": "id", "type": "long"},
    {"name": "name", "type": "string"},
    {"name": "email", "type": ["null", "string"], "default": null},
    {"name": "age", "type": "int", "default": 0},
    {
      "name": "role",
      "type": {
        "type": "enum",
        "name": "Role",
        "symbols": ["ADMIN", "MEMBER", "GUEST"]
      },
      "default": "GUEST"
    },
    {
      "name": "createdAt",
      "type": {"type": "long", "logicalType": "timestamp-millis"},
      "default": 0
    }
  ]
}
```

### Producer với Schema Registry

```java
import io.confluent.kafka.serializers.KafkaAvroSerializer;
import io.confluent.kafka.serializers.KafkaAvroSerializerConfig;
import org.apache.avro.generic.GenericData;
import org.apache.avro.generic.GenericRecord;
import org.apache.avro.Schema;
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.common.serialization.StringSerializer;

import java.io.InputStream;
import java.util.Properties;

public class AvroProducer {

    public static void main(String[] args) throws Exception {
        Properties props = new Properties();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, KafkaAvroSerializer.class);
        props.put(KafkaAvroSerializerConfig.SCHEMA_REGISTRY_URL_CONFIG, "http://localhost:8081");

        // Auto-register schema (tắt trong production!)
        props.put(KafkaAvroSerializerConfig.AUTO_REGISTER_SCHEMAS, true);

        // Idempotent
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
        props.put(ProducerConfig.ACKS_CONFIG, "all");

        // Load schema
        Schema schema;
        try (InputStream is = AvroProducer.class.getResourceAsStream("/avro/User.avsc")) {
            schema = new Schema.Parser().parse(is);
        }

        try (KafkaProducer<String, GenericRecord> producer = new KafkaProducer<>(props)) {
            for (int i = 1; i <= 10; i++) {
                GenericRecord user = new GenericData.Record(schema);
                user.put("id", (long) i);
                user.put("name", "User-" + i);
                user.put("email", "user" + i + "@example.com");
                user.put("age", 20 + i);
                user.put("role", "MEMBER");
                user.put("createdAt", System.currentTimeMillis());

                ProducerRecord<String, GenericRecord> record =
                        new ProducerRecord<>("user-events", String.valueOf(i), user);

                producer.send(record, (metadata, exception) -> {
                    if (exception != null) {
                        System.err.println("Send failed: " + exception.getMessage());
                    } else {
                        System.out.printf("Sent to partition=%d, offset=%d%n",
                                metadata.partition(), metadata.offset());
                    }
                });
            }
            producer.flush();
        }
    }
}
```

### Consumer với Schema Registry

```java
import io.confluent.kafka.serializers.KafkaAvroDeserializer;
import io.confluent.kafka.serializers.KafkaAvroDeserializerConfig;
import org.apache.avro.generic.GenericRecord;
import org.apache.kafka.clients.consumer.*;
import org.apache.kafka.common.serialization.StringDeserializer;

import java.time.Duration;
import java.util.List;
import java.util.Properties;

public class AvroConsumer {

    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "user-consumer-group");
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, KafkaAvroDeserializer.class);
        props.put(KafkaAvroDeserializerConfig.SCHEMA_REGISTRY_URL_CONFIG, "http://localhost:8081");
        props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, false); // manual commit

        try (KafkaConsumer<String, GenericRecord> consumer = new KafkaConsumer<>(props)) {
            consumer.subscribe(List.of("user-events"));

            while (true) {
                ConsumerRecords<String, GenericRecord> records =
                        consumer.poll(Duration.ofMillis(1000));

                for (ConsumerRecord<String, GenericRecord> record : records) {
                    GenericRecord user = record.value();

                    System.out.printf("offset=%d, key=%s, id=%d, name=%s, email=%s%n",
                            record.offset(),
                            record.key(),
                            user.get("id"),
                            user.get("name"),
                            user.get("email"));

                    // Schema evolution: field mới có thể null
                    Object age = user.get("age");
                    if (age != null) {
                        System.out.println("  age=" + age);
                    }
                }

                if (!records.isEmpty()) {
                    consumer.commitSync(); // commit sau khi xử lý xong batch
                }
            }
        }
    }
}
```

---

## 7. Schema Evolution thực tế

### V1 → V2: Thêm field (BACKWARD compatible)

```json
// V1
{"name": "id", "type": "long"},
{"name": "name", "type": "string"}

// V2 — thêm email với default null
{"name": "id", "type": "long"},
{"name": "name", "type": "string"},
{"name": "email", "type": ["null", "string"], "default": null}
```

**Deploy strategy:**
1. Register schema v2 → Schema Registry accept (BACKWARD ok)
2. Deploy **Consumer** mới (handle cả v1 và v2)
3. Deploy **Producer** mới (gửi v2 format)
4. Consumer đọc data v1 cũ → email = null (OK)
5. Consumer đọc data v2 mới → email có giá trị (OK)

### V2 → V3: Rename field (BREAKING!)

```json
// V2: {"name": "email", ...}
// V3: {"name": "emailAddress", ...}  ← BREAKING CHANGE!
```

**Đây là breaking change — Schema Registry sẽ reject!**

**Giải pháp:**
```json
// V3: Thêm field mới, deprecate field cũ
{"name": "email", "type": ["null", "string"], "default": null},
{"name": "emailAddress", "type": ["null", "string"], "default": null}

// Producer v3: ghi cả hai
user.put("email", emailValue);
user.put("emailAddress", emailValue);

// Consumer v3: đọc emailAddress, fallback email
String email = record.get("emailAddress") != null
    ? record.get("emailAddress").toString()
    : record.get("email") != null ? record.get("email").toString() : null;

// Sau khi tất cả consumers migrate → V4: xóa "email"
```

### Migration Plan

```
V1: {id, name}
V2: {id, name, email?}                    ← thêm mới
V3: {id, name, email?, emailAddress?}      ← dual-write giai đoạn chuyển
V4: {id, name, emailAddress?}              ← xóa field cũ (sau khi mọi consumer dùng V3+)
```

---

## 8. Subject Naming Strategies

### TopicNameStrategy (default)

```
Topic: user-events
Subject for key:   user-events-key
Subject for value: user-events-value

// Mỗi topic có schema riêng
// Cùng schema dùng cho nhiều topic → phải register nhiều lần
```

### RecordNameStrategy

```
Subject = Avro record full name
Subject: com.example.avro.User

// Cùng schema dùng cho nhiều topics
// Cần set: value.subject.name.strategy = RecordNameStrategy
```

### TopicRecordNameStrategy

```
Subject = topic + record name
Subject: user-events-com.example.avro.User

// Một topic có thể chứa nhiều schema types
```

```java
// Config
props.put("value.subject.name.strategy",
    "io.confluent.kafka.serializers.subject.TopicRecordNameStrategy");
```

---

## 9. Thực tế Production

### Tắt auto-register trong Production

```java
// Producer PRODUCTION config
props.put(KafkaAvroSerializerConfig.AUTO_REGISTER_SCHEMAS, false);
// → Schema phải được register trước (via CI/CD pipeline)
// → Ngăn producer vô tình tạo schema mới
```

### CI/CD Pipeline cho Schema

```bash
#!/bin/bash
# ci/register-schema.sh — chạy trong CI/CD

SCHEMA_REGISTRY_URL="http://schema-registry:8081"
SUBJECT="user-events-value"
SCHEMA_FILE="src/main/avro/User.avsc"

# 1. Check compatibility trước
COMPAT=$(curl -s -X POST \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data "{\"schema\": $(jq -Rs . < $SCHEMA_FILE)}" \
  "$SCHEMA_REGISTRY_URL/compatibility/subjects/$SUBJECT/versions/latest")

if echo "$COMPAT" | jq -e '.is_compatible == false' > /dev/null 2>&1; then
    echo "INCOMPATIBLE SCHEMA CHANGE! Aborting."
    exit 1
fi

# 2. Register schema
curl -X POST \
  -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data "{\"schema\": $(jq -Rs . < $SCHEMA_FILE)}" \
  "$SCHEMA_REGISTRY_URL/subjects/$SUBJECT/versions"

echo "Schema registered successfully"
```
