# WebSocket trong Anti-Cheat — Event Streaming & Real-time Alert Push

> Bổ sung kiến trúc WebSocket cho Anti-Cheat Service. WebSocket giải quyết 2 bài toán: (1) Event streaming từ client → server hiệu quả hơn REST batch, (2) Push cảnh báo gian lận xuống client & admin dashboard real-time.

---

## Mục lục

1. [Tại sao Anti-Cheat cần WebSocket?](#1-tại-sao-anti-cheat-cần-websocket)
2. [Kiến trúc: REST Batch vs WebSocket Streaming](#2-kiến-trúc-rest-batch-vs-websocket-streaming)
3. [Event Streaming Protocol](#3-event-streaming-protocol)
4. [Admin Dashboard — Real-time Monitoring](#4-admin-dashboard--real-time-monitoring)
5. [Cài đặt — Java (Spring Boot)](#5-cài-đặt--java-spring-boot)
6. [Cài đặt — Golang (Gorilla)](#6-cài-đặt--golang-gorilla)
7. [Scale cho 10,000+ concurrent users](#7-scale-cho-10000-concurrent-users)
8. [Interview Quick Reference](#8-interview-quick-reference)

---

## 1. Tại sao Anti-Cheat cần WebSocket?

### Hiện tại: REST batch mỗi vài giây

```
Client ─── POST /events (batch 5-10 events) ───▶ Anti-Cheat API
Client ◀── 200 OK ─────────────────────────────── Anti-Cheat API

Vấn đề:
  1. DELAY: Event xảy ra → đợi batch interval → mới gửi
     → Tab switch lúc 14:32:01, gửi lên server lúc 14:32:05 (4s delay)
     → Real-time detection bị trễ

  2. KHÔNG PUSH ĐƯỢC CẢNH BÁO:
     → Server detect gian lận → không gửi warning xuống client được
     → Client phải poll "có cảnh báo nào không?" → lãng phí

  3. ADMIN DASHBOARD:
     → Admin muốn xem live: thí sinh nào đang nghi ngờ?
     → Polling dashboard mỗi 5s = 200 admin × 12 req/phút = 2,400 req/phút
     → Vẫn có delay 5s

  4. KHÔNG BIẾT CLIENT CÒN SỐNG:
     → Client tắt browser → server không biết ngay
     → Phải đợi timeout (30-60s) mới phát hiện
```

### WebSocket giải quyết

```
Với WebSocket:

  1. EVENT STREAMING (Client → Server):
     → Tab switch → gửi ngay qua WS (< 100ms)
     → Không cần batch → real-time detection thật sự < 1ms sau khi nhận

  2. ALERT PUSH (Server → Client):
     → Detect gian lận → push WARNING ngay xuống client
     → "Hệ thống phát hiện bạn chuyển tab. Đây là cảnh báo lần 2."
     → Push VOID nếu critical: "Bài thi đã bị hủy do phát hiện bot."

  3. ADMIN LIVE DASHBOARD (Server → Admin):
     → Mỗi detection mới → push ngay lên admin dashboard
     → Admin thấy real-time: "User X vừa tab switch lần 5"
     → Admin click "Void exam" → push VOID xuống client ngay

  4. CONNECTION MONITORING:
     → Client disconnect → server biết ngay (onClose)
     → Log event "connection_lost" → useful cho analysis
     → Client reconnect → server biết "quay lại sau 5 phút offline"
```

---

## 2. Kiến trúc: REST Batch vs WebSocket Streaming

### Kiến trúc đề xuất: Hybrid

```
┌──────────────────────────────────────────────────────────────────────┐
│                     EXAM CLIENT (Browser JS SDK)                      │
│                                                                       │
│  ┌───────────────────────┐     ┌────────────────────────────┐        │
│  │  Anti-Cheat SDK        │     │  Alert Receiver             │       │
│  │                        │     │                             │       │
│  │  Collect events:       │     │  Nhận từ server:            │       │
│  │  - tab_switch          │     │  - WARNING (hiện popup)     │       │
│  │  - paste_detected      │     │  - FORCE_SUBMIT (nộp bài)   │       │
│  │  - devtools_open       │     │  - SESSION_VOID (hủy bài)   │       │
│  │  - typing_pattern      │     │                             │       │
│  └───────────┬────────────┘     └──────────────┬──────────────┘       │
│              │                                  │                     │
│              │     ┌─────────────────────┐      │                     │
│              └────▶│  WebSocket Client   │◀─────┘                     │
│                    │  (bidirectional)     │                            │
│                    └──────────┬───────────┘                            │
└───────────────────────────────┼────────────────────────────────────────┘
                                │
                    ════════════╪════════════  Network
                                │
              ┌─────────────────▼─────────────────┐
              │       WebSocket Gateway            │
              │  (Golang / Java)                   │
              │                                    │
              │  ┌─────────────────────────┐       │
              │  │ Connection Manager       │       │
              │  │ - 10,000+ connections    │       │
              │  │ - sessionID → conn map   │       │
              │  │ - examID → sessions map  │       │
              │  └────────────┬─────────────┘       │
              └───────────────┼─────────────────────┘
                              │
                ┌─────────────┼──────────────┐
                │             │              │
          ┌─────▼──────┐ ┌───▼────────┐ ┌───▼──────────┐
          │ Redis       │ │ Rule Engine│ │ Admin WS     │
          │ Buffer +    │ │ (detect)   │ │ Dashboard    │
          │ Real-time   │ │            │ │ (push alerts)│
          │ Counter     │ └────────────┘ └──────────────┘
          └─────┬───────┘
                │ Flush 60s
          ┌─────▼───────┐
          │ TimescaleDB  │
          └──────────────┘
```

### So sánh: REST vs WebSocket cho Anti-Cheat

| | REST Batch | WebSocket |
|--|:---:|:---:|
| Event delivery | Batch mỗi 3-5s | Real-time (< 100ms) |
| Server → Client push | Không | **Có** (warnings, void) |
| Connection awareness | Không | **Có** (biết online/offline) |
| Admin live feed | Polling | **Real-time push** |
| Overhead | HTTP headers mỗi request | 1 lần handshake |
| Offline handling | Retry khi online | Reconnect + flush queue |
| Complexity | Đơn giản | Phức tạp hơn (state management) |

### Khi nào vẫn giữ REST?

```
REST vẫn dùng cho:
  - Batch flush event từ buffer → TimescaleDB (server-internal, không cần WS)
  - Admin API: GET /detections, GET /exams/:id/events (query data)
  - Fallback khi WebSocket không kết nối được
  - Initial event dump khi reconnect (lượng lớn events queued)
```

---

## 3. Event Streaming Protocol

### Client → Server: Event Messages

```json
{
  "type": "EVENT",
  "payload": {
    "id": "evt-uuid",
    "event_type": "navigation",
    "action": "tab_switch",
    "timestamp": "2026-04-06T14:32:01.234Z",
    "question_id": "reading_q12",
    "section_type": "reading",
    "metadata": {
      "hidden_duration_ms": 8500,
      "switch_count_in_session": 3
    }
  }
}
```

### Server → Client: Alert Messages

```json
// Cảnh báo
{
  "type": "WARNING",
  "payload": {
    "severity": "medium",
    "message": "Hệ thống phát hiện bạn chuyển tab lần thứ 3. Vui lòng tập trung làm bài.",
    "detection_id": "det-uuid",
    "warning_count": 3
  }
}

// Hủy bài thi
{
  "type": "SESSION_VOID",
  "payload": {
    "reason": "automation_detected",
    "message": "Bài thi đã bị hủy do phát hiện phần mềm tự động."
  }
}
```

### Server → Admin Dashboard: Live Feed

```json
{
  "type": "DETECTION",
  "payload": {
    "user_id": "user_1234",
    "user_name": "Nguyễn Văn A",
    "exam_id": "ielts_mock_0056",
    "rule": "tab_switch_threshold",
    "severity": "medium",
    "details": "Tab switch 5 lần trong 10 phút",
    "timestamp": "2026-04-06T14:32:01Z",
    "action_taken": "warning_sent"
  }
}
```

---

## 4. Admin Dashboard — Real-time Monitoring

```
Admin mở dashboard → kết nối WebSocket riêng:
  wss://admin-api.example.com/ws/admin?token=xxx&exam_id=ielts_mock_0056

Server push mỗi khi có detection mới:
  → Admin thấy ngay: "User X - Tab switch lần 5 - Medium severity"
  → Admin click "Void exam" → Server push SESSION_VOID xuống client
  → Admin thấy: "User X - Exam voided by admin"
```

### Admin WebSocket khác Client WebSocket

| | Client WS | Admin WS |
|--|:---:|:---:|
| Gửi | Events (tab_switch, paste...) | Commands (void, warn) |
| Nhận | Warnings, timer | Detections feed, status |
| Auth | Exam token (thí sinh) | Admin JWT (role=admin) |
| Số lượng | 10,000 connections | 10-50 connections |
| Endpoint | `/ws/exam` | `/ws/admin` |

---

## 5. Cài đặt — Java (Spring Boot)

### 5.1. Anti-Cheat WebSocket Handler

```java
@Component
public class AntiCheatWebSocketHandler extends TextWebSocketHandler {

    private final ConcurrentHashMap<String, WebSocketSession> examClients = new ConcurrentHashMap<>();
    private final ConcurrentHashMap<String, WebSocketSession> adminClients = new ConcurrentHashMap<>();

    @Autowired
    private EventIngestionService eventService;

    @Autowired
    private RealTimeDetector realTimeDetector;

    @Autowired
    private ObjectMapper objectMapper;

    // === Client gửi event lên ===
    @Override
    protected void handleTextMessage(WebSocketSession session, TextMessage message) {
        try {
            JsonNode msg = objectMapper.readTree(message.getPayload());
            String type = msg.get("type").asText();

            if ("EVENT".equals(type)) {
                handleExamEvent(session, msg.get("payload"));
            }
        } catch (Exception e) {
            log.error("Error processing WS event", e);
        }
    }

    private void handleExamEvent(WebSocketSession session, JsonNode payload) {
        String userId = getUserId(session);
        String examId = getExamId(session);
        String sessionId = getSessionId(session);

        // 1. Parse event
        ExamEvent event = objectMapper.convertValue(payload, ExamEvent.class);
        event.setUserId(userId);
        event.setExamId(examId);
        event.setSessionId(sessionId);

        // 2. Push vào Redis buffer (non-blocking)
        eventService.ingest(event);

        // 3. Real-time detection (Redis counter + threshold)
        Detection detection = realTimeDetector.check(event);

        if (detection != null) {
            // 4a. Push cảnh báo xuống CLIENT
            pushWarningToClient(sessionId, detection);

            // 4b. Push detection lên ADMIN dashboard
            pushDetectionToAdmins(examId, detection);
        }
    }

    // === Push cảnh báo xuống thí sinh ===
    public void pushWarningToClient(String sessionId, Detection detection) {
        WebSocketSession session = examClients.get(sessionId);
        if (session == null || !session.isOpen()) return;

        Map<String, Object> msg;
        if (detection.getSeverity() == Severity.CRITICAL) {
            msg = Map.of("type", "SESSION_VOID", "payload", Map.of(
                    "reason", detection.getRule(),
                    "message", "Bài thi đã bị hủy: " + detection.getDescription()
            ));
        } else {
            msg = Map.of("type", "WARNING", "payload", Map.of(
                    "severity", detection.getSeverity().name(),
                    "message", detection.getDescription(),
                    "warning_count", detection.getWarningCount()
            ));
        }
        sendMessage(session, msg);
    }

    // === Push detection lên admin dashboard ===
    public void pushDetectionToAdmins(String examId, Detection detection) {
        Map<String, Object> msg = Map.of("type", "DETECTION", "payload", Map.of(
                "user_id", detection.getUserId(),
                "exam_id", examId,
                "rule", detection.getRule(),
                "severity", detection.getSeverity().name(),
                "details", detection.getDescription(),
                "timestamp", detection.getTimestamp()
        ));

        adminClients.values().stream()
                .filter(WebSocketSession::isOpen)
                .forEach(adminSession -> sendMessage(adminSession, msg));
    }

    // === Admin gửi command (void exam, send extra warning) ===
    public void handleAdminCommand(WebSocketSession adminSession, JsonNode payload) {
        String command = payload.get("command").asText();
        String targetSessionId = payload.get("session_id").asText();

        switch (command) {
            case "VOID_EXAM" -> {
                pushWarningToClient(targetSessionId, Detection.voidByAdmin(
                        payload.get("reason").asText()
                ));
            }
            case "SEND_WARNING" -> {
                pushWarningToClient(targetSessionId, Detection.warningByAdmin(
                        payload.get("message").asText()
                ));
            }
        }
    }

    private void sendMessage(WebSocketSession session, Object payload) {
        try {
            session.sendMessage(new TextMessage(objectMapper.writeValueAsString(payload)));
        } catch (IOException e) {
            log.error("Failed to send WS message", e);
        }
    }
}
```

### 5.2. STOMP Alternative (Spring-native, đơn giản hơn)

```java
// Nếu muốn dùng STOMP (Spring hỗ trợ built-in):
@Configuration
@EnableWebSocketMessageBroker
public class StompConfig implements WebSocketMessageBrokerConfigurer {

    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic");    // Server → Client
        config.setApplicationDestinationPrefixes("/app"); // Client → Server
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws/anticheat")
                .setAllowedOrigins("https://exam.example.com")
                .withSockJS(); // Fallback cho browser cũ
    }
}

// Controller xử lý event
@Controller
public class AntiCheatStompController {

    @Autowired
    private SimpMessagingTemplate messagingTemplate;

    // Client gửi event → /app/event
    @MessageMapping("/event")
    public void handleEvent(@Payload ExamEvent event, Principal principal) {
        event.setUserId(principal.getName());
        eventService.ingest(event);

        Detection detection = realTimeDetector.check(event);
        if (detection != null) {
            // Push cảnh báo → /topic/warnings/{sessionId}
            messagingTemplate.convertAndSendToUser(
                    event.getSessionId(),
                    "/topic/warnings",
                    detection
            );

            // Push lên admin → /topic/admin/detections
            messagingTemplate.convertAndSend(
                    "/topic/admin/detections",
                    detection
            );
        }
    }
}
```

---

## 6. Cài đặt — Golang (Gorilla)

### 6.1. Anti-Cheat Hub (quản lý cả exam client + admin)

```go
package ws

import "sync"

type AntiCheatHub struct {
    mu          sync.RWMutex
    examClients map[string]*Client  // sessionID → client (thí sinh)
    adminConns  map[string]*Client  // adminID → client (admin)
    examIndex   map[string][]string // examID → []sessionID
}

func NewAntiCheatHub() *AntiCheatHub {
    return &AntiCheatHub{
        examClients: make(map[string]*Client),
        adminConns:  make(map[string]*Client),
        examIndex:   make(map[string][]string),
    }
}

// Push cảnh báo xuống thí sinh
func (h *AntiCheatHub) PushWarning(sessionID string, detection *Detection) {
    h.mu.RLock()
    client, ok := h.examClients[sessionID]
    h.mu.RUnlock()

    if !ok || client == nil {
        return
    }

    var msg *Message
    if detection.Severity == "critical" {
        msg = &Message{
            Type: "SESSION_VOID",
            Payload: mustMarshal(map[string]interface{}{
                "reason":  detection.Rule,
                "message": "Bài thi đã bị hủy: " + detection.Description,
            }),
        }
    } else {
        msg = &Message{
            Type: "WARNING",
            Payload: mustMarshal(map[string]interface{}{
                "severity":      detection.Severity,
                "message":       detection.Description,
                "warning_count": detection.WarningCount,
            }),
        }
    }

    client.Send(msg)
}

// Push detection lên tất cả admin dashboard
func (h *AntiCheatHub) PushToAdmins(detection *Detection) {
    msg := &Message{
        Type: "DETECTION",
        Payload: mustMarshal(map[string]interface{}{
            "user_id":   detection.UserID,
            "exam_id":   detection.ExamID,
            "rule":      detection.Rule,
            "severity":  detection.Severity,
            "details":   detection.Description,
            "timestamp": detection.Timestamp,
        }),
    }

    h.mu.RLock()
    defer h.mu.RUnlock()

    for _, admin := range h.adminConns {
        admin.Send(msg)
    }
}
```

### 6.2. Event Handler — Nhận event, detect, push alert

```go
package ws

import (
    "encoding/json"
    "time"
)

type AntiCheatHandler struct {
    hub             *AntiCheatHub
    eventService    EventIngestionService
    realtimeDetector RealTimeDetector
}

func (h *AntiCheatHandler) HandleMessage(client *Client, msg *Message) {
    switch msg.Type {
    case "EVENT":
        h.handleExamEvent(client, msg)
    case "ADMIN_COMMAND":
        h.handleAdminCommand(client, msg)
    }
}

func (h *AntiCheatHandler) handleExamEvent(client *Client, msg *Message) {
    var event ExamEvent
    json.Unmarshal(msg.Payload, &event)
    event.UserID = client.UserID
    event.ExamID = client.ExamID
    event.SessionID = client.SessionID

    // 1. Push vào Redis buffer (goroutine riêng, non-blocking)
    go h.eventService.Ingest(&event)

    // 2. Real-time detection
    detection := h.realtimeDetector.Check(&event)
    if detection == nil {
        return
    }

    // 3. Push cảnh báo xuống thí sinh
    h.hub.PushWarning(client.SessionID, detection)

    // 4. Push lên admin dashboard
    h.hub.PushToAdmins(detection)
}

func (h *AntiCheatHandler) handleAdminCommand(client *Client, msg *Message) {
    var cmd struct {
        Command   string `json:"command"`
        SessionID string `json:"session_id"`
        Reason    string `json:"reason"`
        Message   string `json:"message"`
    }
    json.Unmarshal(msg.Payload, &cmd)

    switch cmd.Command {
    case "VOID_EXAM":
        h.hub.PushWarning(cmd.SessionID, &Detection{
            Severity:    "critical",
            Rule:        "admin_void",
            Description: cmd.Reason,
        })
    case "SEND_WARNING":
        h.hub.PushWarning(cmd.SessionID, &Detection{
            Severity:    "medium",
            Rule:        "admin_warning",
            Description: cmd.Message,
        })
    }
}
```

### 6.3. HTTP Endpoints — 2 endpoint WS riêng

```go
func main() {
    hub := NewAntiCheatHub()
    handler := &AntiCheatHandler{
        hub:              hub,
        eventService:     eventService,
        realtimeDetector: detector,
    }

    // Endpoint cho thí sinh (exam client JS SDK)
    http.HandleFunc("/ws/anticheat", ServeExamWs(hub, handler, jwtValidator))

    // Endpoint riêng cho admin dashboard
    http.HandleFunc("/ws/admin", ServeAdminWs(hub, handler, adminJwtValidator))

    // REST API vẫn giữ
    http.HandleFunc("/api/events", restEventHandler)       // Fallback
    http.HandleFunc("/api/detections", restDetectionHandler) // Admin query

    log.Fatal(http.ListenAndServe(":8081", nil))
}
```

---

## 7. Scale cho 10,000+ concurrent users

### Bottleneck Analysis

```
10,000 thí sinh × 1 WS connection = 10,000 connections
Mỗi thí sinh gửi ~2 event/giây = 20,000 events/giây
+ 50 admin connections

Goroutines: 10,000 × 2 (read + write) + 50 × 2 = ~20,100
Memory:     10,000 × 50KB = ~500MB
CPU:        Redis INCR per event = 20,000 ops/giây (Redis chịu dễ)
```

### Multi-server với Redis Pub/Sub

```go
// Khi detect → publish lên Redis channel
// Tất cả server subscribe → push xuống local clients

func (h *AntiCheatHub) PublishDetection(detection *Detection) {
    // Publish lên Redis
    data, _ := json.Marshal(detection)
    redisClient.Publish(ctx, "anticheat:detections", data)
}

// Mỗi server subscribe
func (h *AntiCheatHub) SubscribeDetections() {
    sub := redisClient.Subscribe(ctx, "anticheat:detections")
    for msg := range sub.Channel() {
        var detection Detection
        json.Unmarshal([]byte(msg.Payload), &detection)

        // Push warning tới client nếu nó ở server này
        h.PushWarning(detection.SessionID, &detection)

        // Push lên admin
        h.PushToAdmins(&detection)
    }
}
```

---

## 8. Interview Quick Reference

### "Tại sao Anti-Cheat cần WebSocket?"

> Hai lý do chính: (1) **Event streaming real-time** — events gửi ngay khi xảy ra thay vì batch 3-5s, giúp real-time detection nhanh hơn. (2) **Server push alerts** — khi detect gian lận, server push cảnh báo xuống client ngay lập tức thay vì client phải poll. Đặc biệt quan trọng cho admin dashboard: admin thấy live feed detections, có thể void exam ngay.

### "WebSocket có ảnh hưởng tới Offline-First không?"

> Không. Anti-cheat events khi offline được **queue trong SDK**, khi reconnect sẽ flush lên qua WebSocket hoặc REST fallback. Tuy nhiên, offline events có **giá trị detection thấp hơn** vì timestamp không xác minh được — đây là trade-off chấp nhận được vì thí sinh offline cũng khó gian lận (không có internet để tra cứu).

### "Connection lost có phải là gian lận không?"

> WebSocket disconnect **không tự động là gian lận**. Có thể do mạng yếu. Nhưng **pattern** đáng ngờ: disconnect ngay sau tab_switch, disconnect rồi reconnect với IP khác, disconnect nhiều lần ngắn. Server log tất cả connect/disconnect events → batch analysis phân tích pattern.

### So sánh implementation

| | Java (Spring) | Go (Gorilla) |
|--|:---:|:---:|
| 10,000 connections | Cần NIO/WebFlux | Native goroutines |
| Memory | ~1-2GB | ~500MB |
| STOMP support | Built-in | Không |
| Redis integration | Spring Data Redis | go-redis |
| Phù hợp | Team Java, STOMP cần | High throughput |
