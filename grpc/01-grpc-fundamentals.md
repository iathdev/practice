# gRPC Fundamentals

---

## 1. gRPC là gì?

**gRPC** (gRPC Remote Procedure Call) là framework RPC open-source do Google phát triển, cho phép client gọi method trên server ở máy khác như thể gọi local function.

```
Client (Java)                          Server (Go)
─────────────────────────────────────────────────────
userService.GetUser(req)  ──HTTP/2──→  GetUser(ctx, req)
                          ←─response─  return &User{...}
```

**Đặc điểm chính:**
- **Protocol Buffers (protobuf):** Format serialize nhị phân, nhỏ và nhanh hơn JSON
- **HTTP/2:** Multiplexing, bidirectional streaming, header compression
- **Language agnostic:** Generate code cho Java, Go, Python, C++, etc.
- **Strongly typed:** Contract rõ ràng qua `.proto` file

---

## 2. So sánh gRPC vs REST

| | gRPC | REST |
|--|------|------|
| Protocol | HTTP/2 | HTTP/1.1 (thường) |
| Data format | **Protobuf (binary)** | JSON (text) |
| Contract | `.proto` file (strict) | OpenAPI/Swagger (loose) |
| Streaming | **Bidirectional** | Hạn chế (SSE, WebSocket) |
| Code generation | **Built-in** | Cần thêm tool |
| Browser support | Hạn chế (cần grpc-web) | Native |
| Performance | **Nhanh hơn 2-10x** | Chậm hơn |
| Human-readable | Không (binary) | Có (JSON) |
| Caching | Không native | HTTP caching |

**Khi nào dùng gRPC:**
- Microservices communication (service-to-service)
- Low latency, high throughput
- Streaming data (real-time updates)
- Polyglot systems (nhiều ngôn ngữ)

**Khi nào dùng REST:**
- Public-facing API (browser clients)
- Simple CRUD
- Cần human-readable, dễ debug
- Caching quan trọng

---

## 3. Protocol Buffers (Protobuf)

**Protobuf** là IDL (Interface Definition Language) và format serialization của gRPC.

### File `.proto` cơ bản:

```protobuf
syntax = "proto3";

package user;

option java_package = "com.example.grpc.user";
option java_multiple_files = true;
option go_package = "github.com/example/grpc/user";

// Message definitions
message User {
  int64 id = 1;           // field number (không phải giá trị)
  string name = 2;
  string email = 3;
  UserRole role = 4;
  repeated string tags = 5;        // list
  optional string bio = 6;         // nullable
  map<string, string> metadata = 7; // map
  google.protobuf.Timestamp created_at = 8;
}

enum UserRole {
  USER_ROLE_UNSPECIFIED = 0;  // proto3 bắt buộc 0 = default
  USER_ROLE_ADMIN = 1;
  USER_ROLE_MEMBER = 2;
}

// Service definition
service UserService {
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  rpc ListUsers(ListUsersRequest) returns (stream User);        // server streaming
  rpc UploadUsers(stream User) returns (UploadResponse);        // client streaming
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);    // bidirectional
}

message GetUserRequest {
  int64 id = 1;
}

message GetUserResponse {
  User user = 1;
}

message ListUsersRequest {
  int32 page_size = 1;
  string page_token = 2;
}

message UploadResponse {
  int32 count = 1;
}

message ChatMessage {
  string from = 1;
  string content = 2;
}
```

### Field numbers:
- **1-15:** Dùng 1 byte → dùng cho fields phổ biến
- **16-2047:** Dùng 2 bytes
- **Không bao giờ reuse** field numbers đã remove (dùng `reserved`)

```protobuf
message User {
  reserved 4, 6 to 8;         // giữ chỗ field numbers đã xóa
  reserved "old_field_name";   // giữ chỗ tên đã xóa
}
```

### Scalar types:

| Protobuf | Java | Go | Default |
|----------|------|----|---------|
| int32 | int | int32 | 0 |
| int64 | long | int64 | 0 |
| float | float | float32 | 0.0 |
| double | double | float64 | 0.0 |
| bool | boolean | bool | false |
| string | String | string | "" |
| bytes | ByteString | []byte | empty |

---

## 4. Bốn loại RPC

### 4.1 Unary RPC (Request-Response)
```
Client ──request──→ Server
Client ←─response── Server
```
Giống REST call thông thường. Một request, một response.

### 4.2 Server Streaming RPC
```
Client ──request──→ Server
Client ←─stream──── Server (nhiều messages)
Client ←─stream──── Server
Client ←─stream──── Server
```
Client gửi một request, server trả về stream of responses.
**Use case:** Real-time stock prices, log tailing, list kết quả lớn.

### 4.3 Client Streaming RPC
```
Client ──stream──→ Server (nhiều messages)
Client ──stream──→ Server
Client ──stream──→ Server
Client ←─response── Server
```
Client gửi stream of messages, server trả về một response.
**Use case:** File upload, batch insert, sensor data.

### 4.4 Bidirectional Streaming RPC
```
Client ──stream──⇄── Server
Client ──stream──⇄── Server
Client ──stream──⇄── Server
```
Cả hai đều gửi stream độc lập.
**Use case:** Chat, multiplayer game, collaborative editing.

---

## 5. HTTP/2 Features cho gRPC

**Multiplexing:** Nhiều RPC calls trên cùng một TCP connection
```
Connection
├── Stream 1: GetUser()
├── Stream 2: ListOrders()
└── Stream 3: UpdateProfile()
```

**Header compression (HPACK):** Giảm overhead cho metadata/headers

**Flow control:** Ngăn sender gửi quá nhanh cho receiver

**Server push:** Server có thể gửi data mà client chưa request

---

## 6. gRPC Lifecycle

```
1. Client gọi stub method
2. Server nhận notification: metadata (headers), method name, deadline
3. Server có thể gửi initial metadata ngay (hoặc đợi)
4. Server nhận request message(s)
5. Server xử lý và gửi response message(s)
6. Server gửi status (OK/error) + trailing metadata
7. Client nhận response(s) và status
```

**Deadline/Timeout:**
```
Client set deadline = 5 giây
→ Server PHẢI trả lời trong 5 giây
→ Nếu quá → client nhận DEADLINE_EXCEEDED
→ Server cũng nên check deadline và dừng xử lý nếu hết
```

---

## 7. gRPC Status Codes

| Code | Tên | Mô tả |
|------|-----|--------|
| 0 | OK | Thành công |
| 1 | CANCELLED | Client hủy |
| 2 | UNKNOWN | Lỗi không xác định |
| 3 | INVALID_ARGUMENT | Tham số sai |
| 4 | DEADLINE_EXCEEDED | Hết timeout |
| 5 | NOT_FOUND | Không tìm thấy |
| 7 | PERMISSION_DENIED | Không có quyền |
| 8 | RESOURCE_EXHAUSTED | Rate limit / quota |
| 12 | UNIMPLEMENTED | Method chưa implement |
| 13 | INTERNAL | Lỗi server nội bộ |
| 14 | UNAVAILABLE | Service tạm thời không khả dụng |
| 16 | UNAUTHENTICATED | Chưa xác thực |

---

## 8. Interceptors / Middleware

Interceptors cho phép xen xử lý trước/sau mỗi RPC call (giống middleware trong REST).

**Use cases:**
- Logging
- Authentication/Authorization
- Metrics/Monitoring
- Rate limiting
- Error handling
- Request/Response validation

```
Client → [Client Interceptor] → Network → [Server Interceptor] → Handler
                                                                      ↓
Client ← [Client Interceptor] ← Network ← [Server Interceptor] ← Response
```

---

## 9. Load Balancing

**Client-side load balancing:**
```
Client (có danh sách servers)
├──→ Server A
├──→ Server B
└──→ Server C
```
Client tự chọn server. Algorithms: round-robin, weighted, least-connections.

**Proxy-based load balancing:**
```
Client ──→ Load Balancer (Envoy, NGINX) ──→ Server A/B/C
```
Proxy nhận request và forward đến backend.

**Service mesh (Istio, Linkerd):**
Sidecar proxy tự động xử lý load balancing, retry, circuit breaking.

---

## 10. Security

**TLS/mTLS:**
```
Client ←──TLS──→ Server  (server cert only)
Client ←─mTLS──→ Server  (cả hai verify cert)
```

**Token-based auth:**
- Gửi JWT/API key qua gRPC metadata (tương đương HTTP headers)
- Server interceptor validate token

**Per-RPC credentials:**
- Attach credentials cho từng call
- Channel credentials vs Call credentials
