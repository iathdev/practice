# WebSocket trong Offline-First Exam — Real-time Sync & Server Push

> Bổ sung kiến trúc WebSocket cho hệ thống thi offline-first. WebSocket giải quyết 2 bài toán mà REST polling không tối ưu: (1) Sync đáp án real-time thay vì polling 10s, (2) Server push thông báo xuống client (hết giờ, force submit, admin announcement).

---

## Mục lục

1. [Tại sao cần WebSocket?](#1-tại-sao-cần-websocket)
2. [Kiến trúc kết hợp REST + WebSocket](#2-kiến-trúc-kết-hợp-rest--websocket)
3. [WebSocket Message Protocol](#3-websocket-message-protocol)
4. [Xử lý Offline — Fallback Strategy](#4-xử-lý-offline--fallback-strategy)
5. [Cài đặt WebSocket — Java (Spring Boot)](#5-cài-đặt-websocket--java-spring-boot)
6. [Cài đặt WebSocket — Golang (Gorilla)](#6-cài-đặt-websocket--golang-gorilla)
7. [Scale WebSocket cho 5,000+ connections](#7-scale-websocket-cho-5000-connections)
8. [Interview Quick Reference](#8-interview-quick-reference)

---

## 1. Tại sao cần WebSocket?

### Bài toán hiện tại với REST Polling

```
Hiện tại (REST polling 10s):
  Client ──── POST /answers/batch ───▶ Server     (mỗi 10 giây)
  Client ◀─── 200 OK ──────────────── Server

Vấn đề:
  1. Server KHÔNG THỂ chủ động gửi message xuống client
     → Hết giờ trên server? Client không biết ngay (phải đợi lần poll tiếp)
     → Admin muốn force-submit? Phải đợi client poll

  2. Polling 10s = 10 giây delay tối đa
     → Timer sync: server timer và client timer lệch đến 10 giây
     → 5,000 clients × 6 req/phút = 30,000 requests/phút (lãng phí nếu không có thay đổi)

  3. Không biết trạng thái connection real-time
     → Client offline 5 phút → server không biết (chỉ biết khi không nhận poll)
```

### WebSocket giải quyết gì?

```
Với WebSocket (persistent connection):
  Client ◀══════ BIDIRECTIONAL ══════▶ Server

  1. SERVER PUSH:
     → Hết giờ → server push "TIME_UP" ngay lập tức (0 delay)
     → Admin force-submit → push ngay
     → Cảnh báo gian lận → push ngay
     → Timer correction → push server time mỗi 30s

  2. EFFICIENT SYNC:
     → Chỉ gửi khi có data mới (không polling empty)
     → Latency < 100ms (vs 10s polling)
     → 1 connection thay vì nhiều HTTP requests

  3. CONNECTION AWARENESS:
     → Server biết ngay client disconnect (onClose event)
     → Heartbeat/ping-pong → biết client còn sống
     → Hữu ích cho anti-cheat: "thí sinh mất kết nối lúc 14:32"
```

### Khi nào KHÔNG nên dùng WebSocket?

```
WebSocket KHÔNG thay thế REST hoàn toàn trong hệ thống thi:

DÙNG REST cho:
  - Download đề thi (payload lớn, 1 lần)
  - Authentication (stateless, standard)
  - Submit bài thi cuối cùng (quan trọng, cần HTTP status code rõ ràng)
  - Fallback khi WebSocket fail

DÙNG WebSocket cho:
  - Sync đáp án real-time (frequent, small payload)
  - Server push (timer, notifications, force actions)
  - Connection monitoring (heartbeat)
  - Anti-cheat event streaming
```

---

## 2. Kiến trúc kết hợp REST + WebSocket

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          EXAM CLIENT (Browser)                          │
│                                                                         │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌──────────────┐  │
│  │  Exam Engine │  │  Answer     │  │  Timer      │  │  Connection  │  │
│  │  (UI + Logic)│  │  Store      │  │  Manager    │  │  Manager     │  │
│  │              │  │  (IndexedDB)│  │  (local)    │  │              │  │
│  └──────────────┘  └──────┬──────┘  └──────┬──────┘  │ ┌──────────┐│  │
│                           │                │         │ │ WebSocket ││  │
│                           │                │         │ │ Client    ││  │
│                           │                │         │ ├──────────┤│  │
│                           │                │         │ │ REST     ││  │
│                           │                │         │ │ Fallback ││  │
│                           │                │         │ └──────────┘│  │
│                           │                │         └──────┬──────┘  │
└───────────────────────────┼────────────────┼────────────────┼─────────┘
                            │                │                │
                ════════════╪════════════════╪════════════════╪══════════
                            │                │                │
                            │                │    ┌───────────▼──────────┐
                            │                │    │   WebSocket Server   │
                            │                │    │   (ws://exam/ws)     │
                            │                │    │                      │
                            │                │    │  - Answer sync       │
                            │                │    │  - Timer correction  │
                            │                │    │  - Server push       │
                            │                │    │  - Heartbeat         │
                            │                │    └───────────┬──────────┘
                            │                │                │
                            │                │    ┌───────────▼──────────┐
                            │                │    │   REST API Server    │
                            │                │    │   (Fallback + CRUD)  │
                            │                │    └───────────┬──────────┘
                            │                │                │
                            │                │    ┌───────────▼──────────┐
                            │                │    │     PostgreSQL       │
                            │                │    └──────────────────────┘
```

### Connection Manager — Chiến lược kết nối

```
Ưu tiên: WebSocket > REST Polling

1. App khởi động → thử kết nối WebSocket
2. WebSocket OK → dùng WS cho sync + nhận push
3. WebSocket FAIL → fallback REST polling (mỗi 10s)
4. WebSocket RECONNECT → exponential backoff (1s, 2s, 4s, 8s, max 30s)
5. Offline → queue local, resume khi online (bất kể WS hay REST)
```

---

## 3. WebSocket Message Protocol

### Message Format

```json
{
  "type": "ANSWER_SYNC",
  "id": "msg-uuid-v4",
  "timestamp": "2026-04-06T14:32:01Z",
  "payload": { ... }
}
```

### Message Types — Client → Server

| Type | Mô tả | Payload |
|------|--------|---------|
| `ANSWER_SYNC` | Sync đáp án lên server | `{ answers: [{question_id, answer, sequence}] }` |
| `HEARTBEAT` | Client còn sống | `{ client_time }` |
| `EXAM_EVENT` | Anti-cheat event | `{ event_type, action, metadata }` |

### Message Types — Server → Client

| Type | Mô tả | Payload |
|------|--------|---------|
| `SYNC_ACK` | Xác nhận đã nhận answers | `{ synced_sequences: [1,2,3], server_time }` |
| `TIMER_CORRECTION` | Điều chỉnh timer | `{ remaining_seconds, server_time }` |
| `FORCE_SUBMIT` | Bắt buộc nộp bài | `{ reason: "time_up" \| "admin" }` |
| `WARNING` | Cảnh báo từ anti-cheat | `{ message, severity }` |
| `ANNOUNCEMENT` | Thông báo từ admin | `{ message }` |
| `PONG` | Phản hồi heartbeat | `{ server_time }` |
| `SESSION_INVALID` | Session hết hạn / bị void | `{ reason }` |

### Ví dụ luồng message

```
Client                                    Server
  │                                          │
  │──── ANSWER_SYNC {q1: "B", seq: 1} ─────▶│
  │◀─── SYNC_ACK {synced: [1]} ─────────────│
  │                                          │
  │◀─── TIMER_CORRECTION {remaining: 3540} ─│  (server push mỗi 30s)
  │                                          │
  │──── HEARTBEAT {client_time} ────────────▶│
  │◀─── PONG {server_time} ─────────────────│
  │                                          │
  │──── ANSWER_SYNC {q5: "A", seq: 2} ─────▶│
  │◀─── SYNC_ACK {synced: [2]} ─────────────│
  │                                          │
  │◀─── WARNING {msg: "Tab switch detected"}│  (anti-cheat push)
  │                                          │
  │◀─── FORCE_SUBMIT {reason: "time_up"} ───│  (hết giờ)
  │                                          │
```

---

## 4. Xử lý Offline — Fallback Strategy

```
                  ┌──────────────┐
                  │ Connection   │
                  │ Manager      │
                  └──────┬───────┘
                         │
              ┌──────────┼──────────┐
              │                     │
     ┌────────▼────────┐  ┌────────▼────────┐
     │  WebSocket       │  │  REST Fallback  │
     │  (primary)       │  │  (secondary)    │
     └────────┬─────────┘  └────────┬────────┘
              │                     │
              │  Cả 2 fail?        │
              └─────────┬──────────┘
                        │
               ┌────────▼────────┐
               │  Offline Queue  │
               │  (IndexedDB)    │
               │  → Resume khi   │
               │    online       │
               └─────────────────┘
```

### Client JavaScript — Connection Manager

```javascript
class ConnectionManager {
  constructor(examId, token) {
    this.examId = examId;
    this.token = token;
    this.ws = null;
    this.mode = 'disconnected'; // 'websocket' | 'polling' | 'disconnected'
    this.reconnectAttempts = 0;
    this.maxReconnectDelay = 30000;
  }

  async connect() {
    try {
      await this.connectWebSocket();
      this.mode = 'websocket';
      this.reconnectAttempts = 0;
    } catch (err) {
      console.warn('WebSocket failed, falling back to REST polling');
      this.mode = 'polling';
      this.startPolling();
    }
  }

  connectWebSocket() {
    return new Promise((resolve, reject) => {
      const url = `wss://exam-api.example.com/ws?token=${this.token}`;
      this.ws = new WebSocket(url);

      this.ws.onopen = () => {
        console.log('WebSocket connected');
        resolve();
      };

      this.ws.onmessage = (event) => {
        const msg = JSON.parse(event.data);
        this.handleServerMessage(msg);
      };

      this.ws.onclose = (event) => {
        console.log('WebSocket closed:', event.code);
        this.mode = 'disconnected';
        this.scheduleReconnect();
      };

      this.ws.onerror = (err) => reject(err);

      // Timeout 5s nếu không connect được
      setTimeout(() => {
        if (this.ws.readyState !== WebSocket.OPEN) {
          this.ws.close();
          reject(new Error('WebSocket timeout'));
        }
      }, 5000);
    });
  }

  // Gửi answers qua WS hoặc REST
  async syncAnswers(answers) {
    if (this.mode === 'websocket' && this.ws?.readyState === WebSocket.OPEN) {
      this.ws.send(JSON.stringify({
        type: 'ANSWER_SYNC',
        id: crypto.randomUUID(),
        payload: { answers }
      }));
    } else if (this.mode === 'polling') {
      await fetch(`/api/exams/${this.examId}/answers/batch`, {
        method: 'POST',
        headers: { 'Authorization': `Bearer ${this.token}` },
        body: JSON.stringify({ answers })
      });
    } else {
      // Offline → queue trong IndexedDB
      await this.queueForLater(answers);
    }
  }

  handleServerMessage(msg) {
    switch (msg.type) {
      case 'SYNC_ACK':
        this.onSyncAck(msg.payload.synced_sequences);
        break;
      case 'TIMER_CORRECTION':
        timerManager.correct(msg.payload.remaining_seconds);
        break;
      case 'FORCE_SUBMIT':
        examEngine.forceSubmit(msg.payload.reason);
        break;
      case 'WARNING':
        examEngine.showWarning(msg.payload.message);
        break;
      case 'PONG':
        this.updateClockDrift(msg.payload.server_time);
        break;
    }
  }

  scheduleReconnect() {
    const delay = Math.min(
      1000 * Math.pow(2, this.reconnectAttempts),
      this.maxReconnectDelay
    );
    this.reconnectAttempts++;
    setTimeout(() => this.connect(), delay);
  }

  // Heartbeat mỗi 15 giây
  startHeartbeat() {
    setInterval(() => {
      if (this.ws?.readyState === WebSocket.OPEN) {
        this.ws.send(JSON.stringify({
          type: 'HEARTBEAT',
          payload: { client_time: Date.now() }
        }));
      }
    }, 15000);
  }
}
```

---

## 5. Cài đặt WebSocket — Java (Spring Boot)

### 5.1. Dependencies

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
</dependency>
```

### 5.2. WebSocket Configuration

```java
@Configuration
@EnableWebSocket
public class WebSocketConfig implements WebSocketConfigurer {

    @Autowired
    private ExamWebSocketHandler examHandler;

    @Autowired
    private ExamHandshakeInterceptor authInterceptor;

    @Override
    public void registerWebSocketHandlers(WebSocketHandlerRegistry registry) {
        registry.addHandler(examHandler, "/ws/exam")
                .addInterceptors(authInterceptor)
                .setAllowedOrigins("https://exam.example.com");
    }
}
```

### 5.3. Authentication Interceptor — Xác thực trước khi kết nối

```java
@Component
public class ExamHandshakeInterceptor implements HandshakeInterceptor {

    @Autowired
    private JwtTokenProvider jwtProvider;

    @Override
    public boolean beforeHandshake(ServerHttpRequest request,
                                    ServerHttpResponse response,
                                    WebSocketHandler wsHandler,
                                    Map<String, Object> attributes) {
        // Lấy token từ query param: ws://host/ws/exam?token=xxx
        String token = extractToken(request);
        if (token == null) return false;

        try {
            Claims claims = jwtProvider.validate(token);
            attributes.put("userId", claims.getSubject());
            attributes.put("examId", claims.get("examId", String.class));
            attributes.put("sessionId", claims.get("sessionId", String.class));
            return true;  // Cho phép kết nối
        } catch (JwtException e) {
            return false; // Từ chối kết nối
        }
    }

    @Override
    public void afterHandshake(ServerHttpRequest request,
                                ServerHttpResponse response,
                                WebSocketHandler wsHandler,
                                Exception exception) {
        // Không cần xử lý gì thêm
    }

    private String extractToken(ServerHttpRequest request) {
        String query = request.getURI().getQuery();
        if (query == null) return null;
        return Arrays.stream(query.split("&"))
                .filter(p -> p.startsWith("token="))
                .map(p -> p.substring(6))
                .findFirst()
                .orElse(null);
    }
}
```

### 5.4. WebSocket Handler — Xử lý message

```java
@Component
public class ExamWebSocketHandler extends TextWebSocketHandler {

    // Lưu trữ active connections: sessionId -> WebSocketSession
    private final ConcurrentHashMap<String, WebSocketSession> sessions = new ConcurrentHashMap<>();

    @Autowired
    private AnswerService answerService;

    @Autowired
    private ObjectMapper objectMapper;

    // === KẾT NỐI ===
    @Override
    public void afterConnectionEstablished(WebSocketSession session) {
        String sessionId = getSessionId(session);
        sessions.put(sessionId, session);
        log.info("WebSocket connected: userId={}, examId={}, sessionId={}",
                getUserId(session), getExamId(session), sessionId);
    }

    // === NHẬN MESSAGE TỪ CLIENT ===
    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) {
        try {
            JsonNode msg = objectMapper.readTree(message.getPayload());
            String type = msg.get("type").asText();

            switch (type) {
                case "ANSWER_SYNC" -> handleAnswerSync(session, msg.get("payload"));
                case "HEARTBEAT"   -> handleHeartbeat(session, msg.get("payload"));
                case "EXAM_EVENT"  -> handleExamEvent(session, msg.get("payload"));
                default -> log.warn("Unknown message type: {}", type);
            }
        } catch (Exception e) {
            log.error("Error handling WS message", e);
            sendError(session, "Invalid message format");
        }
    }

    // === XỬ LÝ SYNC ĐÁP ÁN ===
    private void handleAnswerSync(WebSocketSession session, JsonNode payload) {
        String userId = getUserId(session);
        String examId = getExamId(session);

        List<AnswerDto> answers = objectMapper.convertValue(
                payload.get("answers"),
                new TypeReference<>() {}
        );

        // Lưu vào database (idempotent bằng sync_id)
        List<Integer> syncedSequences = answerService.batchSync(userId, examId, answers);

        // Gửi ACK về client
        sendMessage(session, Map.of(
                "type", "SYNC_ACK",
                "payload", Map.of(
                        "synced_sequences", syncedSequences,
                        "server_time", Instant.now().toEpochMilli()
                )
        ));
    }

    // === HEARTBEAT ===
    private void handleHeartbeat(WebSocketSession session, JsonNode payload) {
        sendMessage(session, Map.of(
                "type", "PONG",
                "payload", Map.of("server_time", Instant.now().toEpochMilli())
        ));
    }

    // === SERVER PUSH: Gửi message xuống client cụ thể ===
    public void pushToUser(String sessionId, Map<String, Object> message) {
        WebSocketSession session = sessions.get(sessionId);
        if (session != null && session.isOpen()) {
            sendMessage(session, message);
        }
    }

    // === SERVER PUSH: Broadcast cho tất cả thí sinh trong 1 exam ===
    public void broadcastToExam(String examId, Map<String, Object> message) {
        sessions.values().stream()
                .filter(s -> examId.equals(getExamId(s)))
                .forEach(s -> sendMessage(s, message));
    }

    // === NGẮT KẾT NỐI ===
    @Override
    public void afterConnectionClosed(WebSocketSession session, CloseStatus status) {
        String sessionId = getSessionId(session);
        sessions.remove(sessionId);
        log.info("WebSocket disconnected: sessionId={}, status={}", sessionId, status);
    }

    // === HELPER METHODS ===
    private void sendMessage(WebSocketSession session, Object payload) {
        try {
            String json = objectMapper.writeValueAsString(payload);
            session.sendMessage(new TextMessage(json));
        } catch (IOException e) {
            log.error("Failed to send WS message", e);
        }
    }

    private String getUserId(WebSocketSession session) {
        return (String) session.getAttributes().get("userId");
    }

    private String getExamId(WebSocketSession session) {
        return (String) session.getAttributes().get("examId");
    }

    private String getSessionId(WebSocketSession session) {
        return (String) session.getAttributes().get("sessionId");
    }

    private void sendError(WebSocketSession session, String message) {
        sendMessage(session, Map.of("type", "ERROR", "payload", Map.of("message", message)));
    }
}
```

### 5.5. Timer Correction Service — Push timer mỗi 30s

```java
@Service
public class TimerCorrectionService {

    @Autowired
    private ExamWebSocketHandler wsHandler;

    @Autowired
    private ExamSessionRepository sessionRepo;

    // Push timer correction mỗi 30 giây cho tất cả active sessions
    @Scheduled(fixedRate = 30000)
    public void pushTimerCorrections() {
        List<ExamSession> activeSessions = sessionRepo.findByStatus("IN_PROGRESS");

        for (ExamSession session : activeSessions) {
            long remainingSeconds = calculateRemaining(session);

            if (remainingSeconds <= 0) {
                // Hết giờ → force submit
                wsHandler.pushToUser(session.getSessionId(), Map.of(
                        "type", "FORCE_SUBMIT",
                        "payload", Map.of("reason", "time_up")
                ));
            } else {
                // Điều chỉnh timer
                wsHandler.pushToUser(session.getSessionId(), Map.of(
                        "type", "TIMER_CORRECTION",
                        "payload", Map.of(
                                "remaining_seconds", remainingSeconds,
                                "server_time", Instant.now().toEpochMilli()
                        )
                ));
            }
        }
    }

    private long calculateRemaining(ExamSession session) {
        Instant deadline = session.getStartedAt().plus(session.getDurationMinutes(), ChronoUnit.MINUTES);
        return ChronoUnit.SECONDS.between(Instant.now(), deadline);
    }
}
```

---

## 6. Cài đặt WebSocket — Golang (Gorilla WebSocket)

### 6.1. Dependencies

```bash
go get github.com/gorilla/websocket
```

### 6.2. Connection Hub — Quản lý tất cả connections

```go
package ws

import (
    "sync"
    "log"
)

// Hub quản lý tất cả WebSocket connections
type Hub struct {
    mu       sync.RWMutex
    clients  map[string]*Client   // sessionID -> Client
    examMap  map[string][]string  // examID -> []sessionID
}

func NewHub() *Hub {
    return &Hub{
        clients: make(map[string]*Client),
        examMap: make(map[string][]string),
    }
}

// Đăng ký client mới
func (h *Hub) Register(client *Client) {
    h.mu.Lock()
    defer h.mu.Unlock()

    h.clients[client.SessionID] = client
    h.examMap[client.ExamID] = append(h.examMap[client.ExamID], client.SessionID)
    log.Printf("Client registered: user=%s exam=%s session=%s",
        client.UserID, client.ExamID, client.SessionID)
}

// Hủy đăng ký client
func (h *Hub) Unregister(client *Client) {
    h.mu.Lock()
    defer h.mu.Unlock()

    delete(h.clients, client.SessionID)

    // Xóa khỏi examMap
    sessions := h.examMap[client.ExamID]
    for i, sid := range sessions {
        if sid == client.SessionID {
            h.examMap[client.ExamID] = append(sessions[:i], sessions[i+1:]...)
            break
        }
    }
    log.Printf("Client unregistered: session=%s", client.SessionID)
}

// Push message tới 1 client cụ thể
func (h *Hub) SendToSession(sessionID string, msg *Message) {
    h.mu.RLock()
    client, ok := h.clients[sessionID]
    h.mu.RUnlock()

    if ok {
        client.Send(msg)
    }
}

// Broadcast tới tất cả thí sinh trong 1 exam
func (h *Hub) BroadcastToExam(examID string, msg *Message) {
    h.mu.RLock()
    sessions := h.examMap[examID]
    h.mu.RUnlock()

    for _, sid := range sessions {
        h.SendToSession(sid, msg)
    }
}
```

### 6.3. Client — Đại diện 1 WebSocket connection

```go
package ws

import (
    "encoding/json"
    "log"
    "time"
    "github.com/gorilla/websocket"
)

const (
    writeWait      = 10 * time.Second
    pongWait       = 60 * time.Second
    pingPeriod     = (pongWait * 9) / 10
    maxMessageSize = 8192
)

// Message format giữa client-server
type Message struct {
    Type      string          `json:"type"`
    ID        string          `json:"id,omitempty"`
    Timestamp time.Time       `json:"timestamp"`
    Payload   json.RawMessage `json:"payload"`
}

// Client đại diện 1 WebSocket connection
type Client struct {
    UserID    string
    ExamID    string
    SessionID string
    conn      *websocket.Conn
    hub       *Hub
    send      chan *Message
}

func NewClient(conn *websocket.Conn, hub *Hub, userID, examID, sessionID string) *Client {
    return &Client{
        UserID:    userID,
        ExamID:    examID,
        SessionID: sessionID,
        conn:      conn,
        hub:       hub,
        send:      make(chan *Message, 64), // buffer 64 messages
    }
}

// Send message vào channel (non-blocking)
func (c *Client) Send(msg *Message) {
    select {
    case c.send <- msg:
    default:
        // Channel đầy → client chậm → đóng connection
        log.Printf("Client send buffer full, closing: session=%s", c.SessionID)
        c.hub.Unregister(c)
        c.conn.Close()
    }
}

// ReadPump — đọc message từ client (goroutine riêng)
func (c *Client) ReadPump(handler MessageHandler) {
    defer func() {
        c.hub.Unregister(c)
        c.conn.Close()
    }()

    c.conn.SetReadLimit(maxMessageSize)
    c.conn.SetReadDeadline(time.Now().Add(pongWait))
    c.conn.SetPongHandler(func(string) error {
        c.conn.SetReadDeadline(time.Now().Add(pongWait))
        return nil
    })

    for {
        _, rawMsg, err := c.conn.ReadMessage()
        if err != nil {
            if websocket.IsUnexpectedCloseError(err,
                websocket.CloseGoingAway, websocket.CloseNormalClosure) {
                log.Printf("WebSocket error: session=%s err=%v", c.SessionID, err)
            }
            break
        }

        var msg Message
        if err := json.Unmarshal(rawMsg, &msg); err != nil {
            log.Printf("Invalid message: session=%s err=%v", c.SessionID, err)
            continue
        }

        handler.HandleMessage(c, &msg)
    }
}

// WritePump — gửi message tới client (goroutine riêng)
func (c *Client) WritePump() {
    ticker := time.NewTicker(pingPeriod)
    defer func() {
        ticker.Stop()
        c.conn.Close()
    }()

    for {
        select {
        case msg, ok := <-c.send:
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if !ok {
                // Channel đã đóng
                c.conn.WriteMessage(websocket.CloseMessage, []byte{})
                return
            }

            data, err := json.Marshal(msg)
            if err != nil {
                log.Printf("Marshal error: %v", err)
                return
            }

            if err := c.conn.WriteMessage(websocket.TextMessage, data); err != nil {
                return
            }

        case <-ticker.C:
            // Ping client → kiểm tra connection còn sống
            c.conn.SetWriteDeadline(time.Now().Add(writeWait))
            if err := c.conn.WriteMessage(websocket.PingMessage, nil); err != nil {
                return
            }
        }
    }
}
```

### 6.4. HTTP Handler — Upgrade HTTP → WebSocket

```go
package ws

import (
    "encoding/json"
    "net/http"
    "time"
    "github.com/gorilla/websocket"
)

var upgrader = websocket.Upgrader{
    ReadBufferSize:  1024,
    WriteBufferSize: 1024,
    CheckOrigin: func(r *http.Request) bool {
        origin := r.Header.Get("Origin")
        return origin == "https://exam.example.com"
    },
}

// MessageHandler interface để xử lý business logic
type MessageHandler interface {
    HandleMessage(client *Client, msg *Message)
}

// ServeWs — HTTP handler upgrade lên WebSocket
func ServeWs(hub *Hub, handler MessageHandler, jwtValidator JWTValidator) http.HandlerFunc {
    return func(w http.ResponseWriter, r *http.Request) {
        // 1. Xác thực token
        token := r.URL.Query().Get("token")
        claims, err := jwtValidator.Validate(token)
        if err != nil {
            http.Error(w, "Unauthorized", http.StatusUnauthorized)
            return
        }

        // 2. Upgrade HTTP → WebSocket
        conn, err := upgrader.Upgrade(w, r, nil)
        if err != nil {
            log.Printf("Upgrade failed: %v", err)
            return
        }

        // 3. Tạo client và đăng ký vào Hub
        client := NewClient(conn, hub, claims.UserID, claims.ExamID, claims.SessionID)
        hub.Register(client)

        // 4. Chạy 2 goroutine: đọc và ghi
        go client.ReadPump(handler)
        go client.WritePump()
    }
}
```

### 6.5. Exam Message Handler — Business Logic

```go
package ws

import (
    "encoding/json"
    "time"
    "log"
)

type ExamMessageHandler struct {
    answerService AnswerService
    hub           *Hub
}

func (h *ExamMessageHandler) HandleMessage(client *Client, msg *Message) {
    switch msg.Type {
    case "ANSWER_SYNC":
        h.handleAnswerSync(client, msg)
    case "HEARTBEAT":
        h.handleHeartbeat(client)
    case "EXAM_EVENT":
        h.handleExamEvent(client, msg)
    default:
        log.Printf("Unknown message type: %s from session=%s", msg.Type, client.SessionID)
    }
}

func (h *ExamMessageHandler) handleAnswerSync(client *Client, msg *Message) {
    var payload struct {
        Answers []AnswerDto `json:"answers"`
    }
    json.Unmarshal(msg.Payload, &payload)

    syncedSeqs := h.answerService.BatchSync(client.UserID, client.ExamID, payload.Answers)

    client.Send(&Message{
        Type:      "SYNC_ACK",
        Timestamp: time.Now(),
        Payload: mustMarshal(map[string]interface{}{
            "synced_sequences": syncedSeqs,
            "server_time":      time.Now().UnixMilli(),
        }),
    })
}

func (h *ExamMessageHandler) handleHeartbeat(client *Client) {
    client.Send(&Message{
        Type:      "PONG",
        Timestamp: time.Now(),
        Payload: mustMarshal(map[string]interface{}{
            "server_time": time.Now().UnixMilli(),
        }),
    })
}

func mustMarshal(v interface{}) json.RawMessage {
    data, _ := json.Marshal(v)
    return data
}
```

### 6.6. Main — Kết nối tất cả

```go
package main

import (
    "log"
    "net/http"
    "myapp/ws"
)

func main() {
    hub := ws.NewHub()
    handler := &ws.ExamMessageHandler{
        answerService: answerService,
        hub:           hub,
    }

    http.HandleFunc("/ws/exam", ws.ServeWs(hub, handler, jwtValidator))
    http.HandleFunc("/api/exams/", examRESTHandler) // REST vẫn giữ

    log.Println("Server starting on :8080")
    log.Fatal(http.ListenAndServe(":8080", nil))
}
```

---

## 7. Scale WebSocket cho 5,000+ connections

### Vấn đề

```
1 server giữ 5,000 WebSocket connections đồng thời:
  - Memory: ~5,000 × 50KB = ~250MB (chấp nhận được)
  - File descriptors: cần ulimit -n > 10000
  - Goroutines (Go): 5,000 × 2 = 10,000 goroutines (nhẹ)
  - Threads (Java): cần dùng NIO hoặc WebFlux (không dùng thread-per-connection)
```

### Giải pháp: Nhiều server + Redis Pub/Sub

```
Client A ──▶ Server 1 ──┐
Client B ──▶ Server 1    ├──▶ Redis Pub/Sub ──▶ Tất cả servers nhận
Client C ──▶ Server 2 ──┘

Admin muốn broadcast "Còn 5 phút":
  1. Admin API → Server 1
  2. Server 1 publish "exam:ielts_mock_0056" → Redis
  3. Redis fan-out → Server 1 + Server 2
  4. Mỗi server push tới clients local của mình
```

### Sticky sessions (đơn giản hơn)

```
Dùng Load Balancer sticky session (IP hash hoặc cookie):
  - Client luôn kết nối cùng 1 server
  - Server chỉ quản lý clients local
  - Broadcast qua Redis Pub/Sub

Nginx config:
  upstream exam_ws {
      ip_hash;  # sticky session
      server ws-server-1:8080;
      server ws-server-2:8080;
  }
```

---

## 8. Interview Quick Reference

### "Tại sao dùng WebSocket cho hệ thống thi?"

> WebSocket cung cấp **bidirectional communication**. REST polling có 2 hạn chế: (1) server không thể push message chủ động (hết giờ, cảnh báo gian lận), (2) polling tạo request thừa khi không có data mới. WebSocket giải quyết cả 2 với 1 persistent connection, đồng thời giúp server **biết real-time** client còn kết nối hay không (heartbeat).

### "WebSocket có thay thế REST không?"

> Không. Dùng **kết hợp**: REST cho download đề (large payload), authentication, final submit (cần HTTP status rõ ràng). WebSocket cho sync đáp án (frequent, small), server push (timer, warnings), heartbeat. Khi WebSocket fail → **fallback** về REST polling tự động.

### "Xử lý offline thế nào khi dùng WebSocket?"

> WebSocket chỉ hoạt động khi online. Khi offline: (1) đáp án queue trong IndexedDB, (2) UI vẫn responsive (offline-first). Khi reconnect: (3) WebSocket reconnect với exponential backoff, (4) flush queue, (5) server gửi lại timer correction. **Offline-first architecture không thay đổi** — WebSocket chỉ là upgrade cho online experience.

### "Scale 5,000 connections thế nào?"

> 1 server Go chịu được 5,000-10,000 connections (goroutines nhẹ). Nếu cần nhiều server: dùng **Redis Pub/Sub** để cross-server broadcast + **sticky sessions** ở load balancer. Java dùng Spring WebSocket với NIO (non-blocking) hoặc WebFlux.

### So sánh Java vs Go cho WebSocket

| | Java (Spring) | Go (Gorilla) |
|--|:---:|:---:|
| Setup | Nhanh (annotation-based) | Code nhiều hơn |
| Performance | Tốt (NIO) | Rất tốt (goroutines nhẹ) |
| Memory/connection | ~100-200KB | ~10-50KB |
| Max connections/server | ~5,000-10,000 | ~10,000-50,000 |
| Ecosystem | Spring Security, STOMP | Tự build |
| Khi nào chọn | Team Java, cần ecosystem | High concurrency, nhẹ |
