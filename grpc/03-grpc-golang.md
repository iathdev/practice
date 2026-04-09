# gRPC trong Go (Golang)

---

## 1. Setup Project

### Install tools

```bash
# Protobuf compiler
brew install protobuf

# Go plugins cho protoc
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest

# Đảm bảo $GOPATH/bin trong PATH
export PATH="$PATH:$(go env GOPATH)/bin"
```

### Init module

```bash
mkdir grpc-demo && cd grpc-demo
go mod init github.com/example/grpc-demo

go get google.golang.org/grpc
go get google.golang.org/protobuf
```

### Project structure

```
grpc-demo/
├── go.mod
├── go.sum
├── proto/
│   └── user/
│       └── user.proto
├── pb/                    ← generated code
│   └── user/
│       ├── user.pb.go
│       └── user_grpc.pb.go
├── server/
│   └── main.go
├── client/
│   └── main.go
└── service/
    └── user_service.go
```

---

## 2. Proto file

```protobuf
// proto/user/user.proto
syntax = "proto3";

package user;

option go_package = "github.com/example/grpc-demo/pb/user";

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

### Generate Go code

```bash
protoc --go_out=. --go_opt=paths=source_relative \
       --go-grpc_out=. --go-grpc_opt=paths=source_relative \
       proto/user/user.proto

# Output:
# pb/user/user.pb.go          ← message types
# pb/user/user_grpc.pb.go     ← service interface + client stub
```

Hoặc dùng `buf` (tool hiện đại hơn):
```bash
# buf.yaml
version: v1
breaking:
  use:
    - FILE
lint:
  use:
    - DEFAULT

# buf.gen.yaml
version: v1
plugins:
  - plugin: go
    out: pb
    opt: paths=source_relative
  - plugin: go-grpc
    out: pb
    opt: paths=source_relative

# Generate
buf generate proto
```

---

## 3. Server Implementation

### service/user_service.go

```go
package service

import (
	"context"
	"fmt"
	"io"
	"sync"
	"sync/atomic"
	"time"

	pb "github.com/example/grpc-demo/pb/user"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
	"google.golang.org/protobuf/types/known/emptypb"
	"google.golang.org/protobuf/types/known/timestamppb"
)

type UserService struct {
	pb.UnimplementedUserServiceServer // bắt buộc — forward compatibility

	mu        sync.RWMutex
	users     map[int64]*pb.User
	nextID    atomic.Int64
}

func NewUserService() *UserService {
	s := &UserService{
		users: make(map[int64]*pb.User),
	}
	s.nextID.Store(1)
	return s
}

// ==================== Unary RPCs ====================

func (s *UserService) CreateUser(ctx context.Context, req *pb.CreateUserRequest) (*pb.User, error) {
	// Validation
	if req.GetName() == "" {
		return nil, status.Errorf(codes.InvalidArgument, "name is required")
	}

	user := &pb.User{
		Id:        s.nextID.Add(1) - 1,
		Name:      req.GetName(),
		Email:     req.GetEmail(),
		Role:      req.GetRole(),
		CreatedAt: timestamppb.Now(),
	}

	s.mu.Lock()
	s.users[user.Id] = user
	s.mu.Unlock()

	return user, nil
}

func (s *UserService) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
	s.mu.RLock()
	user, ok := s.users[req.GetId()]
	s.mu.RUnlock()

	if !ok {
		return nil, status.Errorf(codes.NotFound, "user %d not found", req.GetId())
	}

	return user, nil
}

func (s *UserService) UpdateUser(ctx context.Context, req *pb.UpdateUserRequest) (*pb.User, error) {
	s.mu.Lock()
	defer s.mu.Unlock()

	user, ok := s.users[req.GetId()]
	if !ok {
		return nil, status.Errorf(codes.NotFound, "user %d not found", req.GetId())
	}

	if req.GetName() != "" {
		user.Name = req.GetName()
	}
	if req.GetEmail() != "" {
		user.Email = req.GetEmail()
	}

	return user, nil
}

func (s *UserService) DeleteUser(ctx context.Context, req *pb.GetUserRequest) (*emptypb.Empty, error) {
	s.mu.Lock()
	defer s.mu.Unlock()

	if _, ok := s.users[req.GetId()]; !ok {
		return nil, status.Errorf(codes.NotFound, "user %d not found", req.GetId())
	}

	delete(s.users, req.GetId())
	return &emptypb.Empty{}, nil
}

// ==================== Server Streaming ====================

func (s *UserService) ListUsersStream(req *pb.ListUsersRequest, stream pb.UserService_ListUsersStreamServer) error {
	s.mu.RLock()
	defer s.mu.RUnlock()

	count := 0
	limit := int(req.GetPageSize())
	if limit <= 0 {
		limit = 100
	}

	for _, user := range s.users {
		if count >= limit {
			break
		}

		// Check context (client có thể cancel)
		if err := stream.Context().Err(); err != nil {
			return status.FromContextError(err).Err()
		}

		if err := stream.Send(user); err != nil {
			return err
		}
		count++
	}

	return nil // stream tự đóng khi return
}

// ==================== Client Streaming ====================

func (s *UserService) BatchCreateUsers(stream pb.UserService_BatchCreateUsersServer) error {
	var created []*pb.User

	for {
		req, err := stream.Recv()
		if err == io.EOF {
			// Client gửi xong → trả response
			return stream.SendAndClose(&pb.ListUsersResponse{
				Users: created,
			})
		}
		if err != nil {
			return err
		}

		user := &pb.User{
			Id:        s.nextID.Add(1) - 1,
			Name:      req.GetName(),
			Email:     req.GetEmail(),
			Role:      req.GetRole(),
			CreatedAt: timestamppb.Now(),
		}

		s.mu.Lock()
		s.users[user.Id] = user
		s.mu.Unlock()

		created = append(created, user)
	}
}

// ==================== Bidirectional Streaming ====================

func (s *UserService) UserChat(stream pb.UserService_UserChatServer) error {
	for {
		msg, err := stream.Recv()
		if err == io.EOF {
			return nil
		}
		if err != nil {
			return err
		}

		fmt.Printf("[%s]: %s\n", msg.GetUser(), msg.GetContent())

		// Echo lại
		reply := &pb.ChatMessage{
			User:    "Server",
			Content: "Echo: " + msg.GetContent(),
		}
		if err := stream.Send(reply); err != nil {
			return err
		}
	}
}
```

### server/main.go

```go
package main

import (
	"fmt"
	"log"
	"net"
	"os"
	"os/signal"
	"syscall"

	"github.com/example/grpc-demo/service"
	pb "github.com/example/grpc-demo/pb/user"
	"google.golang.org/grpc"
	"google.golang.org/grpc/reflection"
)

func main() {
	port := 50051

	lis, err := net.Listen("tcp", fmt.Sprintf(":%d", port))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	grpcServer := grpc.NewServer(
		grpc.ChainUnaryInterceptor(
			loggingInterceptor,
			// thêm interceptors khác ở đây
		),
		grpc.ChainStreamInterceptor(
			streamLoggingInterceptor,
		),
	)

	pb.RegisterUserServiceServer(grpcServer, service.NewUserService())
	reflection.Register(grpcServer) // cho grpcurl

	// Graceful shutdown
	go func() {
		sigCh := make(chan os.Signal, 1)
		signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
		<-sigCh
		log.Println("Shutting down gRPC server...")
		grpcServer.GracefulStop()
	}()

	log.Printf("gRPC server started on port %d", port)
	if err := grpcServer.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

---

## 4. Client Implementation

### client/main.go

```go
package main

import (
	"context"
	"fmt"
	"io"
	"log"
	"time"

	pb "github.com/example/grpc-demo/pb/user"
	"google.golang.org/grpc"
	"google.golang.org/grpc/credentials/insecure"
	"google.golang.org/grpc/status"
)

func main() {
	// Kết nối
	conn, err := grpc.NewClient(
		"localhost:50051",
		grpc.WithTransportCredentials(insecure.NewCredentials()), // không TLS
	)
	if err != nil {
		log.Fatalf("failed to connect: %v", err)
	}
	defer conn.Close()

	client := pb.NewUserServiceClient(conn)

	// Unary
	unaryDemo(client)

	// Server Streaming
	serverStreamDemo(client)

	// Client Streaming
	clientStreamDemo(client)

	// Bidirectional Streaming
	bidiStreamDemo(client)
}

// ==================== Unary ====================

func unaryDemo(client pb.UserServiceClient) {
	ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()

	// Create
	user, err := client.CreateUser(ctx, &pb.CreateUserRequest{
		Name:  "Alice",
		Email: "alice@example.com",
		Role:  pb.UserRole_USER_ROLE_ADMIN,
	})
	if err != nil {
		handleError(err)
		return
	}
	fmt.Printf("Created user: %s (ID: %d)\n", user.GetName(), user.GetId())

	// Get
	got, err := client.GetUser(ctx, &pb.GetUserRequest{Id: user.GetId()})
	if err != nil {
		handleError(err)
		return
	}
	fmt.Printf("Got user: %s (%s)\n", got.GetName(), got.GetEmail())

	// Get non-existent → NOT_FOUND
	_, err = client.GetUser(ctx, &pb.GetUserRequest{Id: 9999})
	if err != nil {
		handleError(err) // status: NOT_FOUND
	}
}

// ==================== Server Streaming ====================

func serverStreamDemo(client pb.UserServiceClient) {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	stream, err := client.ListUsersStream(ctx, &pb.ListUsersRequest{PageSize: 10})
	if err != nil {
		handleError(err)
		return
	}

	fmt.Println("\n--- Server Streaming ---")
	for {
		user, err := stream.Recv()
		if err == io.EOF {
			break // stream kết thúc
		}
		if err != nil {
			handleError(err)
			return
		}
		fmt.Printf("  Received: %s (%s)\n", user.GetName(), user.GetEmail())
	}
}

// ==================== Client Streaming ====================

func clientStreamDemo(client pb.UserServiceClient) {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	stream, err := client.BatchCreateUsers(ctx)
	if err != nil {
		handleError(err)
		return
	}

	// Gửi nhiều requests
	names := []string{"Bob", "Charlie", "Diana"}
	for _, name := range names {
		if err := stream.Send(&pb.CreateUserRequest{
			Name:  name,
			Email: name + "@example.com",
		}); err != nil {
			log.Fatalf("send error: %v", err)
		}
	}

	// Đóng stream và nhận response
	resp, err := stream.CloseAndRecv()
	if err != nil {
		handleError(err)
		return
	}

	fmt.Printf("\n--- Client Streaming ---\nBatch created %d users\n", len(resp.GetUsers()))
	for _, u := range resp.GetUsers() {
		fmt.Printf("  - %s (ID: %d)\n", u.GetName(), u.GetId())
	}
}

// ==================== Bidirectional Streaming ====================

func bidiStreamDemo(client pb.UserServiceClient) {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	stream, err := client.UserChat(ctx)
	if err != nil {
		handleError(err)
		return
	}

	fmt.Println("\n--- Bidirectional Streaming ---")

	// Goroutine nhận messages
	done := make(chan struct{})
	go func() {
		defer close(done)
		for {
			msg, err := stream.Recv()
			if err == io.EOF {
				return
			}
			if err != nil {
				log.Printf("recv error: %v", err)
				return
			}
			fmt.Printf("  [%s]: %s\n", msg.GetUser(), msg.GetContent())
		}
	}()

	// Gửi messages
	messages := []string{"Hello!", "How are you?", "Goodbye!"}
	for _, msg := range messages {
		if err := stream.Send(&pb.ChatMessage{
			User:    "Client",
			Content: msg,
		}); err != nil {
			log.Fatalf("send error: %v", err)
		}
		time.Sleep(500 * time.Millisecond)
	}

	stream.CloseSend() // đóng gửi
	<-done             // chờ nhận xong
}

// ==================== Error Handling ====================

func handleError(err error) {
	st, ok := status.FromError(err)
	if ok {
		fmt.Printf("gRPC error: code=%s, message=%s\n", st.Code(), st.Message())
	} else {
		fmt.Printf("Non-gRPC error: %v\n", err)
	}
}
```

---

## 5. Interceptors (Middleware)

### Unary Interceptor — Logging

```go
import (
	"context"
	"log"
	"time"

	"google.golang.org/grpc"
)

// Server-side unary interceptor
func loggingInterceptor(
	ctx context.Context,
	req interface{},
	info *grpc.UnaryServerInfo,
	handler grpc.UnaryHandler,
) (interface{}, error) {
	start := time.Now()

	// Gọi handler (actual method)
	resp, err := handler(ctx, req)

	duration := time.Since(start)
	log.Printf("[gRPC] %s | %v | %s", info.FullMethod, duration, statusFromErr(err))

	return resp, err
}

// Server-side stream interceptor
func streamLoggingInterceptor(
	srv interface{},
	ss grpc.ServerStream,
	info *grpc.StreamServerInfo,
	handler grpc.StreamHandler,
) error {
	start := time.Now()
	err := handler(srv, ss)
	log.Printf("[gRPC Stream] %s | %v | %s", info.FullMethod, time.Since(start), statusFromErr(err))
	return err
}

// Đăng ký
grpcServer := grpc.NewServer(
	grpc.ChainUnaryInterceptor(loggingInterceptor, authInterceptor),
	grpc.ChainStreamInterceptor(streamLoggingInterceptor),
)
```

### Unary Interceptor — Authentication

```go
func authInterceptor(
	ctx context.Context,
	req interface{},
	info *grpc.UnaryServerInfo,
	handler grpc.UnaryHandler,
) (interface{}, error) {
	// Skip auth cho health check hoặc reflection
	if info.FullMethod == "/grpc.health.v1.Health/Check" {
		return handler(ctx, req)
	}

	// Lấy metadata (headers)
	md, ok := metadata.FromIncomingContext(ctx)
	if !ok {
		return nil, status.Errorf(codes.Unauthenticated, "missing metadata")
	}

	tokens := md.Get("authorization")
	if len(tokens) == 0 {
		return nil, status.Errorf(codes.Unauthenticated, "missing token")
	}

	token := strings.TrimPrefix(tokens[0], "Bearer ")
	userID, err := validateJWT(token)
	if err != nil {
		return nil, status.Errorf(codes.PermissionDenied, "invalid token: %v", err)
	}

	// Thêm user info vào context
	newCtx := context.WithValue(ctx, "userID", userID)
	return handler(newCtx, req)
}
```

### Client Interceptor

```go
// Client-side — tự động attach token
func authClientInterceptor(token string) grpc.UnaryClientInterceptor {
	return func(
		ctx context.Context,
		method string,
		req, reply interface{},
		cc *grpc.ClientConn,
		invoker grpc.UnaryInvoker,
		opts ...grpc.CallOption,
	) error {
		// Thêm token vào metadata
		ctx = metadata.AppendToOutgoingContext(ctx, "authorization", "Bearer "+token)
		return invoker(ctx, method, req, reply, cc, opts...)
	}
}

// Sử dụng
conn, err := grpc.NewClient(
	"localhost:50051",
	grpc.WithUnaryInterceptor(authClientInterceptor("my-jwt-token")),
	grpc.WithTransportCredentials(insecure.NewCredentials()),
)
```

---

## 6. Error Handling chi tiết

```go
import (
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/status"
	"google.golang.org/genproto/googleapis/rpc/errdetails"
)

// Server — trả lỗi với details
func (s *UserService) CreateUser(ctx context.Context, req *pb.CreateUserRequest) (*pb.User, error) {
	if req.GetName() == "" {
		// Lỗi đơn giản
		return nil, status.Errorf(codes.InvalidArgument, "name is required")
	}

	if len(req.GetName()) < 2 {
		// Lỗi với details (rich error)
		st := status.New(codes.InvalidArgument, "validation failed")
		detailed, _ := st.WithDetails(
			&errdetails.BadRequest{
				FieldViolations: []*errdetails.BadRequest_FieldViolation{
					{
						Field:       "name",
						Description: "must be at least 2 characters",
					},
				},
			},
		)
		return nil, detailed.Err()
	}

	// ... logic
	return user, nil
}

// Client — đọc error details
func handleRichError(err error) {
	st := status.Convert(err)
	fmt.Printf("Code: %s\n", st.Code())
	fmt.Printf("Message: %s\n", st.Message())

	for _, detail := range st.Details() {
		switch d := detail.(type) {
		case *errdetails.BadRequest:
			for _, violation := range d.GetFieldViolations() {
				fmt.Printf("  Field: %s, Error: %s\n", violation.GetField(), violation.GetDescription())
			}
		case *errdetails.RetryInfo:
			fmt.Printf("  Retry after: %v\n", d.GetRetryDelay().AsDuration())
		}
	}
}
```

---

## 7. TLS / mTLS

```go
// Server với TLS
creds, err := credentials.NewServerTLSFromFile("server-cert.pem", "server-key.pem")
if err != nil {
	log.Fatalf("failed to load TLS: %v", err)
}
grpcServer := grpc.NewServer(grpc.Creds(creds))

// Client với TLS
creds, err := credentials.NewClientTLSFromFile("ca-cert.pem", "")
if err != nil {
	log.Fatalf("failed to load TLS: %v", err)
}
conn, err := grpc.NewClient("localhost:50051", grpc.WithTransportCredentials(creds))

// mTLS (mutual TLS) — cả hai verify cert
// Server
cert, _ := tls.LoadX509KeyPair("server-cert.pem", "server-key.pem")
caCert, _ := os.ReadFile("ca-cert.pem")
certPool := x509.NewCertPool()
certPool.AppendCertsFromPEM(caCert)

tlsConfig := &tls.Config{
	Certificates: []tls.Certificate{cert},
	ClientAuth:   tls.RequireAndVerifyClientCert,
	ClientCAs:    certPool,
}
grpcServer := grpc.NewServer(grpc.Creds(credentials.NewTLS(tlsConfig)))
```

---

## 8. Health Check

```go
import "google.golang.org/grpc/health/grpc_health_v1"

// Server — đăng ký health service
healthServer := health.NewServer()
grpc_health_v1.RegisterHealthServer(grpcServer, healthServer)

// Set health status
healthServer.SetServingStatus("user.UserService", grpc_health_v1.HealthCheckResponse_SERVING)
healthServer.SetServingStatus("", grpc_health_v1.HealthCheckResponse_SERVING) // overall

// Client / Load Balancer check health
grpcurl -plaintext localhost:50051 grpc.health.v1.Health/Check
```

---

## 9. Testing

```go
package service_test

import (
	"context"
	"net"
	"testing"

	pb "github.com/example/grpc-demo/pb/user"
	"github.com/example/grpc-demo/service"
	"google.golang.org/grpc"
	"google.golang.org/grpc/codes"
	"google.golang.org/grpc/credentials/insecure"
	"google.golang.org/grpc/status"
	"google.golang.org/grpc/test/bufconn"
)

const bufSize = 1024 * 1024

func setupTest(t *testing.T) (pb.UserServiceClient, func()) {
	t.Helper()

	// In-memory listener (không cần network)
	lis := bufconn.Listen(bufSize)

	s := grpc.NewServer()
	pb.RegisterUserServiceServer(s, service.NewUserService())

	go func() {
		if err := s.Serve(lis); err != nil {
			t.Logf("server error: %v", err)
		}
	}()

	conn, err := grpc.NewClient(
		"passthrough:///bufnet",
		grpc.WithContextDialer(func(ctx context.Context, _ string) (net.Conn, error) {
			return lis.DialContext(ctx)
		}),
		grpc.WithTransportCredentials(insecure.NewCredentials()),
	)
	if err != nil {
		t.Fatalf("dial error: %v", err)
	}

	client := pb.NewUserServiceClient(conn)

	cleanup := func() {
		conn.Close()
		s.Stop()
		lis.Close()
	}

	return client, cleanup
}

func TestCreateUser(t *testing.T) {
	client, cleanup := setupTest(t)
	defer cleanup()

	ctx := context.Background()

	user, err := client.CreateUser(ctx, &pb.CreateUserRequest{
		Name:  "Alice",
		Email: "alice@example.com",
		Role:  pb.UserRole_USER_ROLE_ADMIN,
	})
	if err != nil {
		t.Fatalf("CreateUser failed: %v", err)
	}

	if user.GetName() != "Alice" {
		t.Errorf("expected name Alice, got %s", user.GetName())
	}
	if user.GetId() == 0 {
		t.Error("expected non-zero ID")
	}
}

func TestGetUser_NotFound(t *testing.T) {
	client, cleanup := setupTest(t)
	defer cleanup()

	_, err := client.GetUser(context.Background(), &pb.GetUserRequest{Id: 9999})
	if err == nil {
		t.Fatal("expected error, got nil")
	}

	st, ok := status.FromError(err)
	if !ok {
		t.Fatalf("expected gRPC status error, got: %v", err)
	}
	if st.Code() != codes.NotFound {
		t.Errorf("expected NOT_FOUND, got %s", st.Code())
	}
}

func TestCreateUser_Validation(t *testing.T) {
	client, cleanup := setupTest(t)
	defer cleanup()

	_, err := client.CreateUser(context.Background(), &pb.CreateUserRequest{
		Name: "", // empty — should fail
	})

	st, _ := status.FromError(err)
	if st.Code() != codes.InvalidArgument {
		t.Errorf("expected INVALID_ARGUMENT, got %s", st.Code())
	}
}

func TestBatchCreateUsers(t *testing.T) {
	client, cleanup := setupTest(t)
	defer cleanup()

	stream, err := client.BatchCreateUsers(context.Background())
	if err != nil {
		t.Fatalf("failed to start stream: %v", err)
	}

	names := []string{"A", "B", "C"}
	for _, name := range names {
		if err := stream.Send(&pb.CreateUserRequest{Name: name, Email: name + "@test.com"}); err != nil {
			t.Fatalf("send error: %v", err)
		}
	}

	resp, err := stream.CloseAndRecv()
	if err != nil {
		t.Fatalf("close error: %v", err)
	}

	if len(resp.GetUsers()) != 3 {
		t.Errorf("expected 3 users, got %d", len(resp.GetUsers()))
	}
}
```
