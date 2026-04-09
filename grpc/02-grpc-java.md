# gRPC trong Java

---

## 1. Setup Project (Maven)

### pom.xml

```xml
<properties>
    <grpc.version>1.62.2</grpc.version>
    <protobuf.version>3.25.3</protobuf.version>
</properties>

<dependencies>
    <!-- gRPC -->
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-netty-shaded</artifactId>
        <version>${grpc.version}</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-protobuf</artifactId>
        <version>${grpc.version}</version>
    </dependency>
    <dependency>
        <groupId>io.grpc</groupId>
        <artifactId>grpc-stub</artifactId>
        <version>${grpc.version}</version>
    </dependency>

    <!-- Protobuf -->
    <dependency>
        <groupId>com.google.protobuf</groupId>
        <artifactId>protobuf-java</artifactId>
        <version>${protobuf.version}</version>
    </dependency>

    <!-- Annotation cho generated code -->
    <dependency>
        <groupId>javax.annotation</groupId>
        <artifactId>javax.annotation-api</artifactId>
        <version>1.3.2</version>
    </dependency>
</dependencies>

<build>
    <extensions>
        <extension>
            <groupId>kr.motd.maven</groupId>
            <artifactId>os-maven-plugin</artifactId>
            <version>1.7.1</version>
        </extension>
    </extensions>
    <plugins>
        <plugin>
            <groupId>org.xolstice.maven.plugins</groupId>
            <artifactId>protobuf-maven-plugin</artifactId>
            <version>0.6.1</version>
            <configuration>
                <protocArtifact>
                    com.google.protobuf:protoc:${protobuf.version}:exe:${os.detected.classifier}
                </protocArtifact>
                <pluginId>grpc-java</pluginId>
                <pluginArtifact>
                    io.grpc:protoc-gen-grpc-java:${grpc.version}:exe:${os.detected.classifier}
                </pluginArtifact>
            </configuration>
            <executions>
                <execution>
                    <goals>
                        <goal>compile</goal>
                        <goal>compile-custom</goal>
                    </goals>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### Project structure:
```
src/
├── main/
│   ├── java/com/example/grpc/
│   │   ├── server/
│   │   │   ├── GrpcServer.java
│   │   │   └── UserServiceImpl.java
│   │   └── client/
│   │       └── UserClient.java
│   └── proto/
│       └── user.proto          ← proto files ở đây
└── test/
```

### Generate code:
```bash
mvn compile
# Generated code → target/generated-sources/protobuf/
```

---

## 2. Proto file

```protobuf
// src/main/proto/user.proto
syntax = "proto3";

package user;

option java_package = "com.example.grpc.user";
option java_multiple_files = true;

import "google/protobuf/timestamp.proto";
import "google/protobuf/empty.proto";

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
  UserRole role = 4;
  google.protobuf.Timestamp created_at = 5;
}

enum UserRole {
  USER_ROLE_UNSPECIFIED = 0;
  USER_ROLE_ADMIN = 1;
  USER_ROLE_MEMBER = 2;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
  UserRole role = 3;
}

message GetUserRequest {
  int64 id = 1;
}

message ListUsersRequest {
  int32 page_size = 1;
  string page_token = 2;
}

message ListUsersResponse {
  repeated User users = 1;
  string next_page_token = 2;
}

message UpdateUserRequest {
  int64 id = 1;
  string name = 2;
  string email = 3;
}

service UserService {
  // Unary
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc GetUser(GetUserRequest) returns (User);
  rpc UpdateUser(UpdateUserRequest) returns (User);
  rpc DeleteUser(GetUserRequest) returns (google.protobuf.Empty);

  // Server streaming
  rpc ListUsersStream(ListUsersRequest) returns (stream User);

  // Client streaming
  rpc BatchCreateUsers(stream CreateUserRequest) returns (ListUsersResponse);

  // Bidirectional streaming
  rpc UserChat(stream ChatMessage) returns (stream ChatMessage);
}

message ChatMessage {
  string user = 1;
  string content = 2;
}
```

---

## 3. Server Implementation

### UserServiceImpl.java

```java
package com.example.grpc.server;

import com.example.grpc.user.*;
import com.google.protobuf.Empty;
import com.google.protobuf.Timestamp;
import io.grpc.Status;
import io.grpc.stub.StreamObserver;

import java.time.Instant;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicLong;

public class UserServiceImpl extends UserServiceGrpc.UserServiceImplBase {

    private final Map<Long, User> users = new ConcurrentHashMap<>();
    private final AtomicLong idGenerator = new AtomicLong(1);

    // ==================== Unary RPCs ====================

    @Override
    public void createUser(CreateUserRequest request, StreamObserver<User> responseObserver) {
        // Validation
        if (request.getName().isEmpty()) {
            responseObserver.onError(
                Status.INVALID_ARGUMENT
                    .withDescription("Name is required")
                    .asRuntimeException()
            );
            return;
        }

        Instant now = Instant.now();
        User user = User.newBuilder()
                .setId(idGenerator.getAndIncrement())
                .setName(request.getName())
                .setEmail(request.getEmail())
                .setRole(request.getRole())
                .setCreatedAt(Timestamp.newBuilder()
                        .setSeconds(now.getEpochSecond())
                        .setNanos(now.getNano())
                        .build())
                .build();

        users.put(user.getId(), user);

        responseObserver.onNext(user);      // gửi response
        responseObserver.onCompleted();      // kết thúc
    }

    @Override
    public void getUser(GetUserRequest request, StreamObserver<User> responseObserver) {
        User user = users.get(request.getId());

        if (user == null) {
            responseObserver.onError(
                Status.NOT_FOUND
                    .withDescription("User not found: " + request.getId())
                    .asRuntimeException()
            );
            return;
        }

        responseObserver.onNext(user);
        responseObserver.onCompleted();
    }

    @Override
    public void updateUser(UpdateUserRequest request, StreamObserver<User> responseObserver) {
        User existing = users.get(request.getId());
        if (existing == null) {
            responseObserver.onError(
                Status.NOT_FOUND
                    .withDescription("User not found: " + request.getId())
                    .asRuntimeException()
            );
            return;
        }

        User updated = existing.toBuilder()
                .setName(request.getName().isEmpty() ? existing.getName() : request.getName())
                .setEmail(request.getEmail().isEmpty() ? existing.getEmail() : request.getEmail())
                .build();

        users.put(updated.getId(), updated);

        responseObserver.onNext(updated);
        responseObserver.onCompleted();
    }

    @Override
    public void deleteUser(GetUserRequest request, StreamObserver<Empty> responseObserver) {
        User removed = users.remove(request.getId());
        if (removed == null) {
            responseObserver.onError(
                Status.NOT_FOUND
                    .withDescription("User not found: " + request.getId())
                    .asRuntimeException()
            );
            return;
        }

        responseObserver.onNext(Empty.getDefaultInstance());
        responseObserver.onCompleted();
    }

    // ==================== Server Streaming ====================

    @Override
    public void listUsersStream(ListUsersRequest request,
                                StreamObserver<User> responseObserver) {
        // Stream từng user về client
        users.values().stream()
                .limit(request.getPageSize() > 0 ? request.getPageSize() : 100)
                .forEach(responseObserver::onNext);

        responseObserver.onCompleted();
    }

    // ==================== Client Streaming ====================

    @Override
    public StreamObserver<CreateUserRequest> batchCreateUsers(
            StreamObserver<ListUsersResponse> responseObserver) {

        return new StreamObserver<>() {
            private final ListUsersResponse.Builder responseBuilder =
                    ListUsersResponse.newBuilder();

            @Override
            public void onNext(CreateUserRequest request) {
                // Nhận từng request từ client
                User user = User.newBuilder()
                        .setId(idGenerator.getAndIncrement())
                        .setName(request.getName())
                        .setEmail(request.getEmail())
                        .setRole(request.getRole())
                        .build();
                users.put(user.getId(), user);
                responseBuilder.addUsers(user);
            }

            @Override
            public void onError(Throwable t) {
                System.err.println("Client streaming error: " + t.getMessage());
            }

            @Override
            public void onCompleted() {
                // Client gửi xong → trả response
                responseObserver.onNext(responseBuilder.build());
                responseObserver.onCompleted();
            }
        };
    }

    // ==================== Bidirectional Streaming ====================

    @Override
    public StreamObserver<ChatMessage> userChat(
            StreamObserver<ChatMessage> responseObserver) {

        return new StreamObserver<>() {
            @Override
            public void onNext(ChatMessage message) {
                // Nhận message từ client, echo lại
                ChatMessage reply = ChatMessage.newBuilder()
                        .setUser("Server")
                        .setContent("Echo: " + message.getContent())
                        .build();
                responseObserver.onNext(reply);
            }

            @Override
            public void onError(Throwable t) {
                System.err.println("Chat error: " + t.getMessage());
            }

            @Override
            public void onCompleted() {
                responseObserver.onCompleted();
            }
        };
    }
}
```

### GrpcServer.java

```java
package com.example.grpc.server;

import io.grpc.Server;
import io.grpc.ServerBuilder;
import io.grpc.protobuf.services.ProtoReflectionService;

import java.io.IOException;

public class GrpcServer {

    private final int port;
    private Server server;

    public GrpcServer(int port) {
        this.port = port;
    }

    public void start() throws IOException {
        server = ServerBuilder.forPort(port)
                .addService(new UserServiceImpl())
                .addService(ProtoReflectionService.newInstance()) // cho grpcurl/grpcui
                .build()
                .start();

        System.out.println("gRPC Server started on port " + port);

        // Shutdown hook
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            System.out.println("Shutting down gRPC server...");
            GrpcServer.this.stop();
        }));
    }

    public void stop() {
        if (server != null) {
            server.shutdown();
        }
    }

    public void blockUntilShutdown() throws InterruptedException {
        if (server != null) {
            server.awaitTermination();
        }
    }

    public static void main(String[] args) throws Exception {
        GrpcServer server = new GrpcServer(50051);
        server.start();
        server.blockUntilShutdown();
    }
}
```

---

## 4. Client Implementation

### UserClient.java

```java
package com.example.grpc.client;

import com.example.grpc.user.*;
import io.grpc.ManagedChannel;
import io.grpc.ManagedChannelBuilder;
import io.grpc.StatusRuntimeException;
import io.grpc.stub.StreamObserver;

import java.util.Iterator;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

public class UserClient {

    private final ManagedChannel channel;
    private final UserServiceGrpc.UserServiceBlockingStub blockingStub;   // sync
    private final UserServiceGrpc.UserServiceStub asyncStub;              // async

    public UserClient(String host, int port) {
        channel = ManagedChannelBuilder.forAddress(host, port)
                .usePlaintext()  // không TLS (dev only!)
                .build();

        blockingStub = UserServiceGrpc.newBlockingStub(channel);
        asyncStub = UserServiceGrpc.newStub(channel);
    }

    public void shutdown() throws InterruptedException {
        channel.shutdown().awaitTermination(5, TimeUnit.SECONDS);
    }

    // ==================== Unary (Blocking) ====================

    public User createUser(String name, String email) {
        CreateUserRequest request = CreateUserRequest.newBuilder()
                .setName(name)
                .setEmail(email)
                .setRole(UserRole.USER_ROLE_MEMBER)
                .build();

        try {
            User user = blockingStub
                    .withDeadlineAfter(5, TimeUnit.SECONDS)  // timeout 5s
                    .createUser(request);
            System.out.println("Created user: " + user.getName() + " (ID: " + user.getId() + ")");
            return user;
        } catch (StatusRuntimeException e) {
            System.err.println("RPC failed: " + e.getStatus());
            return null;
        }
    }

    public User getUser(long id) {
        try {
            return blockingStub
                    .withDeadlineAfter(5, TimeUnit.SECONDS)
                    .getUser(GetUserRequest.newBuilder().setId(id).build());
        } catch (StatusRuntimeException e) {
            System.err.println("GetUser failed: " + e.getStatus());
            return null;
        }
    }

    // ==================== Server Streaming (Blocking) ====================

    public void listUsersStream(int pageSize) {
        ListUsersRequest request = ListUsersRequest.newBuilder()
                .setPageSize(pageSize)
                .build();

        try {
            // Iterator-based blocking streaming
            Iterator<User> users = blockingStub
                    .withDeadlineAfter(10, TimeUnit.SECONDS)
                    .listUsersStream(request);

            while (users.hasNext()) {
                User user = users.next();
                System.out.println("Received user: " + user.getName());
            }
        } catch (StatusRuntimeException e) {
            System.err.println("ListUsersStream failed: " + e.getStatus());
        }
    }

    // ==================== Client Streaming (Async) ====================

    public void batchCreateUsers(String... names) throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(1);

        StreamObserver<CreateUserRequest> requestObserver =
                asyncStub.batchCreateUsers(new StreamObserver<>() {
                    @Override
                    public void onNext(ListUsersResponse response) {
                        System.out.println("Batch created " + response.getUsersCount() + " users");
                        response.getUsersList().forEach(u ->
                                System.out.println("  - " + u.getName() + " (ID: " + u.getId() + ")")
                        );
                    }

                    @Override
                    public void onError(Throwable t) {
                        System.err.println("BatchCreate error: " + t.getMessage());
                        latch.countDown();
                    }

                    @Override
                    public void onCompleted() {
                        System.out.println("Batch create completed");
                        latch.countDown();
                    }
                });

        // Gửi từng request
        for (String name : names) {
            requestObserver.onNext(
                    CreateUserRequest.newBuilder()
                            .setName(name)
                            .setEmail(name.toLowerCase() + "@example.com")
                            .build()
            );
        }
        requestObserver.onCompleted(); // báo hiệu gửi xong

        latch.await(10, TimeUnit.SECONDS);
    }

    // ==================== Bidirectional Streaming ====================

    public void chat(String... messages) throws InterruptedException {
        CountDownLatch latch = new CountDownLatch(1);

        StreamObserver<ChatMessage> requestObserver =
                asyncStub.userChat(new StreamObserver<>() {
                    @Override
                    public void onNext(ChatMessage message) {
                        System.out.println("[" + message.getUser() + "]: " + message.getContent());
                    }

                    @Override
                    public void onError(Throwable t) {
                        System.err.println("Chat error: " + t.getMessage());
                        latch.countDown();
                    }

                    @Override
                    public void onCompleted() {
                        System.out.println("Chat ended");
                        latch.countDown();
                    }
                });

        for (String msg : messages) {
            requestObserver.onNext(
                    ChatMessage.newBuilder()
                            .setUser("Client")
                            .setContent(msg)
                            .build()
            );
            Thread.sleep(500); // simulate delay
        }
        requestObserver.onCompleted();

        latch.await(10, TimeUnit.SECONDS);
    }

    // ==================== Main ====================

    public static void main(String[] args) throws Exception {
        UserClient client = new UserClient("localhost", 50051);

        try {
            // Unary
            User user1 = client.createUser("Alice", "alice@example.com");
            User user2 = client.createUser("Bob", "bob@example.com");
            client.getUser(1);

            // Server Streaming
            client.listUsersStream(10);

            // Client Streaming
            client.batchCreateUsers("Charlie", "Diana", "Eve");

            // Bidirectional Streaming
            client.chat("Hello!", "How are you?", "Goodbye!");
        } finally {
            client.shutdown();
        }
    }
}
```

---

## 5. Interceptors (Middleware)

### Server Interceptor — Logging

```java
package com.example.grpc.server;

import io.grpc.*;

public class LoggingInterceptor implements ServerInterceptor {

    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {

        String methodName = call.getMethodDescriptor().getFullMethodName();
        long startTime = System.currentTimeMillis();

        System.out.println("[gRPC] → " + methodName);

        // Wrap response để log khi hoàn thành
        ServerCall<ReqT, RespT> wrappedCall = new ForwardingServerCall
                .SimpleForwardingServerCall<>(call) {
            @Override
            public void close(Status status, Metadata trailers) {
                long duration = System.currentTimeMillis() - startTime;
                System.out.println("[gRPC] ← " + methodName
                        + " | status=" + status.getCode()
                        + " | " + duration + "ms");
                super.close(status, trailers);
            }
        };

        return next.startCall(wrappedCall, headers);
    }
}

// Đăng ký interceptor
Server server = ServerBuilder.forPort(50051)
        .addService(new UserServiceImpl())
        .intercept(new LoggingInterceptor())       // global interceptor
        .build();
```

### Server Interceptor — Authentication

```java
public class AuthInterceptor implements ServerInterceptor {

    private static final Metadata.Key<String> AUTH_KEY =
            Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER);

    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {

        String token = headers.get(AUTH_KEY);

        if (token == null || !token.startsWith("Bearer ")) {
            call.close(
                Status.UNAUTHENTICATED.withDescription("Missing or invalid token"),
                new Metadata()
            );
            return new ServerCall.Listener<>() {}; // no-op listener
        }

        // Validate token
        String jwt = token.substring(7);
        if (!validateToken(jwt)) {
            call.close(
                Status.PERMISSION_DENIED.withDescription("Invalid token"),
                new Metadata()
            );
            return new ServerCall.Listener<>() {};
        }

        // Thêm user info vào Context
        Context ctx = Context.current().withValue(USER_CONTEXT_KEY, extractUser(jwt));
        return Contexts.interceptCall(ctx, call, headers, next);
    }
}
```

### Client Interceptor — Attach Token

```java
public class AuthClientInterceptor implements ClientInterceptor {

    private final String token;

    public AuthClientInterceptor(String token) {
        this.token = token;
    }

    @Override
    public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(
            MethodDescriptor<ReqT, RespT> method,
            CallOptions callOptions,
            Channel next) {

        return new ForwardingClientCall.SimpleForwardingClientCall<>(
                next.newCall(method, callOptions)) {
            @Override
            public void start(Listener<RespT> responseListener, Metadata headers) {
                headers.put(
                    Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER),
                    "Bearer " + token
                );
                super.start(responseListener, headers);
            }
        };
    }
}

// Sử dụng
ManagedChannel channel = ManagedChannelBuilder.forAddress("localhost", 50051)
        .usePlaintext()
        .intercept(new AuthClientInterceptor("my-jwt-token"))
        .build();
```

---

## 6. Error Handling

```java
// Server — trả lỗi chi tiết
@Override
public void getUser(GetUserRequest request, StreamObserver<User> responseObserver) {
    if (request.getId() <= 0) {
        responseObserver.onError(
            Status.INVALID_ARGUMENT
                .withDescription("ID must be positive")
                .asRuntimeException()
        );
        return;
    }

    User user = users.get(request.getId());
    if (user == null) {
        // Có thể attach metadata cho lỗi chi tiết
        Metadata metadata = new Metadata();
        metadata.put(
            Metadata.Key.of("error-code", Metadata.ASCII_STRING_MARSHALLER),
            "USER_NOT_FOUND"
        );
        responseObserver.onError(
            Status.NOT_FOUND
                .withDescription("User " + request.getId() + " not found")
                .asRuntimeException(metadata)
        );
        return;
    }

    responseObserver.onNext(user);
    responseObserver.onCompleted();
}

// Client — xử lý lỗi
try {
    User user = blockingStub.getUser(request);
} catch (StatusRuntimeException e) {
    Status status = e.getStatus();
    switch (status.getCode()) {
        case NOT_FOUND:
            System.out.println("User not found: " + status.getDescription());
            break;
        case INVALID_ARGUMENT:
            System.out.println("Bad request: " + status.getDescription());
            break;
        case DEADLINE_EXCEEDED:
            System.out.println("Request timed out");
            break;
        default:
            System.out.println("RPC error: " + status);
    }
}
```

---

## 7. Spring Boot + gRPC

### Dependency (grpc-spring-boot-starter)

```xml
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-server-spring-boot-starter</artifactId>
    <version>3.0.0.RELEASE</version>
</dependency>
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-client-spring-boot-starter</artifactId>
    <version>3.0.0.RELEASE</version>
</dependency>
```

### application.yml
```yaml
grpc:
  server:
    port: 50051
  client:
    user-service:
      address: static://localhost:50051
      negotiationType: plaintext
```

### Server
```java
@GrpcService  // tự động đăng ký với gRPC server
public class UserGrpcService extends UserServiceGrpc.UserServiceImplBase {

    @Autowired
    private UserRepository userRepository; // inject Spring beans bình thường

    @Override
    public void getUser(GetUserRequest request, StreamObserver<User> responseObserver) {
        // Dùng Spring beans, transaction, etc.
        var entity = userRepository.findById(request.getId())
                .orElseThrow(() -> Status.NOT_FOUND.asRuntimeException());

        responseObserver.onNext(toProto(entity));
        responseObserver.onCompleted();
    }
}
```

### Client
```java
@Service
public class UserClientService {

    @GrpcClient("user-service")  // tên trong config
    private UserServiceGrpc.UserServiceBlockingStub userStub;

    public User getUser(long id) {
        return userStub.getUser(
            GetUserRequest.newBuilder().setId(id).build()
        );
    }
}
```

---

## 8. Testing

```java
import io.grpc.inprocess.InProcessChannelBuilder;
import io.grpc.inprocess.InProcessServerBuilder;
import io.grpc.testing.GrpcCleanupRule;
import org.junit.Rule;
import org.junit.Test;

public class UserServiceTest {

    @Rule
    public final GrpcCleanupRule grpcCleanup = new GrpcCleanupRule();

    @Test
    public void testCreateUser() throws Exception {
        // In-process server (không cần network)
        String serverName = InProcessServerBuilder.generateName();

        grpcCleanup.register(
            InProcessServerBuilder.forName(serverName)
                .directExecutor()
                .addService(new UserServiceImpl())
                .build()
                .start()
        );

        // In-process channel
        ManagedChannel channel = grpcCleanup.register(
            InProcessChannelBuilder.forName(serverName)
                .directExecutor()
                .build()
        );

        UserServiceGrpc.UserServiceBlockingStub stub =
                UserServiceGrpc.newBlockingStub(channel);

        // Test
        User user = stub.createUser(
            CreateUserRequest.newBuilder()
                .setName("TestUser")
                .setEmail("test@example.com")
                .build()
        );

        assertEquals("TestUser", user.getName());
        assertEquals(1L, user.getId());
    }

    @Test(expected = StatusRuntimeException.class)
    public void testGetUserNotFound() throws Exception {
        // ... setup ...
        stub.getUser(GetUserRequest.newBuilder().setId(999).build());
        // Expect NOT_FOUND status
    }
}
```
