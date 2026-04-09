# gRPC Advanced Topics

---

## 1. Java & Go Interop — Cross-language Communication

gRPC cho phép Java server phục vụ Go client và ngược lại — chỉ cần cùng `.proto` file.

```
┌─────────────────┐       ┌─────────────────┐
│  Go Client      │       │  Java Server    │
│  pb.NewClient() │──────→│  @GrpcService   │
│                 │  gRPC │                 │
│  protoc-gen-go  │       │  protoc-gen-java│
└─────────────────┘       └─────────────────┘
        ↑                          ↑
        └──── Cùng user.proto ─────┘
```

**Yêu cầu:**
- Cùng `.proto` file (hoặc compatible version)
- Proto file ở shared repo hoặc proto registry (Buf Schema Registry)

### Monorepo approach:
```
shared-protos/
├── user/user.proto
├── order/order.proto
└── payment/payment.proto

java-service/    → generate từ shared-protos
go-service/      → generate từ shared-protos
```

### Proto Registry (Buf):
```bash
# Push proto lên Buf Schema Registry
buf push

# Java pull
buf generate buf.build/myorg/myprotos --template buf.gen.java.yaml

# Go pull
buf generate buf.build/myorg/myprotos --template buf.gen.go.yaml
```

---

## 2. Deadline Propagation

Khi service A gọi service B gọi service C, deadline phải lan truyền tự động.

```
Client (deadline=5s)
  → Service A (còn 4.8s)
    → Service B (còn 4.5s)
      → Service C (còn 4.2s) ← nếu quá → DEADLINE_EXCEEDED
```

### Go

```go
// Client set deadline
ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()
resp, err := client.GetUser(ctx, req)

// Service A nhận ctx có deadline → forward luôn
func (s *ServiceA) Process(ctx context.Context, req *pb.Request) (*pb.Response, error) {
    // ctx đã có deadline từ client!
    // Gọi Service B — tự động pass deadline
    resp, err := s.serviceBClient.DoWork(ctx, workReq)
    
    // Check deadline trước khi làm việc nặng
    if ctx.Err() == context.DeadlineExceeded {
        return nil, status.Errorf(codes.DeadlineExceeded, "deadline exceeded")
    }
    
    return resp, err
}
```

### Java

```java
// Client
User user = blockingStub
    .withDeadlineAfter(5, TimeUnit.SECONDS)
    .getUser(request);

// Server — forward deadline
@Override
public void process(Request request, StreamObserver<Response> responseObserver) {
    // Deadline tự động propagate qua Context
    Context deadline = Context.current();
    
    // Gọi service khác — deadline tự động forward
    Response resp = serviceBStub.doWork(workRequest);
    
    // Check deadline
    if (Context.current().getDeadline().isExpired()) {
        responseObserver.onError(
            Status.DEADLINE_EXCEEDED.asRuntimeException()
        );
        return;
    }
}
```

---

## 3. Retry & Hedging

### Go — Manual Retry

```go
func retryUnaryInterceptor(maxRetries int) grpc.UnaryClientInterceptor {
    return func(
        ctx context.Context,
        method string,
        req, reply interface{},
        cc *grpc.ClientConn,
        invoker grpc.UnaryInvoker,
        opts ...grpc.CallOption,
    ) error {
        var lastErr error
        for i := 0; i <= maxRetries; i++ {
            lastErr = invoker(ctx, method, req, reply, cc, opts...)
            if lastErr == nil {
                return nil
            }

            st, ok := status.FromError(lastErr)
            if !ok {
                return lastErr
            }

            // Chỉ retry cho retryable codes
            switch st.Code() {
            case codes.Unavailable, codes.ResourceExhausted, codes.Aborted:
                backoff := time.Duration(i+1) * 100 * time.Millisecond
                select {
                case <-ctx.Done():
                    return ctx.Err()
                case <-time.After(backoff):
                }
            default:
                return lastErr // non-retryable
            }
        }
        return lastErr
    }
}
```

### gRPC Service Config — Declarative Retry

```go
// Service config cho retry tự động
serviceConfig := `{
    "methodConfig": [{
        "name": [{"service": "user.UserService"}],
        "retryPolicy": {
            "maxAttempts": 4,
            "initialBackoff": "0.1s",
            "maxBackoff": "1s",
            "backoffMultiplier": 2,
            "retryableStatusCodes": ["UNAVAILABLE", "RESOURCE_EXHAUSTED"]
        }
    }]
}`

conn, err := grpc.NewClient(
    "localhost:50051",
    grpc.WithDefaultServiceConfig(serviceConfig),
    grpc.WithTransportCredentials(insecure.NewCredentials()),
)
```

---

## 4. Load Balancing

### Client-side (Go)

```go
import (
    "google.golang.org/grpc/resolver"
    _ "google.golang.org/grpc/balancer/roundrobin" // import roundrobin balancer
)

// DNS-based discovery + round-robin
conn, err := grpc.NewClient(
    "dns:///my-service.default.svc.cluster.local:50051",
    grpc.WithDefaultServiceConfig(`{"loadBalancingPolicy":"round_robin"}`),
    grpc.WithTransportCredentials(insecure.NewCredentials()),
)

// Static addresses
serviceConfig := `{
    "loadBalancingPolicy": "round_robin"
}`

resolver.SetDefaultScheme("dns")
conn, err := grpc.NewClient(
    "dns:///my-service:50051",
    grpc.WithDefaultServiceConfig(serviceConfig),
    grpc.WithTransportCredentials(insecure.NewCredentials()),
)
```

### Proxy-based (Envoy)

```yaml
# envoy.yaml
static_resources:
  listeners:
    - address:
        socket_address: { address: 0.0.0.0, port_value: 8080 }
      filter_chains:
        - filters:
            - name: envoy.filters.network.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                codec_type: AUTO
                stat_prefix: ingress_http
                route_config:
                  virtual_hosts:
                    - name: grpc_service
                      domains: ["*"]
                      routes:
                        - match: { prefix: "/" }
                          route: { cluster: grpc_backend }
                http_filters:
                  - name: envoy.filters.http.router
  clusters:
    - name: grpc_backend
      type: STRICT_DNS
      lb_policy: ROUND_ROBIN
      typed_extension_protocol_options:
        envoy.extensions.upstreams.http.v3.HttpProtocolOptions:
          "@type": type.googleapis.com/envoy.extensions.upstreams.http.v3.HttpProtocolOptions
          explicit_http_config:
            http2_protocol_options: {}
      load_assignment:
        cluster_name: grpc_backend
        endpoints:
          - lb_endpoints:
              - endpoint:
                  address:
                    socket_address: { address: server1, port_value: 50051 }
              - endpoint:
                  address:
                    socket_address: { address: server2, port_value: 50051 }
```

---

## 5. gRPC-Gateway (REST + gRPC)

gRPC-Gateway tạo REST proxy tự động từ `.proto` file — cho phép expose cả REST lẫn gRPC từ cùng backend.

```
Browser (REST)                   Microservice (gRPC)
GET /api/users/1   ──→ gRPC-Gateway ──→ GetUser(id=1)
                   ←── JSON response ←── User{...}
```

### Proto với HTTP annotations

```protobuf
import "google/api/annotations.proto";

service UserService {
  rpc GetUser(GetUserRequest) returns (User) {
    option (google.api.http) = {
      get: "/api/v1/users/{id}"
    };
  }
  
  rpc CreateUser(CreateUserRequest) returns (User) {
    option (google.api.http) = {
      post: "/api/v1/users"
      body: "*"
    };
  }
  
  rpc UpdateUser(UpdateUserRequest) returns (User) {
    option (google.api.http) = {
      put: "/api/v1/users/{id}"
      body: "*"
    };
  }
  
  rpc DeleteUser(GetUserRequest) returns (google.protobuf.Empty) {
    option (google.api.http) = {
      delete: "/api/v1/users/{id}"
    };
  }
}
```

### Go Gateway Server

```go
import (
    "github.com/grpc-ecosystem/grpc-gateway/v2/runtime"
    pb "github.com/example/grpc-demo/pb/user"
)

func runGateway() error {
    ctx := context.Background()
    mux := runtime.NewServeMux()
    
    opts := []grpc.DialOption{grpc.WithTransportCredentials(insecure.NewCredentials())}
    
    // Register gRPC service handlers
    err := pb.RegisterUserServiceHandlerFromEndpoint(ctx, mux, "localhost:50051", opts)
    if err != nil {
        return err
    }
    
    // Start HTTP server
    log.Println("REST gateway on :8080")
    return http.ListenAndServe(":8080", mux)
}

// Giờ có thể gọi:
// curl http://localhost:8080/api/v1/users/1
// curl -X POST http://localhost:8080/api/v1/users -d '{"name":"Alice"}'
```

---

## 6. Protobuf Best Practices

### Backward Compatibility

```protobuf
// KHÔNG xóa field — đánh dấu reserved
message User {
  int64 id = 1;
  string name = 2;
  // string old_field = 3; ← đã xóa
  reserved 3;
  reserved "old_field";
  string email = 4;
}

// KHÔNG thay đổi field number
// KHÔNG thay đổi type (int32 → string)
// CÓ THỂ thêm field mới (new field number)
// CÓ THỂ rename field (number giữ nguyên)
```

### Naming Conventions

```protobuf
// Package: lowercase, dot-separated
package company.project.v1;

// Message: PascalCase
message UserProfile { ... }

// Field: snake_case
string first_name = 1;
repeated string phone_numbers = 2;

// Enum: SCREAMING_SNAKE_CASE, prefix với enum name
enum UserStatus {
  USER_STATUS_UNSPECIFIED = 0;
  USER_STATUS_ACTIVE = 1;
  USER_STATUS_INACTIVE = 2;
}

// Service: PascalCase
service UserService { ... }

// RPC: PascalCase (verb + noun)
rpc GetUser(...) returns (...);
rpc ListUsers(...) returns (...);
rpc CreateUser(...) returns (...);
rpc UpdateUser(...) returns (...);
rpc DeleteUser(...) returns (...);
```

### API Versioning

```protobuf
// Package versioning
package company.user.v1;  // v1
package company.user.v2;  // v2 — breaking changes

// Service versioning
service UserServiceV1 { ... }
service UserServiceV2 { ... }
```

---

## 7. Monitoring & Observability

### Go — Prometheus Metrics

```go
import grpc_prometheus "github.com/grpc-ecosystem/go-grpc-prometheus"

// Server
grpcServer := grpc.NewServer(
    grpc.ChainUnaryInterceptor(
        grpc_prometheus.UnaryServerInterceptor,
    ),
    grpc.ChainStreamInterceptor(
        grpc_prometheus.StreamServerInterceptor,
    ),
)
grpc_prometheus.Register(grpcServer)

// Expose metrics endpoint
http.Handle("/metrics", promhttp.Handler())
go http.ListenAndServe(":9090", nil)
```

### Distributed Tracing (OpenTelemetry)

```go
import (
    "go.opentelemetry.io/contrib/instrumentation/google.golang.org/grpc/otelgrpc"
)

// Server
grpcServer := grpc.NewServer(
    grpc.StatsHandler(otelgrpc.NewServerHandler()),
)

// Client
conn, err := grpc.NewClient(
    "localhost:50051",
    grpc.WithStatsHandler(otelgrpc.NewClientHandler()),
    grpc.WithTransportCredentials(insecure.NewCredentials()),
)
```

---

## 8. Useful Tools

### grpcurl — gRPC equivalent of curl

```bash
# Install
brew install grpcurl
go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest

# List services (cần server reflection)
grpcurl -plaintext localhost:50051 list

# Describe service
grpcurl -plaintext localhost:50051 describe user.UserService

# Unary call
grpcurl -plaintext -d '{"name":"Alice","email":"alice@example.com"}' \
    localhost:50051 user.UserService/CreateUser

# Get user
grpcurl -plaintext -d '{"id":1}' localhost:50051 user.UserService/GetUser

# Server streaming
grpcurl -plaintext -d '{"page_size":10}' \
    localhost:50051 user.UserService/ListUsersStream
```

### grpcui — Web UI cho gRPC

```bash
# Install
go install github.com/fullstorydev/grpcui/cmd/grpcui@latest

# Start web UI
grpcui -plaintext localhost:50051
# Opens browser → interactive gRPC client
```

### buf — Modern protobuf tooling

```bash
# Install
brew install bufbuild/buf/buf

# Lint proto files
buf lint

# Check breaking changes
buf breaking --against '.git#branch=main'

# Generate code
buf generate

# Push to Buf Schema Registry
buf push
```

---

## 9. gRPC trong Kubernetes

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: user-service
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: user-service
          image: myregistry/user-service:latest
          ports:
            - containerPort: 50051
              name: grpc
          readinessProbe:
            grpc:
              port: 50051
            initialDelaySeconds: 5
          livenessProbe:
            grpc:
              port: 50051
            periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: user-service
spec:
  selector:
    app: user-service
  ports:
    - port: 50051
      targetPort: 50051
      name: grpc
  # headless service cho client-side load balancing
  clusterIP: None
```

### Ingress (gRPC qua NGINX)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grpc-ingress
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "GRPC"
spec:
  rules:
    - host: grpc.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: user-service
                port:
                  number: 50051
  tls:
    - hosts:
        - grpc.example.com
      secretName: grpc-tls
```
