# Anti-Cheat Service — Detection Rules & Kịch bản thực tế

> Hệ thống phát hiện gian lận 2 lớp: real-time rules (Redis) xử lý < 1ms, batch rules (TimescaleDB) phân tích pattern phức tạp. File này đi sâu vào từng rule, ví dụ SQL, kịch bản timeline, và cách action severity hoạt động.

---

## Mục lục

1. [Real-time Rules — Lớp 1 (Redis)](#1-real-time-rules--lớp-1-redis)
2. [Batch Analysis Rules — Lớp 2 (TimescaleDB)](#2-batch-analysis-rules--lớp-2-timescaledb)
3. [Action Severity — Xử lý detection](#3-action-severity--xử-lý-detection)
4. [Kịch bản thực tế](#4-kịch-bản-thực-tế)
5. [Ví dụ SQL cho Batch Rules](#5-ví-dụ-sql-cho-batch-rules)
6. [Rule Engine Design](#6-rule-engine-design)
7. [Interview Quick Reference](#7-interview-quick-reference)

---

## 1. Real-time Rules — Lớp 1 (Redis)

### Đặc điểm

- **Latency**: < 1ms (Redis INCR + Lua script)
- **Trigger**: mỗi event đến, check ngay TRƯỚC khi push vào buffer
- **Mục đích**: bắt hành vi rõ ràng, không cần context nhiều events
- **Cơ chế**: Redis counter với TTL + threshold check

### Bảng Rules

| Rule | Trigger | Threshold | Action | TTL Window |
|------|---------|-----------|--------|------------|
| Tab switch quá nhiều | `tab_switch` count | > 5 lần | Warning trên UI | 10 phút |
| Tab ẩn quá lâu | `tab_hidden` duration | > 30 giây | Warning + log | Per event |
| Paste trong Writing | `paste_detected` length | > 100 chars | Flag ngay | Per event |
| DevTools mở | `devtools_open` | Bất kỳ | Flag + cảnh báo nghiêm trọng | Ngay lập tức |
| Automation detected | `automation_signal` | Bất kỳ | Void exam ngay | Ngay lập tức |
| Focus lost quá nhiều | `focus_lost` count | > 10 lần / section | Warning | 30 phút (1 section) |

### Cơ chế Redis Counter + Lua Script

**Tại sao Lua script mà không phải INCR rồi GET riêng?**

```
Nếu tách INCR và GET:
  Thread A: INCR → count = 5 (chưa đọc)
  Thread B: INCR → count = 6
  Thread A: GET → count = 6 → WARNING (sai, A là lần thứ 5)

Lua script (atomic):
  local count = redis.call('INCR', key)
  if count == 1 then redis.call('EXPIRE', key, ttl) end
  return count
  → Mỗi thread nhận đúng count của mình
```

### Redis Key Pattern

```
Real-time counters:
  rt:{session_id}:tab_switch:count          TTL 600s (10 phút)
  rt:{session_id}:focus_lost:{section}:count  TTL 1800s (30 phút)

Tại sao key theo session_id?
  - 1 user có thể thi nhiều lần → mỗi session count riêng
  - Session kết thúc → key tự expire → không cần cleanup
```

---

## 2. Batch Analysis Rules — Lớp 2 (TimescaleDB)

### Đặc điểm

- **Latency**: 60 giây (flush interval) + query time
- **Trigger**: sau mỗi buffer flush + khi exam kết thúc (`exam_end` event)
- **Mục đích**: phát hiện pattern phức tạp cần cross-reference nhiều events
- **Cơ chế**: SQL queries trên TimescaleDB (window functions, aggregates, joins)

### Bảng Rules

| Rule | Phân tích | Threshold | Phát hiện | Khi nào chạy |
|------|----------|-----------|-----------|-------------|
| **Tab-then-correct** | Chuyển tab → quay lại → trả lời đúng trong 60s | correct_after_tab > 5 | Tra cứu đáp án bên ngoài | Sau flush + cuối bài |
| **Paste ratio** | % bài Writing là paste vs typed | paste_ratio > 50% | Copy từ ChatGPT/nguồn ngoài | Sau flush + cuối bài |
| **Answer speed anomaly** | Thời gian trả lời per câu vs trung bình | < 5 giây cho reading passage | Biết trước đáp án hoặc random click | Sau flush + cuối bài |
| **Typing rhythm** | Stddev của keystroke interval | stddev < 10ms | Bot tự động gõ phím | Cuối bài thi |
| **Cross-exam similarity** | So sánh answer pattern giữa 2 thí sinh | similarity > 90% | Chia sẻ đáp án | Chỉ cuối bài thi |
| **Section timing anomaly** | Tổng thời gian per section | < 20% thời gian trung bình | Skip hoặc biết trước đáp án | Cuối bài thi |
| **Device/IP anomaly** | IP/device thay đổi giữa bài thi | Bất kỳ thay đổi | Nhờ người khác làm thay | Sau flush |

### So sánh Real-time vs Batch

| | Real-time (Lớp 1) | Batch (Lớp 2) |
|---|---|---|
| **Data cần** | 1 event + counter | Nhiều events cross-reference |
| **Ví dụ** | "Tab switch lần thứ 6" | "8/8 lần tab switch đều trả lời đúng" |
| **Kết luận** | Hành vi đáng ngờ | Pattern gian lận rõ ràng |
| **Action** | Warning (nhẹ) | Flag/Void (nặng) |
| **False positive** | Cao hơn (chỉ nhìn 1 tín hiệu) | Thấp hơn (cross-reference nhiều tín hiệu) |

---

## 3. Action Severity — Xử lý detection

### Severity Levels

```
┌──────────────────────────────────────────────────────────────────┐
│  severity=low      → Flag cho admin review sau                    │
│                      Không ảnh hưởng thí sinh trong lúc thi       │
│                      Ví dụ: mouse_leave nhiều lần                 │
├──────────────────────────────────────────────────────────────────┤
│  severity=medium   → Cảnh báo thí sinh (hiện warning trên UI)    │
│                      "Bạn đã chuyển tab nhiều lần.                │
│                       Hành vi này được ghi nhận."                 │
│                      Ví dụ: tab_switch > 5 lần                   │
├──────────────────────────────────────────────────────────────────┤
│  severity=high     → Flag exam result + notify admin ngay         │
│                      Admin review trong dashboard                 │
│                      Ví dụ: tab-then-correct pattern 8/8 câu     │
├──────────────────────────────────────────────────────────────────┤
│  severity=critical → Void exam result + lock account              │
│                      Tự động, không chờ admin review              │
│                      Ví dụ: automation_signal, paste ratio 97%    │
└──────────────────────────────────────────────────────────────────┘
```

### Tại sao có 4 levels chứ không phải chỉ "gian lận" / "không gian lận"?

```
Chỉ 2 levels (binary):
  ✗ false positive → void exam oan → thí sinh khiếu nại → đối tác B2B mất niềm tin
  ✗ false negative → bỏ sót gian lận → kết quả không đáng tin

4 levels (graduated):
  ✓ low/medium: cảnh báo, ghi nhận — admin quyết định sau
  ✓ high: flag nhưng cần admin review — giảm false positive
  ✓ critical: chỉ cho hành vi chắc chắn (automation, paste 97%)
  ✓ Phù hợp B2B: đối tác thấy hệ thống vừa nghiêm khắc vừa công bằng
```

### Notification Flow

```
Detection xảy ra
       │
       ├── severity = low
       │     └── INSERT cheat_detections (admin review sau)
       │
       ├── severity = medium
       │     ├── INSERT cheat_detections
       │     └── Redis PUBLISH "exam:warning:{exam_id}:{user_id}"
       │           → JS SDK subscribe → hiện warning overlay
       │
       ├── severity = high
       │     ├── INSERT cheat_detections
       │     └── Redis PUBLISH "exam:admin:alerts"
       │           → Admin dashboard nhận alert real-time
       │
       └── severity = critical
              ├── INSERT cheat_detections
              ├── Redis PUBLISH warning cho thí sinh
              ├── Redis PUBLISH alert cho admin
              └── Call Exam Service: void result + lock account
```

---

## 4. Kịch bản thực tế

### Kịch bản 1: Thí sinh tra Google Translate trong Reading

```
Timeline:
  14:00:00 - exam_start (IELTS Reading Mock Test)
  14:12:30 - Reading passage 2, question 12
  14:12:31 - tab_switch (hidden_duration: 12s)         ← chuyển tab
  14:12:43 - focus_gained                               ← quay lại
  14:12:45 - answer_submit (question 12, correct: true) ← trả lời đúng

  Pattern lặp lại 8 lần trong 20 phút

Lớp 1 (Real-time):
  14:12:31 - tab_switch count = 1 → OK
  14:15:02 - tab_switch count = 3 → OK
  14:18:45 - tab_switch count = 6 → VƯỢT THRESHOLD
    → Detection: severity=medium
    → Redis PUBLISH warning
    → UI hiện: "Bạn đã chuyển tab nhiều lần. Hành vi này được ghi nhận."

Lớp 2 (Batch — sau flush + cuối bài thi):
  Rule: tab-then-correct pattern
  → Query: 8 lần tab switch, 8 lần trả lời đúng trong 60s sau đó
  → correct_after_tab = 8 > threshold 5
  → Tỷ lệ đúng sau tab: 100% (bình thường reading section ~40%)
  → Detection: severity=high, action=flag_result
  → Admin nhận alert → review evidence → xác nhận gian lận → void result
```

### Kịch bản 2: Paste bài Writing từ ChatGPT

```
Timeline:
  15:00:00 - Writing Task 2 bắt đầu
  15:00:00 → 15:02:00 - typing_pattern: 0 keystrokes (ngồi chờ)
  15:02:01 - paste_detected (length: 287 chars)          ← paste đoạn 1
  15:02:05 - typing_pattern: 12 keystrokes (sửa lỗi nhỏ)
  15:03:00 - paste_detected (length: 195 chars)          ← paste đoạn 2
  15:03:02 - answer_submit

Lớp 1 (Real-time):
  15:02:01 - paste_detected, length = 287 > 100 chars, section = writing
    → Detection: severity=high, action=flag
    → Cảnh báo: "Hành vi paste đã được ghi nhận."

Lớp 2 (Batch — cuối bài thi):
  Rule: paste_ratio
  → total_pasted = 287 + 195 = 482 chars
  → total_typed = 12 chars
  → paste_ratio = 482 / (482 + 12) = 97.6% >> threshold 50%
  → Detection: severity=critical, action=void_result

  Rule: typing_rhythm
  → 12 keystrokes trong 4 giây → đều là sửa lỗi nhỏ
  → Không có pattern typing thật sự → confirms paste behavior
```

### Kịch bản 3: Hai thí sinh chia sẻ đáp án (cùng phòng thi)

```
Thí sinh A: answers = [B, A, C, D, B, A, C, C, D, A, B, D, C, A, ...]
Thí sinh B: answers = [B, A, C, D, B, A, C, C, D, A, B, D, C, B, ...]
                                                               ↑ khác 1 câu
Similarity: 50/52 = 96.2% (cả 2 sai cùng những câu giống nhau)

Lớp 2 (Batch — SAU bài thi):
  Rule: cross_exam_similarity
  → Query so sánh answer pattern mọi cặp thí sinh trong exam
  → Pair (A, B): similarity = 96.2% > threshold 90%
  → Detection: severity=high cho CẢ 2 thí sinh
  → Admin review:
    - Cùng exam session, cùng time window
    - IP khác nhau (2 device)
    - Nhưng sai cùng câu, chọn cùng đáp án sai → copy nhau
    → Confirm: void result cả 2

  Lưu ý: rule này O(n²) cho n thí sinh → chỉ chạy SAU bài thi, không chạy mỗi flush
```

### Kịch bản 4: Bot tự động làm bài (automation)

```
Timeline:
  16:00:00 - exam_start
  16:00:01 - automation_signal
    → Phát hiện: navigator.webdriver = true
    → Hoặc: window.chrome.runtime undefined (headless)

Lớp 1 (Real-time):
  NGAY LẬP TỨC:
    → Detection: severity=critical
    → Action: void exam + lock account
    → Không cần chờ batch analysis
    → Thông báo: "Phát hiện phần mềm tự động. Bài thi đã bị hủy."

  Tại sao void ngay?
  - Automation = chắc chắn gian lận, không có false positive
  - Nếu chờ batch → bot đã xong bài → quá trễ
  - Lock account ngăn tạo session mới
```

### Kịch bản 5: Thí sinh hợp lệ bị cảnh báo nhầm (false positive)

```
Timeline:
  17:00:00 - exam_start
  17:10:00 - tab_switch (notification popup từ OS)
  17:10:01 - focus_gained ngay (dismiss notification, quay lại ngay)
  17:12:00 - tab_switch (notification popup lần 2)
  17:14:00 - tab_switch (notification popup lần 3)
  ...pattern lặp lại vì thí sinh không tắt notification

Lớp 1 (Real-time):
  tab_switch count = 6 → WARNING
  → Thí sinh thấy warning → tắt notification → không chuyển tab nữa

Lớp 2 (Batch — cuối bài):
  Rule: tab-then-correct
  → 6 lần tab switch, chỉ 2 lần trả lời đúng sau đó (33%)
  → 33% < bình thường (40%) → KHÔNG phải pattern tra cứu
  → Không tạo detection → thí sinh không bị flag

  Đây chính là lý do cần Lớp 2:
  - Lớp 1 cảnh báo (đúng, vì tab switch nhiều thật)
  - Lớp 2 không flag (đúng, vì không có pattern gian lận)
  - Thí sinh được bảo vệ bởi graduated severity
```

---

## 5. Ví dụ SQL cho Batch Rules

### 5.1. Tab-then-correct pattern

```sql
WITH tab_switches AS (
    SELECT user_id, exam_id, question_id, created_at as switch_time
    FROM exam_events
    WHERE action = 'tab_switch'
      AND exam_id = :examId
),
answers AS (
    SELECT user_id, exam_id, question_id, created_at as answer_time,
           metadata->>'is_correct' as is_correct
    FROM exam_events
    WHERE action = 'answer_submit'
      AND exam_id = :examId
)
SELECT
    a.user_id,
    COUNT(*) as tab_then_answer_count,
    COUNT(*) FILTER (WHERE a.is_correct = 'true') as correct_after_tab
FROM answers a
JOIN tab_switches t
    ON a.user_id = t.user_id
    AND a.question_id = t.question_id
    AND a.answer_time BETWEEN t.switch_time AND t.switch_time + INTERVAL '60 seconds'
GROUP BY a.user_id
HAVING COUNT(*) FILTER (WHERE a.is_correct = 'true') > :threshold;
-- Thí sinh chuyển tab rồi trả lời đúng > 5 lần = rất đáng ngờ
```

### 5.2. Paste ratio trong Writing

```sql
WITH paste_stats AS (
    SELECT user_id,
        COALESCE(SUM((metadata->>'pasted_text_length')::int), 0) as total_pasted,
        COALESCE(SUM((metadata->>'total_typed_chars')::int), 0) as total_typed
    FROM exam_events
    WHERE exam_id = :examId
      AND action IN ('paste_detected', 'typing_pattern')
      AND section_type = 'writing'
    GROUP BY user_id
)
SELECT user_id, total_pasted, total_typed,
    total_pasted::float / NULLIF(total_pasted + total_typed, 0) as paste_ratio
FROM paste_stats
WHERE total_pasted::float / NULLIF(total_pasted + total_typed, 0) > :threshold;
```

### 5.3. Answer speed anomaly

```sql
WITH answer_times AS (
    SELECT user_id, question_id, section_type, created_at,
        LAG(created_at) OVER (
            PARTITION BY user_id, section_type ORDER BY created_at
        ) as prev_answer_time
    FROM exam_events
    WHERE exam_id = :examId
      AND action = 'answer_submit'
)
SELECT user_id,
    COUNT(*) as fast_answer_count,
    MIN(EXTRACT(EPOCH FROM (created_at - prev_answer_time))) as min_answer_sec
FROM answer_times
WHERE prev_answer_time IS NOT NULL
  AND EXTRACT(EPOCH FROM (created_at - prev_answer_time)) < :minSpeedSec
GROUP BY user_id
HAVING COUNT(*) >= 3;
-- 3 câu trả lời nhanh bất thường → đáng ngờ
```

### 5.4. Typing rhythm (bot detection)

```sql
SELECT user_id,
    AVG((metadata->>'keystroke_interval_ms')::float) as avg_interval,
    STDDEV((metadata->>'keystroke_interval_ms')::float) as stddev_interval,
    COUNT(*) as sample_count
FROM exam_events
WHERE exam_id = :examId
  AND action = 'typing_pattern'
  AND metadata->>'keystroke_interval_ms' IS NOT NULL
GROUP BY user_id
HAVING COUNT(*) >= 20                                         -- đủ sample
  AND STDDEV((metadata->>'keystroke_interval_ms')::float) < :maxStddev;
-- stddev < 10ms → gõ đều bất thường → khả năng bot cao
```

### 5.5. Cross-exam similarity

```sql
WITH user_answers AS (
    SELECT user_id, question_id,
        metadata->>'selected_answer' as answer
    FROM exam_events
    WHERE exam_id = :examId
      AND action = 'answer_submit'
      AND metadata->>'selected_answer' IS NOT NULL
),
user_pairs AS (
    SELECT a.user_id as user_a, b.user_id as user_b,
        COUNT(*) as total_questions,
        COUNT(*) FILTER (WHERE a.answer = b.answer) as matching_answers
    FROM user_answers a
    JOIN user_answers b
        ON a.question_id = b.question_id
        AND a.user_id < b.user_id           -- tránh duplicate pairs
    GROUP BY a.user_id, b.user_id
)
SELECT user_a, user_b, total_questions, matching_answers,
    matching_answers::float / NULLIF(total_questions, 0) as similarity
FROM user_pairs
WHERE total_questions >= 10                   -- đủ câu để so sánh
  AND matching_answers::float / NULLIF(total_questions, 0) > :threshold;
```

---

## 6. Rule Engine Design

### Interface

```
Rule Engine nhận list events (sau flush) hoặc exam_id (cuối bài thi)
    → chạy tất cả rules đã đăng ký
    → thu thập detections
    → gửi cho Action Executor
```

### Nguyên tắc thiết kế

| Nguyên tắc | Giải thích |
|-----------|-----------|
| **Pluggable rules** | Mỗi rule implement cùng interface. Thêm rule mới = thêm 1 file, register vào engine |
| **Configurable thresholds** | Threshold load từ config (env vars). Thay đổi không cần redeploy |
| **Idempotent** | Chạy cùng rule 2 lần trên cùng data → kết quả giống nhau. Detection có unique ID |
| **Fail-safe** | 1 rule error → log + skip, không block các rule khác |

### Khi nào trigger?

```
Trigger 1: Sau mỗi buffer flush (60 giây)
  → Chạy TẤT CẢ rules (trừ cross-exam similarity)
  → Dữ liệu: events vừa flush + events trước đó
  → Mục đích: phát hiện sớm trong lúc thi

Trigger 2: Khi exam kết thúc (exam_end event)
  → Chạy TẤT CẢ rules BAO GỒM cross-exam similarity
  → Dữ liệu: toàn bộ events của bài thi
  → Mục đích: phân tích toàn diện cuối bài

Tại sao cross-exam similarity chỉ chạy cuối bài?
  → O(n²) cho n thí sinh: 1,000 users → 500,000 pairs
  → Chạy mỗi 60 giây = lãng phí (data chưa đủ)
  → Cuối bài mới có đủ answers để so sánh
```

---

## 7. Interview Quick Reference

### "Detection rules hoạt động thế nào?"

> "2 lớp. Lớp 1 dùng Redis counter + Lua script, check mỗi event < 1ms — bắt hành vi rõ ràng (tab switch > 5, devtools, automation). Lớp 2 dùng TimescaleDB SQL, chạy sau mỗi flush 60 giây — phát hiện pattern phức tạp (tab-then-correct, paste ratio, cross-exam similarity). Lớp 1 cảnh báo, Lớp 2 kết luận."

### "Làm sao tránh false positive?"

> "Graduated severity. Tab switch nhiều = warning (medium), không void. Chỉ khi Lớp 2 confirm pattern (8/8 lần tab switch đều trả lời đúng) mới flag (high). Chỉ automation_signal và paste ratio >90% mới void (critical). Admin review cho severity=high trước khi action. Thí sinh có cơ hội giải thích."

### "Cross-exam similarity có tốn không?"

> "O(n²) pair comparison. 1,000 thí sinh = 500,000 pairs. Chỉ chạy cuối bài thi, không chạy mỗi flush. Optimize: filter early (chỉ so cặp có >= 10 câu chung), `user_id < user_id` tránh duplicate, consider locality-sensitive hashing nếu scale lên."

### "Kịch bản nguy hiểm nhất?"

> "False positive ở severity=critical: void exam oan → thí sinh khiếu nại → đối tác B2B mất niềm tin. Giải pháp: chỉ auto-void cho automation_signal (chắc chắn 100%) và paste ratio cực cao (>90%). Còn lại đều cần admin review. Trong B2B, mỗi lần void sai = risk mất hợp đồng."

### Bảng tổng hợp rules

| Rule | Lớp | Trigger | Severity | False Positive Risk |
|------|-----|---------|----------|-------------------|
| Tab switch count | 1 | Mỗi event | Medium | Cao (notification OS) |
| Tab hidden duration | 1 | Mỗi event | Medium | Trung bình |
| Paste > 100 chars | 1 | Mỗi event | High | Thấp (writing section) |
| DevTools open | 1 | Mỗi event | High | Rất thấp |
| Automation signal | 1 | Mỗi event | Critical | Gần 0 |
| Tab-then-correct | 2 | Flush + cuối bài | High | Thấp |
| Paste ratio | 2 | Flush + cuối bài | High/Critical | Rất thấp |
| Answer speed | 2 | Flush + cuối bài | High | Trung bình |
| Typing rhythm | 2 | Cuối bài | Critical | Thấp |
| Cross-exam similarity | 2 | Chỉ cuối bài | High | Thấp |
