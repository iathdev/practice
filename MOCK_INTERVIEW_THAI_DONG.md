# Mock Interview — Deep Dive vào Dự Án Thực Tế | Thái Đông

> **Tổng thời gian:** ~90 phút
> **Format:** EdTech · Enterprise · Digital Transformation
> **Vai trò phỏng vấn:** Senior Backend Engineer

---

## 📅 Career Timeline

| Thời gian | Công ty | Vai trò |
|---|---|---|
| 09/2025 – Hiện tại | **PREP Technology JSC** | Backend Developer (PHP/Go) |
| 10/2024 – 09/2025 | **GHTK** | Web Engineer (Java/PHP) |
| 06/2020 – 10/2024 | **Tomosia Việt Nam** | Software Engineer (Java/PHP) |

**PREP:** Nền tảng thi trực tuyến B2B · Anti-cheating real-time (Go + Redis) · Event-driven: Laravel + Go · Learning Profile với Kafka

**GHTK:** Enterprise platforms GAM & HRM · DDD + Hexagonal Architecture · IAM với OAuth2 · Deadlock production, tối ưu query

**Tomosia:** Số hoá toàn diện chuỗi công viên giải trí (bán vé online, QR check-in, quản lý capacity, vận hành sự kiện) · AWS Data Pipeline (Lambda + SQS + OpenSearch + Puppeteer)

---

## 👋 Phần 1 — Giới thiệu & Context Setting (10 phút)

**Interviewer:** Chào Đông, bạn có kinh nghiệm khá đa dạng — EdTech (PREP), Enterprise (GHTK), Digital Transformation (Tomosia). Bạn có thể share tổng quan và điều gì khiến bạn phát triển nhiều nhất qua từng giai đoạn không?

**Thái Đông:**

### 🎢 Tomosia (4 năm — 2020–2024)

- **Dự án flagship — Ticket Booking Platform:** Số hoá toàn bộ chuỗi công viên giải trí — inventory management, payment integration (webhook + polling), QR check-in, quản lý vận hành. Bài toán kết hợp nhiều thách thức kỹ thuật trong cùng một hệ thống.
- **Inventory & Concurrency:** Inventory model tách 3 trạng thái `total_capacity / reserved / booked`. Redis Lua script (atomic pre-check) + DB optimistic lock để chống oversell — reject sớm tại Redis, chỉ request hợp lệ mới đụng đến database.
- **Payment Reliability:** Dual Confirmation (webhook + gateway polling job) giải quyết "tiền trừ nhưng vé không có". Transactional Outbox pattern đảm bảo event không bao giờ bị mất.
- **AWS Data Pipeline:** Lambda + SQS + OpenSearch + Puppeteer để thu thập và index dữ liệu; monitoring với CloudWatch.

### 📦 GHTK (1 năm — 2024–2025)

- **Domain:** Enterprise platforms — GAM, HRM nội bộ
- **Thách thức cốt lõi:** Maintainability và scalability của hệ thống high-concurrency
- **Điểm nổi bật:** DDD + Hexagonal Architecture để cải thiện codebase, xử lý production deadlocks

### 🎓 PREP (hiện tại — 2025)

- **Domain:** EdTech — nền tảng thi tiếng Anh B2B
- **Thách thức cốt lõi:** Real-time processing và system reliability khi hàng nghìn thí sinh thi cùng lúc
- **Điểm nổi bật:** Anti-cheating service bằng Go, event-driven architecture tách biệt exam domain và management domain

> **Điểm chung:** Mỗi giai đoạn đều phải đối mặt với câu hỏi _"làm sao để hệ thống chạy ổn định dưới áp lực"_ — dù là áp lực traffic cao, business logic phức tạp, hay yêu cầu real-time.

---

## 🎓 Phần 2 — PREP: Exam Platform & Anti-Cheating Deep Dive (25 phút)

**Stack:** Go (Exam Domain) · Laravel (Management) · Redis · Kafka · PostgreSQL · Docker

**Kiến trúc tổng quan:**

```
Trình duyệt (Thí sinh)    Admin Portal      Partner B2B
        │                      │                 │
        └──────────────────────┴─────────────────┘
                               │
                     API Gateway / Load Balancer
                               │
          ┌────────────────────┼────────────────────┐
          │                    │                    │
  ┌───────┴──────┐   ┌─────────┴───────┐   ┌───────┴──────────┐
  │ Exam Service │   │ Management Svc  │   │ Learning Profile │
  │    (Go)      │   │   (Laravel)     │   │   Service (Go)   │
  │ - Bắt đầu thi│   │ - Khóa học      │   │ - Tổng hợp HĐ   │
  │ - Nộp bài    │   │ - Người dùng    │   │ - Tiến độ        │
  │ - Bộ đếm giờ │   │ - B2B config    │   │ - Cá nhân hóa   │
  └───────┬──────┘   └─────────┬───────┘   └───────┬──────────┘
          │                    │                    │
          └────────────────────┴────────────────────┘
                               │
          ┌────────────────────┴────────────────────┐
          │             Event Bus (Kafka)            │
          │  exam.started · exam.submitted           │
          │  cheat.detected · score.calculated       │
          │  learning.activity.completed             │
          └────────┬────────────────┬───────────────┘
                   │                │
         ┌─────────┴──────┐  ┌──────┴──────────┐
         │ Anti-Cheat Svc │  │ Scoring Pipeline │
         │     (Go)       │  │    (Async)       │
         │ - Tab switch   │  │ - Chấm tự động   │
         │ - Focus loss   │  │ - Batch scoring  │
         │ - Redis buffer │  │ - Ghi log        │
         └────────────────┘  └──────────────────┘
```

---

### Q1: Anti-Cheating Service

**Interviewer:** Walk through flow khi thí sinh tab switch hay focus loss. Tại sao chọn Go? Tại sao Redis làm event buffer thay vì push thẳng lên Kafka?

**Thái Đông:**

**Detection Flow:**

```
Trình duyệt        Go Anti-Cheat Svc    Redis (Buffer)    Kafka / Alert
     │                     │                  │                │
     │ 1. EventStream       │                  │                │
     │   {type:"tab_switch"}│                  │                │
     ├─────────────────────►│                  │                │
     │                      │ 2. LPUSH         │                │
     │                      │   cheat:E001     │                │
     │                      ├─────────────────►│                │
     │ 3. Thêm events       │                  │                │
     │   (focus_loss x3)    ├─────────────────►│ (buffer tiếp)  │
     │                      │ 4. Flush (1s)    │                │
     │                      │ LRANGE + analyze │                │
     │                      │◄─────────────────┤                │
     │                      │ 5. Đánh giá:     │                │
     │                      │ tab_switch≥3/60s │                │
     │                      │   → WARN         │                │
     │                      │ 6. Vi phạm → Pub │                │
     │                      ├──────────────────┼───────────────►│
     │ 7. Cảnh báo realtime │                  │                │
     │◄─────────────────────┤                  │                │
```

**Tại sao chọn Go?**
- **Goroutines:** 5.000 thí sinh = 5.000 goroutines. Goroutine chỉ ~2KB stack vs ~1MB thread Java
- **Low latency:** Anti-cheat xử lý trong milliseconds. Go compile binary, không có JVM overhead
- **Concurrency model:** Channel + goroutine pattern phù hợp với event processing pipeline

**Tại sao Redis buffer thay vì push thẳng Kafka?**
- **Tần suất cực cao:** 10–50 events/phút/thí sinh × 5.000 = 50K–250K events/phút. Small messages lên Kafka = không hiệu quả
- **Batching:** Redis buffer gom events theo session, flush mỗi 1–2 giây → giảm 95% message volume lên Kafka
- **Tốc độ:** Redis LPUSH ~0.1ms vs Kafka produce ~5–10ms
- **Transient data:** Raw cheat events chỉ cần giữ vài phút — không cần Kafka's durability

```go
type AntiCheatProcessor struct {
    redisClient *redis.Client
    kafkaWriter *kafka.Writer
    rules       []DetectionRule
}

// Buffer event vào Redis
func (p *AntiCheatProcessor) BufferEvent(ctx context.Context, event CandidateEvent) error {
    key := fmt.Sprintf("cheat:events:%s:%s", event.ExamID, event.CandidateID)
    data, _ := json.Marshal(event)

    pipe := p.redisClient.Pipeline()
    pipe.LPush(ctx, key, data)
    pipe.Expire(ctx, key, 5*time.Minute) // TTL — tự dọn dẹp
    _, err := pipe.Exec(ctx)
    return err
}

// Flush & analyze — chạy mỗi 1 giây
func (p *AntiCheatProcessor) FlushAndAnalyze(ctx context.Context, examID, candidateID string) {
    key := fmt.Sprintf("cheat:events:%s:%s", examID, candidateID)

    rawEvents, _ := p.redisClient.LRange(ctx, key, 0, -1).Result()
    p.redisClient.Del(ctx, key)

    var events []CandidateEvent
    for _, raw := range rawEvents {
        var e CandidateEvent
        json.Unmarshal([]byte(raw), &e)
        events = append(events, e)
    }

    for _, rule := range p.rules {
        if violation := rule.Evaluate(events); violation != nil {
            p.publishViolation(ctx, violation) // Chỉ violations mới lên Kafka
        }
    }
}

// Detection rule: Tab switch frequency
func (r *TabSwitchRule) Evaluate(events []CandidateEvent) *Violation {
    count := 0
    cutoff := time.Now().Add(-r.Window)
    for _, e := range events {
        if e.EventType == "tab_switch" && e.Timestamp.After(cutoff) {
            count++
        }
    }
    if count >= r.MaxCount {
        return &Violation{Type: "FREQUENT_TAB_SWITCH", Severity: "WARNING"}
    }
    return nil
}
```

> **Key insight:** Chỉ violations mới được push lên Kafka → giảm 95% message volume. Rules engine có thể thêm/sửa rule không cần redeploy.

---

### Q2: Exam Submission & Không mất bài thi

**Interviewer:** Khi thí sinh bấm "Nộp bài", flow diễn ra thế nào? Làm sao đảm bảo không mất bài thi nếu system gặp sự cố?

**Thái Đông:**

**Tại sao tách 2 domain:**
- **Management (Laravel):** CRUD-heavy, admin ops, report — phù hợp Eloquent ORM, rapid dev
- **Exam (Go):** High-concurrency, real-time, latency-sensitive — cần compiled language
- **Communication:** Event-driven qua Kafka. 2 services KHÔNG gọi trực tiếp nhau

**Submission Flow:**

```
Thí sinh      Go Exam Service    Kafka     Scoring Pipeline   Laravel Mgmt
   │                 │             │               │                │
   │ POST /submit    │             │               │                │
   ├────────────────►│             │               │                │
   │                 │ Validate    │               │                │
   │                 │ Persist     │               │                │
   │                 │ status=PENDING              │                │
   │                 │ Publish ────►"exam.submitted"               │
   │ ACK: "Đã nhận" │             │               │                │
   │◄────────────────┤             │               │                │
   │                 │             │ Consume event │                │
   │                 │             ├──────────────►│                │
   │                 │             │               │ Chấm bài       │
   │                 │             │"score.calculated"              │
   │                 │             │◄──────────────┤                │
   │                 │             ├───────────────────────────────►│
   │                 │             │               │  Update report │
```

**3 Lớp bảo vệ không mất bài thi:**

**Lớp 1 — Persist TRƯỚC khi publish:** Answers lưu vào DB trước (status: PENDING_SCORE). Kafka down → bài vẫn còn trong DB.

**Lớp 2 — Idempotent submission (Upsert):** Thí sinh bấm nhiều lần → `ON CONFLICT (exam_id, candidate_id) DO UPDATE`.

**Lớp 3 — Reconciliation Job:**

```go
func (j *ReconciliationJob) Run(ctx context.Context) {
    stale, _ := j.repo.FindByStatusOlderThan(ctx, "PENDING_SCORE", 10*time.Minute)
    for _, sub := range stale {
        log.Warn("Re-publishing submission bị trễ", "id", sub.ID)
        j.kafkaProducer.Publish(ctx, "exam.submitted", sub.ID, sub.ToEvent())
    }
}
```

> **Kết quả:** 0 bài thi bị mất · <200ms submission latency · <30s chấm bài · 5000+ thí sinh đồng thời

---

### Q3: Learning Profile & Consistency

**Interviewer:** Learning Profile aggregate data từ nhiều services. Làm sao đảm bảo data consistency? Nếu consumer lag, profile có bị sai không?

**Thái Đông:**

Learning Profile là **read-optimized aggregate** được xây từ nhiều event sources:

```go
// Events được consume:
// "exam.submitted"       → Cập nhật số bài thi đã làm
// "score.calculated"     → Cập nhật điểm trung bình, skill breakdown
// "lesson.completed"     → Cập nhật tiến độ học tập

// Partition key = user_id → events cùng 1 user xử lý đúng thứ tự

func (s *ProfileService) OnScoreCalculated(ctx context.Context, event ScoreCalculatedEvent) error {
    profile, _ := s.repo.GetOrCreate(ctx, event.CandidateID)

    // Idempotency: bài thi này đã được tính chưa?
    if s.repo.IsExamAlreadyCounted(ctx, event.CandidateID, event.ExamID) {
        return nil
    }

    profile.TotalExams++
    profile.AverageScore = recalculateAverage(profile.AverageScore, profile.TotalExams-1, event.Score)
    profile.LastActivityAt = event.Timestamp
    return s.repo.Save(ctx, profile)
}
```

**Chiến lược consistency:**
- **Idempotency:** Mỗi event có unique ID. Kiểm tra trước khi apply
- **Ordering:** Kafka partition by `user_id` → events cùng 1 user theo đúng thứ tự
- **Rebuild:** Có thể replay từ Kafka để rebuild toàn bộ profile

> **Trade-off chấp nhận:** Profile delay 1–5 giây (OK cho learning analytics). Consumer lag → profile tạm cũ nhưng tự catch up.

---

## 📦 Phần 3 — GHTK: Enterprise Platform & DDD Deep Dive (25 phút)

**Stack:** Spring Boot · Laravel · Kafka · Redis · MySQL · OAuth2/IAM

---

### Q4: DDD + Hexagonal Architecture

**Interviewer:** Codebase trước đó ở trạng thái như thế nào? Bạn tổ chức Hexagonal Architecture ra sao trong Spring Boot?

**Thái Đông:**

**Vấn đề của codebase cũ:**
- **Fat controllers:** Business logic trong controllers, 500–800 dòng/method
- **Tight coupling:** Service A gọi trực tiếp repository của Service B
- **Không có domain boundary:** Logic "Lịch làm việc" rải rác ở 5–6 files
- **Testing khó:** Test business logic phải setup cả DB, HTTP context — mất vài phút

**Module restructure: Working Schedule domain (HRM)**

```
┌─────────────────────────────────────────────┐
│            Driving Adapters (Input)          │
│  ┌─────────────────┐  ┌──────────────────┐  │
│  │ REST Controller  │  │  Kafka Consumer  │  │
│  └────────┬─────────┘  └────────┬─────────┘  │
└───────────┼────────────────────┼─────────────┘
            │                    │
            ▼                    ▼
┌─────────────────────────────────────────────┐
│           Application Layer (Use Cases)      │
│   ScheduleApplicationService                 │
│   - createSchedule · assignShift             │
│   - requestLeave · approveOvertime           │
└─────────────────────────────────────────────┘
            │                    │
            ▼                    ▼
┌─────────────────────────────────────────────┐
│             Domain Layer (Core)              │
│   Entities: Shift (VO) · LeaveRequest        │
│             WorkingSchedule (AR)             │
│   Ports (Interfaces):                        │
│   ScheduleRepository · NotificationPort      │
└─────────────────────────────────────────────┘
            │                    │
            ▼                    ▼
┌─────────────────────────────────────────────┐
│           Driven Adapters (Output)           │
│  MySQL Adapter · Kafka Producer              │
│  Redis Adapter · IAM HTTP Client             │
└─────────────────────────────────────────────┘
```

```java
// === DOMAIN LAYER — không phụ thuộc bất kỳ framework nào ===

// Value Object — Shift
public final class Shift {
    private final LocalTime startTime;
    private final LocalTime endTime;
    private final ShiftType type;

    public Shift(LocalTime startTime, LocalTime endTime, ShiftType type) {
        if (endTime.isBefore(startTime) && type != ShiftType.NIGHT) {
            throw new InvalidShiftException("Giờ kết thúc phải sau giờ bắt đầu");
        }
        // ...
    }
}

// Aggregate Root
public class WorkingSchedule {
    public void assignShift(LocalDate date, Shift shift) {
        if (status == ScheduleStatus.LOCKED) {
            throw new ScheduleLockedException("Không thể sửa lịch đã khoá");
        }
        validateRestGap(date, shift); // Business rule: nghỉ tối thiểu 11h giữa 2 ca
        assignments.add(new DailyAssignment(date, shift));
    }
}

// Port — Domain định nghĩa interface, Adapter implement
public interface ScheduleRepository {
    Optional<WorkingSchedule> findByEmployeeAndPeriod(Long employeeId, YearMonth period);
    void save(WorkingSchedule schedule);
}

// === APPLICATION LAYER ===
@Service @Transactional
public class ScheduleApplicationService {
    private final ScheduleRepository scheduleRepo; // inject Port, không phải impl
    private final IAMPort iamPort;

    public void assignShift(AssignShiftCommand cmd) {
        if (!iamPort.hasPermission(cmd.getRequesterId(), "SCHEDULE_EDIT")) {
            throw new UnauthorizedException();
        }
        WorkingSchedule schedule = scheduleRepo
            .findByEmployeeAndPeriod(cmd.getEmployeeId(), cmd.getPeriod())
            .orElseGet(() -> WorkingSchedule.create(cmd.getEmployeeId(), cmd.getPeriod()));

        Shift shift = new Shift(cmd.getStartTime(), cmd.getEndTime(), cmd.getShiftType());
        schedule.assignShift(cmd.getDate(), shift);
        scheduleRepo.save(schedule);
    }
}

// === DRIVEN ADAPTER ===
@Repository
public class MySQLScheduleRepository implements ScheduleRepository {
    @Override
    public Optional<WorkingSchedule> findByEmployeeAndPeriod(Long employeeId, YearMonth period) {
        return jpaRepo.findByEmployeeIdAndPeriod(employeeId, period)
            .map(this::toDomain); // JPA entity → Domain object
    }
}
```

**Kết quả sau restructure:**
- **Testability:** Domain logic test thuần túy, không cần DB, chạy <100ms
- **Maintainability:** Thay DB (MySQL → PostgreSQL) chỉ cần thay adapter
- **Giảm coupling:** Module khác chỉ gọi Application Service, không access trực tiếp DB
- **Onboarding nhanh:** Dev mới đọc domain layer là hiểu business rules

---

### Q5: Production Deadlock Resolution

**Interviewer:** "Resolved production deadlocks" — kể chi tiết một case: nguyên nhân, detect, fix?

**Thái Đông:**

**Triệu chứng:**
- Giờ cao điểm sáng 8–9h: HRM chậm dần
- Request timeout 30s, log thấy `"Lock wait timeout exceeded"`
- DB connection pool cạn kiệt, các module khác bị ảnh hưởng

**Root Cause — 2 transactions lock ngược thứ tự:**

```sql
-- Transaction A: Manager approve đơn nghỉ phép
BEGIN;
UPDATE leave_requests SET status='APPROVED' WHERE id=101;      -- Lock leave row
-- ...
UPDATE employee_schedules SET day_off=true
    WHERE employee_id=55 AND date='2025-03-15';                -- Chờ Transaction B... DEADLOCK!

-- Transaction B: Job tự động phân ca (chạy song song)
BEGIN;
UPDATE employee_schedules SET shift_id=3
    WHERE employee_id=55 AND date='2025-03-15';                -- Lock schedule row
-- ...
UPDATE leave_requests SET status='CONFLICT'
    WHERE employee_id=55 AND date='2025-03-15';                -- Chờ Transaction A... DEADLOCK!
```

**Detect:**

```sql
SHOW ENGINE INNODB STATUS;
-- "LATEST DETECTED DEADLOCK" → thấy 2 transactions lock ngược nhau
-- Setup alert: "Lock wait timeout" > 5 lần/phút
```

**Solution: Consistent Lock Ordering + Composite Index + Retry**

```java
// FIX 1: LUÔN lock theo thứ tự: leave_requests TRƯỚC, employee_schedules SAU

@Service
public class LeaveApprovalService {

    @Transactional
    @Retryable(value = DeadlockException.class, maxAttempts = 3,
               backoff = @Backoff(delay = 100, multiplier = 2))
    public void approveLeave(Long leaveRequestId) {
        // Bước 1: Lock leave_requests TRƯỚC
        LeaveRequest leave = leaveRepo.findByIdWithLock(leaveRequestId);
        // Bước 2: Sau đó mới lock schedule
        EmployeeSchedule schedule = scheduleRepo
            .findByEmployeeAndDateWithLock(leave.getEmployeeId(), leave.getDate());

        leave.setStatus(LeaveStatus.APPROVED);
        schedule.setDayOff(true);
        leaveRepo.save(leave);
        scheduleRepo.save(schedule);
    }
}

@Service
public class ShiftAssignmentService {

    @Transactional
    @Retryable(value = DeadlockException.class, maxAttempts = 3,
               backoff = @Backoff(delay = 100, multiplier = 2))
    public void assignShift(Long employeeId, LocalDate date, Long shiftId) {
        // Bước 1: Lock leave_requests TRƯỚC — CÙNG THỨ TỰ!
        leaveRepo.findPendingByEmployeeAndDateWithLock(employeeId, date)
            .ifPresent(leave -> {
                leave.setStatus(LeaveStatus.CONFLICT);
                leaveRepo.save(leave);
            });
        // Bước 2: Sau đó mới lock schedule
        EmployeeSchedule schedule = scheduleRepo.findByEmployeeAndDateWithLock(employeeId, date);
        schedule.setShiftId(shiftId);
        scheduleRepo.save(schedule);
    }
}
```

```sql
-- FIX 2: Composite index — giảm lock scope
-- TRƯỚC: chỉ index (employee_id) → scan nhiều rows → lock nhiều rows
-- SAU: composite index → chỉ lock đúng 1 row cần thiết
ALTER TABLE employee_schedules ADD INDEX idx_employee_date (employee_id, date);

-- FIX 3: @Retryable — safety net, retry max 3 lần với exponential backoff (100ms, 200ms, 400ms)
```

> **Kết quả:** 0 deadlocks/ngày · P99 response time: 30s → <500ms · Throughput tăng 3×

> **Lessons learned:** Định nghĩa lock ordering convention cho team và enforce trong code review. Composite index không chỉ tăng tốc query mà còn giảm lock scope.

---

### Q5b: IAM & OAuth2 Authorization Service

**Interviewer:** Bạn implement OAuth2-based authorization và IAM service cho các enterprise platform nội bộ. Tại sao không dùng Keycloak hay Auth0? IAM được thiết kế thế nào — RBAC hay ABAC? Và làm sao HRM, GAM verify quyền mà không tight coupling với IAM?

**Thái Đông:**

**Tại sao không dùng Keycloak / Auth0?**
- **Permission model đặc thù:** GHTK có fine-grained business permissions như `LEAVE:APPROVE:OWN_DEPT`, `SHIFT:APPROVE:DEPT_LEAD`. Keycloak phù hợp generic auth, không model được điều này mà không hack phức tạp
- **Tự chủ data:** Dữ liệu nhân sự rất nhạy cảm. External SaaS = data đi ra ngoài, không phù hợp policy bảo mật nội bộ
- **Integration cost:** Keycloak vẫn cần thêm layer transform giữa Keycloak roles và business permissions — effort không kém tự build

**Permission Model: RBAC + Resource Scope**

RBAC làm base, extend thêm **resource scope** cho fine-grained permissions:

```sql
CREATE TABLE roles (
    id   BIGINT PRIMARY KEY,
    name VARCHAR(100) UNIQUE NOT NULL  -- 'HR_MANAGER', 'DEPT_LEAD', 'EMPLOYEE'
);

CREATE TABLE permissions (
    id       BIGINT PRIMARY KEY,
    action   VARCHAR(100) NOT NULL,   -- 'APPROVE', 'VIEW', 'EDIT', 'DELETE'
    resource VARCHAR(100) NOT NULL,   -- 'LEAVE_REQUEST', 'SHIFT', 'SALARY'
    scope    VARCHAR(100),            -- NULL = toàn công ty; 'OWN_DEPT' = chỉ phòng ban
    UNIQUE (action, resource, scope)
);

-- Examples:
-- ('APPROVE', 'LEAVE_REQUEST', NULL)       → HR Manager approve bất kỳ đơn nào
-- ('APPROVE', 'LEAVE_REQUEST', 'OWN_DEPT') → Dept Lead chỉ approve đơn trong phòng mình
-- ('VIEW',    'SALARY',        'OWN')      → Nhân viên chỉ xem lương bản thân

CREATE TABLE role_permissions (role_id BIGINT, permission_id BIGINT, PRIMARY KEY (role_id, permission_id));
CREATE TABLE user_roles       (user_id BIGINT, role_id BIGINT,       PRIMARY KEY (user_id, role_id));
```

**OAuth2 Flow:**

```
User / Admin Portal
        │
        │ 1. POST /oauth/token (grant_type=password)
        ▼
┌──────────────┐
│  IAM Service │ ← Validate credentials, load roles & permissions
│              │
│  Issue JWT:  │
│  {           │
│   sub: 12345,│
│   roles: ["HR_MGR"],
│   perms: ["APPROVE_LEAVE_OWN_DEPT"],
│   exp: +8h   │
│  }           │
└──────┬───────┘
        │ 2. access_token (JWT) + refresh_token
        ▼
Client (browser / mobile)
        │
        │ 3. Bearer token trong mọi request
        ▼
HRM / GAM Service
        │ 4. Verify JWT locally (verify signature) — KHÔNG gọi lại IAM
        ▼
Business Logic
```

**Service-to-service (Machine-to-Machine — Client Credentials flow):**

```http
POST /oauth/token
grant_type=client_credentials
client_id=scoring-service
client_secret=***

→ IAM trả JWT với scope: ["notification:send"]
→ Scoring Service dùng token này gọi Notification Service
```

**Verify quyền mà không tight coupling với IAM:**

Vấn đề: Nếu mỗi request phải gọi IAM để verify → 1.000 req/s → 1.000 calls/s sang IAM → IAM thành bottleneck và single point of failure.

**Solution: JWT Self-contained + Local Verification**

```java
// IAM ký JWT bằng RSA private key
// Mỗi service có public key để verify locally — KHÔNG gọi IAM

// Shared internal SDK — mỗi service include
@Component
public class JwtAuthFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest req, ...) {
        String token = extractBearerToken(req);
        Claims claims = jwtVerifier.verify(token); // local RSA verify, ~0.1ms

        SecurityContextHolder.getContext().setAuthentication(
            new JwtAuthenticationToken(claims)
        );
    }
}

// Sử dụng trong service:
@Service
public class LeaveApprovalService {

    @PreAuthorize("hasPermission('LEAVE_REQUEST', 'APPROVE')")
    public void approveLeave(Long leaveId, SecurityContext ctx) {
        Claims claims = (Claims) ctx.getAuthentication().getPrincipal();

        // Scope check: OWN_DEPT → chỉ approve đơn trong phòng mình
        if (claims.hasScope("OWN_DEPT")) {
            LeaveRequest leave = leaveRepo.findById(leaveId);
            if (!leave.getEmployeeDeptId().equals(claims.getDepartmentId())) {
                throw new AccessDeniedException("Không có quyền approve đơn ngoài phòng ban");
            }
        }
        // Business logic...
    }
}
```

**Revoke token sớm khi cần (blacklist trong Redis):**

```java
// Vấn đề: JWT TTL 8h. User bị revoke quyền lúc 10:00 nhưng token vẫn valid đến 18:00
// Solution: Token Blacklist trong Redis chỉ cho critical operations

// Khi revoke: SETEX blacklist:{jti} {remaining_ttl} "revoked"
// Trong filter:
if (isSensitiveOperation(req) && redis.hasKey("blacklist:" + claims.getId())) {
    throw new TokenRevokedException();
}
// Normal ops: chỉ verify signature, không check Redis → P99 < 1ms
```

**Kết quả & Trade-offs:**
- **0 IAM calls per request:** JWT self-contained → IAM chỉ bị gọi khi login/refresh, không phải single point of failure runtime
- **Shared SDK:** Permission logic centralize trong 1 library, mọi service dùng — thay đổi 1 chỗ
- **Trade-off chấp nhận:** Permission thay đổi chỉ có hiệu lực sau khi token cũ hết hạn (max 8h). Với business nội bộ là OK — không phải banking
- **Sensitive ops:** Blacklist Redis xử lý các case cần revoke ngay (nhân viên nghỉ đột xuất, tài khoản bị compromise)

> **Kết quả:** 0 IAM calls/request · <1ms auth overhead · Access 8h / Refresh 7d · 1 shared SDK cho mọi service

---

## 🎢 Phần 4 — Tomosia: Digital Transformation & Ticket Platform (15 phút)

**Stack:** Java Spring Boot · PHP Laravel · MySQL · Redis · AWS (Lambda, SQS, OpenSearch, CloudWatch)

---

### Q6: Inventory Model & Chống Oversell

**Interviewer:** Làm sao thiết kế inventory model và đảm bảo không bao giờ oversell khi hàng trăm người mua vé cùng lúc? Tại sao single `remaining` field không đủ?

**Thái Đông:**

**Tại sao single `remaining` field không đủ:**

Khi user chọn vé, hệ thống cần giữ chỗ tạm trong khi thanh toán (5–15 phút). Nếu chỉ dùng `remaining`, không thể phân biệt "chỗ đang được giữ chờ thanh toán" và "chỗ thực sự còn trống" → bán nhầm, hoặc hiển thị "sold out" sớm hơn thực tế.

**Inventory Model — Tách 3 trạng thái:**

```sql
CREATE TABLE daily_inventory (
    park_id         UUID NOT NULL,
    ticket_type_id  UUID NOT NULL,
    visit_date      DATE NOT NULL,
    total_capacity  INT NOT NULL,           -- 500 (admin set, ít thay đổi)
    reserved        INT NOT NULL DEFAULT 0, -- đang giữ chỗ tạm, chờ thanh toán
    booked          INT NOT NULL DEFAULT 0, -- đã thanh toán xong
    version         INT NOT NULL DEFAULT 0, -- optimistic locking
    PRIMARY KEY (park_id, ticket_type_id, visit_date),

    -- Safety net cuối cùng: dù code có bug cũng không oversell
    CONSTRAINT chk_no_oversell CHECK (reserved + booked <= total_capacity),
    CONSTRAINT chk_non_negative CHECK (reserved >= 0 AND booked >= 0)
);

-- available = total_capacity - reserved - booked (tính toán, KHÔNG lưu riêng)
```

**Lifecycle của 3 field:**

```sql
-- Khách chọn 5 vé → Reserve (giữ chỗ tạm):
    reserved += 5       -- available giảm 5

-- Payment thành công → Confirm:
    reserved -= 5, booked += 5  -- chuyển reserved → booked

-- Payment fail / TTL hết → Release:
    reserved -= 5       -- available tăng lại 5

-- Cancel booking đã confirm → Release:
    booked -= 5         -- available tăng lại 5
```

**Vấn đề race condition:**

```sql
-- Còn 6 chỗ. Cùng lúc: Khách A đặt 5 vé, Khách B đặt 3 vé
Thread A: READ available=6 → check 5≤6 ✓ → UPDATE reserved+=5
Thread B: READ available=6 → check 3≤6 ✓ → UPDATE reserved+=3
                ↑ đọc cùng giá trị cũ vì A chưa commit
-- Kết quả: bán 8 vé cho 6 chỗ → OVERSELL!
```

**Solution: Redis Lua (reject sớm) + DB Optimistic Lock (source of truth):**

```lua
-- Redis Lua script — atomic pre-check (không thể bị race condition)
local key      = KEYS[1]  -- "inventory:{park}:{type}:{date}"
local qty      = tonumber(ARGV[1])
local total    = tonumber(redis.call('HGET', key, 'total')    or 0)
local reserved = tonumber(redis.call('HGET', key, 'reserved') or 0)
local booked   = tonumber(redis.call('HGET', key, 'booked')   or 0)
local available = total - reserved - booked

if available < qty then
    return -1  -- hết chỗ, reject ngay tại Redis
end

redis.call('HINCRBY', key, 'reserved', qty)
return available - qty  -- remaining sau khi reserve
```

```sql
-- DB Optimistic Lock — source of truth
UPDATE daily_inventory
SET    reserved = reserved + :quantity,
       version  = version  + 1
WHERE  park_id        = :parkId
  AND  ticket_type_id = :ticketTypeId
  AND  visit_date     = :visitDate
  AND  version        = :expectedVersion           -- optimistic check
  AND  (total_capacity - reserved - booked) >= :quantity;  -- safety check
-- affected rows = 0 → conflict → rollback Redis, retry (max 3 lần)
```

```java
// Compensation: rollback Redis nếu DB conflict
if (affectedRows == 0) {
    redisClient.hIncrBy(key, "reserved", -quantity); // rollback
    if (retryCount < 3) retry();
    else throw new SoldOutException();
}
```

**Reconciliation job** chạy mỗi 30 giây sync Redis ↔ DB, sửa drift nếu app crash giữa chừng. Redis down → fallback dùng `SELECT FOR UPDATE` trực tiếp trên DB.

> **Kết quả:** 0 overselling incidents · <1ms Redis pre-check · DB load giảm 80–90% · 30s reconciliation interval

---

### Q7: Booking State Machine & TTL

**Interviewer:** Khi user chọn vé xong nhưng không thanh toán thì sao? Chỗ đó bị "treo" mãi à? Booking state machine thiết kế thế nào?

**Thái Đông:**

**Two-Phase Reservation với TTL:**

Booking tạo ở trạng thái `PENDING` với `expires_at = now + 15 phút`. Sau TTL, hệ thống tự release chỗ về pool.

```
User chọn vé → reserve inventory
                    │
                    ▼
              ┌──────────┐
              │  PENDING  │ ── expires_at = now + 15min
              └─────┬─────┘    (reserved inventory đang giữ)
                    │
    ┌───────────────┼───────────────┐
    │               │               │
    ▼               ▼               ▼
┌──────────┐  ┌─────────┐    ┌──────────┐
│CONFIRMED │  │ EXPIRED │    │CANCELLED │
│(đã trả)  │  │(hết TTL)│    │(user hủy)│
└─────┬────┘  └─────────┘    └──────────┘
      │        release inv    release inv
      ▼
┌──────────┐
│   USED   │ ── Đã scan QR vào cổng
└──────────┘
```

```sql
-- Booking record
INSERT INTO bookings (id, status, expires_at, quantity, ...)
VALUES (gen_random_uuid(), 'PENDING', NOW() + INTERVAL '15 minutes', 5, ...);

-- Job chạy mỗi 15–30 giây: expire PENDING quá hạn
UPDATE bookings
SET    status = 'EXPIRED'
WHERE  status = 'PENDING' AND expires_at < NOW()
RETURNING id, park_id, ticket_type_id, visit_date, quantity;
-- → Với mỗi expired booking: release inventory (reserved -= quantity)
```

**Quy tắc quan trọng của State Machine:**
- **EXPIRED và USED là terminal states** — không bao giờ revert
- **Payment FAILED ≠ EXPIRED:** Payment fail → booking vẫn `PENDING`, cho phép retry (max 3 lần). Chỉ expire khi hết TTL
- **CONFIRMED → CANCELLED** phải trigger refund async, release `booked` inventory
- **Price snapshot:** Giá được snapshot vào booking record tại thời điểm tạo. Payment dùng giá này — tránh "admin đổi giá giữa chừng"

> **Tại sao cho retry thay vì cancel ngay khi payment fail?** Cancel ngay → user phải đặt lại từ đầu, inventory release → có thể bị người khác mua mất. Cho retry (max 3, trong TTL) → user chỉ cần chọn phương thức khác, inventory vẫn giữ cho họ.

---

### Q8: Payment Reliability — Dual Confirmation & Outbox Pattern

**Interviewer:** Nếu user thanh toán thành công nhưng webhook callback bị miss (server đang deploy, network issue), hệ thống xử lý thế nào? Làm sao đảm bảo event sinh vé không bao giờ bị mất?

**Thái Đông:**

**Bài toán nguy hiểm nhất: "Tiền trừ nhưng vé không có"**

```
1. User thanh toán → gateway charge thành công
2. Gateway gửi webhook → server đang deploy → callback fail
3. Gateway retry vài lần → vẫn fail → ngừng retry
4. Hệ thống: payment = PENDING, booking = PENDING
5. Timeout job: booking EXPIRED, inventory released
6. User: "Tiền bị trừ nhưng không nhận được vé!"
```

**Solution: Dual Confirmation (Belt and Suspenders)**

Không thụ động chờ callback — chủ động query gateway:

```
Webhook callback ──────────────┐
(primary)                      │
                               ▼
                        ┌──────────────┐
Payment PENDING ─────── │ handleSuccess │ ── Booking CONFIRMED
                        └──────────────┘
                               ▲
Polling job ───────────────────┘
(fallback, mỗi 2 phút)
```

```java
// Job 1: Status Polling — chạy mỗi 2 phút
// Query gateway cho tất cả payment PENDING > 5 phút
List<Payment> stalePendings = paymentRepo
    .findByStatusAndCreatedAtBefore("PENDING", now().minusMinutes(5));

for (Payment p : stalePendings) {
    GatewayStatus status = gatewayClient.queryStatus(p.getGatewayTxnId());
    if (status == SUCCESS) {
        confirmBooking(p); // update reserved→booked, booking→CONFIRMED
    }
}

// Job 2: Timeout Expiry — chạy mỗi 30 giây
// QUAN TRỌNG: query gateway LẦN CUỐI trước khi expire
List<Payment> expiring = paymentRepo.findExpired();
for (Payment p : expiring) {
    // Scenario: user trả lúc 10:14:50, expires_at=10:15:00
    // callback chưa đến, nếu expire ngay → user mất tiền
    GatewayStatus status = gatewayClient.queryStatus(p.getGatewayTxnId());
    if (status == SUCCESS) {
        confirmBooking(p); // gateway nói thành công → confirm bất kể timeout
    } else {
        expirePayment(p);  // chỉ expire khi chắc chắn gateway không charge
    }
}
```

**Idempotency của webhook handler:**

```java
public void handleWebhookCallback(GatewayCallback callback) {
    Payment payment = paymentRepo.findByGatewayTxnId(callback.getTxnId());

    if (payment.getStatus() == SUCCESS) {
        return; // Đã xử lý, trả 200 OK để gateway không retry nữa
    }
    if (payment.getStatus() != PENDING) {
        throw new InvalidStateTransitionException();
    }
    if (!callback.getAmount().equals(payment.getAmount())) {
        throw new AmountMismatchException(); // chống tampering
    }

    confirmPaymentAndBooking(payment); // optimistic lock chặn race condition
}
```

**Xử lý callback đến SAU khi booking đã expire:**

```java
// 10:00 - Payment created (expires_at = 10:15)
// 10:15 - Timeout job expire booking, inventory released
// 10:16 - Callback đến: payment SUCCESS

// Quyết định: AUTO-REFUND — KHÔNG cố confirm lại
// Vì inventory đã release, có thể bị người khác mua
// Booking EXPIRED là terminal state → không revert
if (booking.getStatus() == EXPIRED) {
    refundService.initiateRefund(payment);
    notify(user, "Thanh toán trễ. Đã hoàn tiền, vui lòng đặt lại.");
}
```

**Transactional Outbox — Đảm bảo event không bao giờ mất:**

Vấn đề: Nếu app crash giữa `UPDATE booking` và `publish event` → booking CONFIRMED nhưng vé không được sinh.

```sql
-- GIẢI PHÁP: Lưu event vào DB CÙNG transaction với business logic
BEGIN;
  UPDATE bookings SET status = 'CONFIRMED' WHERE id = :bookingId;

  INSERT INTO outbox_events (event_type, payload, published)
  VALUES ('BookingConfirmed', :payload, FALSE);
COMMIT;
-- Cả 2 thành công hoặc cả 2 rollback — KHÔNG BAO GIỜ mất event

CREATE TABLE outbox_events (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    event_type TEXT    NOT NULL,
    payload    JSONB   NOT NULL,
    published  BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_outbox_unpublished ON outbox_events (created_at)
    WHERE published = FALSE;
```

```java
// Outbox publisher: background job chạy mỗi vài giây
List<OutboxEvent> unpublished = outboxRepo.findByPublished(false);
for (OutboxEvent e : unpublished) {
    messageBroker.publish(e.getEventType(), e.getPayload());
    outboxRepo.markPublished(e.getId());
    // Nếu publish fail → retry lần sau. Consumer phải idempotent.
}
```

```
BookingConfirmed event
  ├──▶ Ticketing Service  → sinh QR code / e-ticket
  └──▶ Notification Svc   → email + SMS "Đặt vé thành công"
```

**Reconciliation 3 tầng — safety net cuối cùng:**
- **Real-time:** Verify amount match tại mỗi callback
- **Near real-time (mỗi 2 phút):** Polling job query gateway cho PENDING payments
- **Daily batch (03:00 AM):** Download settlement report từ gateway, cross-check toàn bộ, alert nếu discrepancy

> **Kết quả:** 0 "tiền trừ vé không có" · ≤2 phút detect missed callback · 0 event bị mất (Outbox) · 3-level reconciliation

---

### Q9: AWS Data Pipeline

**Interviewer:** Ngoài ticket system, bạn xây data pipeline trên AWS. Pipeline làm gì và tại sao serverless?

**Thái Đông:**

**Context:** Data ingestion pipeline cho e-commerce — thu thập sản phẩm từ nhiều nguồn (websites đối tác, marketplace có JS-rendered content) để indexing và search.

```
CloudWatch Scheduler    SQS Queue     Lambda (Node.js)     OpenSearch
       │                    │                │                  │
       │ Trigger (mỗi 6h)   │                │                  │
       ├───────────────────►│                │                  │
       │  [url1, url2, ...]  │                │                  │
       │                    │ Dequeue batch  │                  │
       │                    ├───────────────►│                  │
       │                    │                │ Puppeteer:       │
       │                    │                │ headless Chrome  │
       │                    │                │ wait networkidle2│
       │                    │                │ extract data     │
       │                    │                │                  │
       │                    │                │ Transform &      │
       │                    │                │ validate         │
       │                    │                │                  │
       │                    │                │ bulk index ─────►│
       │                    │ fail → DLQ     │                  │
       │                    │◄───────────────┤                  │
       │                    │ (retry / alert)│                  │
```

**Tại sao serverless?** Crawl chạy 4 lần/ngày, mỗi lần ~30 phút — EC2 24/7 lãng phí 95% thời gian. Lambda chỉ tính tiền khi chạy. SQS tự scale concurrent executions. DLQ bắt mọi failure.

**Tại sao Puppeteer?** Nhiều trang dùng SPA (React/Vue) — HTTP request thuần chỉ nhận HTML skeleton. Puppeteer chạy headless Chrome, đợi JS render xong mới extract.

**Monitoring với CloudWatch:**
- Lambda metrics: duration, errors, throttles, concurrent executions
- SQS metrics: queue depth, messages in flight, DLQ count
- Custom metrics: products crawled/giờ, success rate, index latency
- Alarms: DLQ > 0 → alert; Lambda error rate > 5% → alert

---

## ⚡ Phần 5 — Cross-cutting Challenges & Growth (8 phút)

**Interviewer:** Bạn làm việc với nhiều tech stack: Java Spring Boot, PHP Laravel, Go, AWS Serverless. Đây là advantage hay disadvantage? Khi nào chọn ngôn ngữ/framework nào?

**Thái Đông:**

**Đây là advantage, nhưng cần có chiều sâu — mỗi lựa chọn có lý do cụ thể:**

| Scenario | Chọn | Lý do |
|---|---|---|
| CRUD-heavy admin, rapid prototyping | **Laravel** | Eloquent ORM, Blade, artisan — ship nhanh |
| Enterprise, domain logic phức tạp | **Spring Boot** | Strong typing, DI container, DDD support tốt |
| High-concurrency, low-latency service | **Go** | Goroutines, compiled, minimal memory footprint |
| Batch job, traffic không đều | **Lambda (Node.js)** | Pay-per-use, auto-scale, không quản lý server |

**Nguyên tắc:**
- "Right tool for the right job" — không ép 1 ngôn ngữ cho mọi bài toán
- **Team context quan trọng:** Nếu team chưa biết Go, training cost cũng là cost
- **Core concepts transferable:** DDD, SOLID, design patterns — áp dụng ở mọi ngôn ngữ

**Điểm cần cải thiện:**
- Go còn mới (bắt đầu từ PREP) — cần học sâu về Go-specific patterns (channels, context propagation, error wrapping)
- Distributed systems theory (consensus algorithms, CAP theorem trong thực tế)
- Observability stack — đã làm CloudWatch nhưng muốn học sâu Prometheus + Grafana + distributed tracing

---

## 🏗️ Phần 6 — System Design Question (10 phút)

**Interviewer:** Design một **Online Examination Platform** phục vụ **50.000 thí sinh đồng thời** (gấp 10x hiện tại). Focus vào quyết định kiến trúc quan trọng nhất.

**Thái Đông:**

**Clarify requirements:**
- 50.000 concurrent candidates = peak load khi bắt đầu và kết thúc kỳ thi
- Mỗi candidate gửi ~20 events/phút (anti-cheat) + 1 submission cuối
- Zero tolerance cho mất bài thi
- Anti-cheating real-time
- Score có trong <5 phút sau nộp bài

**Architecture tổng quan:**

```
┌──────────────────────────────────────────────────────────┐
│            CDN (CloudFront / Fastly)                      │
│         Static assets, Exam UI (React SPA)               │
└──────────────────────────┬───────────────────────────────┘
                           │
┌──────────────────────────┴───────────────────────────────┐
│         Load Balancer (ALB/NGINX) + WAF + Rate Limiting   │
└────┬──────────────────┬──────────────────┬────────────────┘
     │                  │                  │
┌────┴──────┐  ┌────────┴──────┐  ┌───────┴───────┐
│WebSocket  │  │  Exam API     │  │  Mgmt API     │
│Gateway(Go)│  │  (Go)         │  │  (Laravel)    │
│- Events   │  │- Bắt đầu thi  │  │- Khóa học     │
│  anti-    │  │- Lấy câu hỏi  │  │- Người dùng   │
│  cheat    │  │- Nộp bài      │  │- B2B config   │
│- Sync     │  │- Auto-save    │  │- Báo cáo      │
│  timer    │  │               │  │               │
└────┬──────┘  └────────┬──────┘  └───────┬───────┘
     └──────────────────┴──────────────────┘
                         │
┌────────────────────────┴──────────────────────────────────┐
│              Kafka Cluster (3 brokers, replication=3)      │
│  exam.submitted   (50 partitions,  acks=all)              │
│  cheat.events    (100 partitions, key=examId+candidateId) │
│  autosave.data   (100 partitions)                         │
│  score.calculated (50 partitions)                         │
└──────────┬────────────────────┬──────────────────────────┘
           │                    │
┌──────────┴──────┐    ┌────────┴────────────┐
│ Anti-Cheat Svc  │    │  Scoring Pipeline    │
│ (Go, 20 replicas│    │  (Go, 10 replicas)   │
│ Redis buffer    │    │  Auto-grade          │
│ Rules engine    │    │  Batch 500 subs      │
└─────────────────┘    └──────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                      Data Layer                          │
│  PostgreSQL              Redis Cluster     Object Storage │
│  (Primary + 2 Replicas)  (6 nodes)         (S3)          │
│  Shard by exam_id        - Cheat buffer    - Đề thi       │
│                          - Session         - Bài nộp     │
│                          - Auto-save       - Audit logs  │
└──────────────────────────────────────────────────────────┘
```

**Quyết định kiến trúc 1: WebSocket Gateway tách riêng**

50K × 20 events/phút = ~1M events/phút chỉ riêng anti-cheat. WebSocket connections là long-lived và memory-intensive. Tách riêng để:
- Scale WebSocket layer độc lập với REST API
- Go WebSocket server giữ 50K connections với ~2–4 GB RAM
- WebSocket service restart không ảnh hưởng exam API

**Quyết định kiến trúc 2: Auto-save qua Redis (không thẳng DB)**

```go
// 50K × 1 save/30s = ~1.700 writes/giây → nếu save thẳng DB sẽ overwhelm PostgreSQL

// SOLUTION: Save vào Redis, flush vào DB mỗi 5 phút
func (s *AutoSaveService) SaveProgress(ctx context.Context, req AutoSaveRequest) error {
    key := fmt.Sprintf("autosave:%s:%s", req.ExamID, req.CandidateID)
    data, _ := json.Marshal(req.Answers)
    return s.redis.Set(ctx, key, data, 2*time.Hour).Err() // O(1), < 1ms
}

// Background job: flush Redis → PostgreSQL mỗi 5 phút
func (s *AutoSaveService) FlushToDB(ctx context.Context) {
    keys, _ := s.redis.Keys(ctx, "autosave:*").Result()
    // ...
    s.repo.BulkUpsert(ctx, batch) // 1 round-trip thay vì 50K round-trips
}
```

**Quyết định kiến trúc 3: Xử lý "thundering herd" khi hết giờ**

```go
// Vấn đề: 50K thí sinh hết giờ cùng lúc → spike submit cực lớn

func (s *ExamService) ScheduleAutoSubmit(examID string, endsAt time.Time) {
    candidates, _ := s.repo.GetCandidatesByExam(examID)

    for _, c := range candidates {
        // Random jitter: 0–30 giây → trải đều spike thành 30s window
        jitter := time.Duration(rand.Intn(30)) * time.Second
        submitAt := endsAt.Add(jitter)

        s.scheduler.Schedule(submitAt, func() {
            s.AutoSubmitIfNotDone(examID, c.ID)
        })
    }
    // Client hiển thị "Hết giờ" ngay, nhưng actual submit được trải đều
    // → Giảm peak load từ 50K/s xuống ~1.700/s
}
```

> **Trade-offs:** WebSocket tách riêng (tăng ops complexity nhưng cần cho isolation) · Redis auto-save (có thể mất tối đa 5 phút nếu Redis crash, nhưng acceptable vì có client-side backup) · Staggered submit (user thấy "Đang nộp..." thêm 0–30 giây, hệ thống ổn định hơn nhiều)

---

## 📝 Tổng kết & Đánh giá

| Tiêu chí | Đánh giá | Nhận xét |
|---|---|---|
| Domain Knowledge | ⭐⭐⭐⭐⭐ Strong | Đa dạng: EdTech, Enterprise, Digital Transformation, Data Pipeline |
| Architecture & Design | ⭐⭐⭐⭐⭐ Strong | DDD, Hexagonal, Event-Driven — hiểu trade-offs và áp dụng đúng context |
| Technical Depth | ⭐⭐⭐⭐ Good | Multi-language, concurrency, caching, messaging. Go còn đang phát triển |
| Problem Solving | ⭐⭐⭐⭐⭐ Strong | Deadlock resolution, anti-cheat design, overselling prevention |
| Communication | ⭐⭐⭐⭐⭐ Strong | Giải thích rõ ràng, có diagram, biết nói về trade-offs trung thực |
| Self-awareness | ⭐⭐⭐⭐⭐ Strong | Nhận biết điểm cần cải thiện (Go depth, distributed systems theory) |

**Điểm mạnh nổi bật:**
- Kinh nghiệm thực tế với event-driven architecture, Kafka, Redis across multiple projects
- Hiểu và áp dụng DDD + Hexagonal Architecture để cải thiện maintainability
- Đã xử lý production incidents (deadlocks, performance bottlenecks) — không chỉ lý thuyết
- Multi-language capability với lý do rõ ràng cho mỗi lựa chọn
- System design thinking tốt: biết dùng Redis buffer thay vì Kafka cho high-frequency events

**Cần chuẩn bị thêm trước phỏng vấn thật:**
- Chuẩn bị metrics cụ thể (TPS, latency p95/p99) từ dự án thực tế để tăng tính thuyết phục
- Học sâu hơn Saga pattern: orchestration vs choreography, Outbox pattern
- Thực hành system design ở scale lớn hơn (100K+ concurrent)
- Tìm hiểu distributed tracing (Jaeger/Zipkin) và Prometheus + Grafana stack
- Ôn lại Go-specific patterns: channels, context propagation, error wrapping conventions

---

> **Kết luận: STRONG HIRE — Senior Backend Engineer**
>
> Ứng viên thể hiện chiều sâu kỹ thuật tốt và kinh nghiệm thực tế đa dạng. Đặc biệt ấn tượng với khả năng giải quyết production incidents (deadlock), thiết kế hệ thống real-time (anti-cheating), và áp dụng DDD/Hexagonal Architecture có chủ đích, không phải áp dụng vì trào lưu.
>
> Phù hợp vị trí **Senior Backend Engineer**. Tiềm năng phát triển thành **Tech Lead** khi Go experience và distributed systems knowledge trưởng thành hơn.

---

*Tổng thời gian: ~90 phút · Chúc Thái Đông phỏng vấn thành công! 💪*
