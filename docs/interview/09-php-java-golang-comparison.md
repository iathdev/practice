# So sánh PHP vs Java vs Golang

> Dùng khi bị hỏi: "Tại sao chọn Golang thay vì PHP hay Java?"

---

## Bảng so sánh nhanh

| | PHP | Java/Spring | Go |
|---|---|---|---|
| Concurrency | Mỗi request 1 process/thread — nặng | Thread nặng (~1MB), thread pool phức tạp | Goroutine nhẹ (~4KB), tạo hàng nghìn dễ dàng |
| GC pause | Không có GC (request-scoped) | Stop-the-world có thể > 10ms | GC pause < 1ms — đảm bảo real-time check |
| Long-running process | Không phù hợp — thiết kế request/response | Được, nhưng nặng | Phù hợp — binary chạy liên tục 24/7 |
| Pipeline / background job | Cần thêm queue worker riêng (Laravel Queue) | Được nhưng boilerplate nhiều | Goroutine + channel built-in, gọn |
| Graceful shutdown | Khó — không có lifecycle hook chuẩn | ShutdownHook có nhưng phức tạp | `context` + `signal.NotifyContext` tự nhiên |
| Memory footprint | Thấp nhưng không có concurrency tốt | JVM overhead ~200MB+ khi start | Binary nhỏ, start nhanh, RAM thấp |
| Startup time | Nhanh | Chậm (JVM warm-up) | Rất nhanh — phù hợp container/K8s |

---

## Khi nào dùng cái nào?

**PHP** — phù hợp cho:
- Web app truyền thống, request/response đơn giản
- Team quen Laravel/Symfony
- Không cần xử lý concurrent cao

**Java/Spring** — phù hợp cho:
- Enterprise system, nhiều integration
- Team lớn, cần ecosystem phong phú (Spring Security, JPA...)
- Chấp nhận JVM overhead và boilerplate

**Go** — phù hợp cho:
- Service cần throughput cao, latency thấp
- Long-running process, pipeline architecture
- Container/K8s — startup nhanh, binary nhỏ

---

## Câu trả lời mẫu

> "PHP không phù hợp cho long-running service cần xử lý 20K events/giây liên tục — nó được thiết kế cho request/response, muốn có background job phải thêm queue worker riêng. Java được, nhưng GC stop-the-world có thể vi phạm SLA real-time < 1ms, và boilerplate nhiều hơn. Go phù hợp nhất vì goroutine + context + select khớp tự nhiên với pipeline architecture: ingestion goroutine và flush goroutine chạy song song, graceful shutdown chỉ cần cancel context là toàn bộ pipeline dừng đúng thứ tự."

---

## "Go và Java chống race condition khác nhau thế nào?"

### Go — ưu tiên channel, tránh share memory

```go
// Truyền data qua channel — chỉ 1 goroutine giữ data tại 1 thời điểm
ch := make(chan int, 1)
go func() { ch <- <-ch + 1 }()

// Khi cần shared state — dùng Mutex
var mu sync.Mutex
mu.Lock()
count++
mu.Unlock()
```

### Java — lock shared memory

```java
// synchronized — lock cả method
public synchronized void increment() { count++; }

// AtomicInteger — CAS, không cần lock (nhanh hơn)
AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();

// volatile — đảm bảo visibility, KHÔNG đảm bảo atomicity
volatile boolean running = true;
```

### So sánh

| | Go | Java |
|---|---|---|
| Triết lý | Share bằng channel, tránh shared memory | Lock shared memory |
| Atomic counter | `sync/atomic` | `AtomicInteger` (CAS) |
| Lock | `sync.Mutex` | `synchronized` / `ReentrantLock` |
| Dễ sai | Ít hơn — channel trực quan | Dễ quên unlock, dùng volatile sai chỗ, deadlock |
| Detect race | `go run -race` | ThreadSanitizer |

### Trong anti-cheat — dùng Redis Lua thay cả 2

```lua
-- Atomic trên Redis, single-threaded
-- Không cần mutex, không cần channel
local count = redis.call('INCR', key)
if count > threshold then
    redis.call('SET', flagKey, 'cheating')
end
```

10K thí sinh đồng thời INCR cùng 1 key → Redis Lua đảm bảo atomic tự nhiên, không race condition.

### Câu trả lời mẫu

> "Go ưu tiên channel — truyền data qua channel thay vì share memory, chỉ 1 goroutine giữ data tại 1 thời điểm nên không thể race. Khi cần shared state thì dùng Mutex. Java thì ngược lại — lock shared memory bằng synchronized hoặc AtomicInteger (CAS). Java có nhiều tool hơn nhưng cũng dễ sai hơn: quên unlock, dùng volatile sai chỗ, deadlock do lock sai thứ tự. Trong bài toán anti-cheat, em dùng Redis Lua script — chạy single-threaded trên Redis nên atomic tự nhiên, không cần xử lý race condition ở application layer."
