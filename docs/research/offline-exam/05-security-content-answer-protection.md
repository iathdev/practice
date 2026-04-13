# Offline-First Exam — Bảo mật Nội dung Đề thi & Đáp án

> Nghiên cứu toàn diện các kỹ thuật bảo mật cho hai bài toán riêng biệt: (1) bảo vệ nội dung câu hỏi khỏi bị đọc trước hoặc chia sẻ, và (2) bảo vệ đáp án/bài làm khỏi bị giả mạo hoặc đọc trộm. Bao gồm các kỹ thuật phổ biến lẫn ít được biết đến, từ DRM, ZKP, đến hardware security.

---

## TL;DR — Bảng tóm tắt nhanh

### Bảo mật Nội dung Đề thi

| Kỹ thuật | Ý tưởng | Thực tế dùng |
|---|---|---|
| AES-GCM + Time-locked key | Pre-fetch encrypted content, server chỉ trả key khi đến đúng giờ | Netflix/Widevine DRM |
| Envelope Encryption | Phân cấp key: Master → Exam → Section → Session. Một section bị lộ key không ảnh hưởng section khác | AWS KMS pattern |
| EME/Widevine cho audio | Audio Listening section decrypt trong hardware enclave — JS không bao giờ thấy plaintext audio | Netflix, YouTube |
| CSP + SRI | Ngăn XSS đọc content, đảm bảo exam JS không bị tamper | Defense in depth |
| ~~Time-lock puzzle~~ | ~~Tính toán CPU-bound để unlock~~ — không ổn định thực tế | Bỏ qua |

### Bảo mật Đáp án / Bài làm

| Kỹ thuật | Ý tưởng | Thực tế dùng |
|---|---|---|
| AES-GCM + `extractable: false` | Key không export được kể cả qua DevTools | Web Crypto API |
| HMAC-SHA256 trên mỗi answer | Mỗi answer có tag = HMAC(answer + timestamp + seq + question_id) — tamper thì tag fail | AWS, Stripe |
| Merkle Tree audit chain | Mỗi answer hash chain từ answer trước — sửa 1 answer phá toàn bộ chain | Git, blockchain |
| Sequence number + monotonic counter | Server reject answer có seq ≤ max_received — chống replay/rollback attack | Standard |
| Server-side immutable event log | Ground truth là server log, local chỉ là cache — tamper local vô nghĩa | Most reliable |

---

## Mục lục

1. [Bối cảnh & Threat Model](#1-bối-cảnh--threat-model)
2. [Bảo mật Nội dung Đề thi (Question Content Security)](#2-bảo-mật-nội-dung-đề-thi)
3. [Bảo mật Đáp án / Bài làm (Answer Security)](#3-bảo-mật-đáp-án--bài-làm)
4. [Kỹ thuật Nền tảng Dùng Chung](#4-kỹ-thuật-nền-tảng-dùng-chung)
5. [Kiến trúc Kết hợp Thực tế](#5-kiến-trúc-kết-hợp-thực-tế)
6. [So sánh & Lựa chọn](#6-so-sánh--lựa-chọn)
7. [Interview Quick Reference](#7-interview-quick-reference)

---

## 1. Bối cảnh & Threat Model

### Điều kiện hệ thống

| Thuộc tính | Giá trị |
|---|---|
| Platform | Browser-based (web app, PWA) |
| Storage | IndexedDB (offline-first) |
| Format | IELTS/TOEIC style — text + audio + image |
| User | Thí sinh đã xác thực, có session hợp lệ |

### Các mối đe dọa cần chống

**Đối với nội dung đề thi:**
- Thí sinh đọc câu hỏi trước giờ thi (pre-exam leak)
- Trích xuất câu hỏi từ IndexedDB sau khi tải về
- Chia sẻ câu hỏi cho người khác (cross-session sharing)
- Screen capture / copy-paste nội dung
- Man-in-the-middle khi tải đề từ server

**Đối với đáp án / bài làm:**
- Sửa đáp án đã nộp trong IndexedDB (local tampering)
- Inject câu trả lời từ bên ngoài (answer injection)
- Replay attack — gửi lại bài làm cũ
- Đọc trộm đáp án từ storage (privacy breach)
- Sửa timestamp để giả mạo thời gian nộp

**Giới hạn không thể chống hoàn toàn trong browser:**
- Screen capture bằng thiết bị khác (phone chụp màn hình)
- Memory dump của tab đang chạy (cần OS-level protection)
- Browser DevTools (cần lockdown browser)

---

## 2. Bảo mật Nội dung Đề thi

### 2.1 AES-GCM Encryption + Time-Locked Key Release (★★★ Khuyến nghị)

**Cách hoạt động:**

```
[Server trước giờ thi]
  → Mã hóa toàn bộ đề thi bằng AES-256-GCM với Content Encryption Key (CEK)
  → Lưu ciphertext vào CDN / server (có thể public)
  → CEK được giữ bí mật trên Key Management Server (KMS)

[Client khi pre-fetch]
  → Tải encrypted_exam_package xuống, lưu vào IndexedDB
  → KHÔNG có key → không đọc được

[Đúng giờ thi]
  → Client gửi: { exam_id, session_token, timestamp }
  → KMS xác minh: session hợp lệ + current_time >= exam_start_time
  → KMS trả về: CEK (chỉ có hiệu lực trong session hiện tại)
  → Client decrypt trong memory, KHÔNG lưu CEK vào IndexedDB
  → Render câu hỏi, xóa CEK khỏi memory sau khi parse xong
```

**Sức mạnh:**
- Nội dung trong IndexedDB luôn là ciphertext — forensic tools cũng không đọc được
- Key không bao giờ rời KMS dưới dạng plaintext lâu dài
- Offline content pre-fetch mà vẫn secure

**Điểm yếu:**
- Cần online để lấy key — cần thiết kế retry/grace period khi mất mạng thoáng qua
- CEK tồn tại trong JS memory trong thời gian render → có thể bị memory dump
- Không chống được screen capture

**Thực tế sử dụng:** Mô hình này tương tự **Netflix/Widevine DRM** — nội dung encrypted được CDN distribute, key chỉ được license server phát theo điều kiện.

---

### 2.2 Envelope Encryption + Key Hierarchy (★★★)

**Cách hoạt động:**

```
Tầng 1 — Master Key (MK): Lưu trong HSM/KMS, không bao giờ rời server
Tầng 2 — Exam Key (EK): Duy nhất cho mỗi kỳ thi, được mã hóa bởi MK
Tầng 3 — Content Encryption Key (CEK): Mỗi câu hỏi một key riêng, encrypted bởi EK
Tầng 4 — Session Wrap Key (SWK): CEK được wrap lại cho từng session cụ thể
```

Mô hình này đến từ **AWS KMS Envelope Encryption**:
- KMS chỉ encrypt/decrypt key, không chạm vào bulk data
- Câu hỏi 1 bị leak → chỉ lộ CEK của câu đó, không lộ toàn bộ đề
- Granular revocation: có thể thu hồi key từng phần

**Sức mạnh:**
- Cô lập damage radius khi một key bị compromise
- Phù hợp cho đề thi lớn với nhiều phần thi độc lập (Listening/Reading/Writing)

**Điểm yếu:**
- Phức tạp hơn trong implementation
- Cần quản lý nhiều key hơn

---

### 2.3 Shamir's Secret Sharing — Phân tán Key (★★ Ít phổ biến)

**Cách hoạt động:**

Thay vì một server giữ toàn bộ CEK, key được chia thành n shares, cần k/n shares để reconstruct (k-of-n threshold scheme):

```
CEK → split thành 5 shares (k=3 để reconstruct)
Share 1 → Server A (Main KMS)
Share 2 → Server B (Exam Platform)
Share 3 → Client (được gửi kèm exam package)
Share 4 → Proctor server (nếu có giám thị online)
Share 5 → Backup escrow
```

Khi thí sinh bắt đầu thi: client gửi Share 3 + xin Share 1 và Share 2 từ hai server. Chỉ khi đủ 3 shares mới reconstruct được CEK.

**Sức mạnh:**
- Không có single point of failure — không server nào đơn độc có thể giải mã
- Bảo vệ khỏi insider attack (nhân viên server không thể tự lấy đề)
- Được dùng trong: **HashiCorp Vault** (unseal keys), **cold wallet crypto**

**Điểm yếu:**
- Cần phối hợp nhiều server khi thi → latency tăng
- Nếu thí sinh offline hoàn toàn → không lấy được share từ server
- Complex implementation; dễ sai

---

### 2.4 Time-Lock Puzzle — Không cần Server (★ Học thuật)

**Cách hoạt động:**

Dựa trên nghiên cứu của **Rivest, Shamir, Wagner (1996)**:

```
1. Server tạo RSA modulus n = p*q
2. CEK được ẩn trong time-lock puzzle: puzzle = CEK XOR (a^(2^T) mod n)
3. Client nhận puzzle + ciphertext
4. Để giải: client phải thực hiện T lần bình phương liên tiếp (sequential squaring)
5. Không thể song song hóa → T iterations ≈ thời gian thực thi nhất định
```

**Sức mạnh:**
- Hoàn toàn offline — không cần server để unlock
- Toán học đảm bảo: không thể rút ngắn nếu không có factorization của n

**Điểm yếu:**
- Thời gian giải phụ thuộc tốc độ CPU của máy khách — máy mạnh hơn giải nhanh hơn
- Không practical cho production: nếu thí sinh có máy ASIC/GPU mạnh → phá vỡ constraint
- Chỉ phù hợp cho môi trường controlled (máy thi cố định, hardware đồng nhất)

**Thực tế sử dụng:** Chủ yếu trong học thuật. Ethereum 2.0 dùng biến thể VDF (Verifiable Delay Function) cho randomness beacon.

---

### 2.5 Encrypted Media Extensions (EME) / Browser DRM (★★ Cho media content)

**Cách hoạt động:**

EME là W3C spec cho phép browser giao tiếp với Content Decryption Module (CDM) — một blackbox được hardware bảo vệ:

```
Widevine DRM workflow:
1. Server gửi encrypted audio/video (CENC format)
2. Browser phát hiện encrypted content → trigger EME
3. Browser tạo license request → gửi đến License Server
4. License Server xác minh session → trả về license (CEK encrypted cho CDM)
5. CDM decrypt trong hardware-isolated environment
6. Decrypted frames KHÔNG đi qua JS, KHÔNG accessible từ JavaScript
```

**Sức mạnh:**
- Decryption xảy ra trong CDM (hardware-backed) → JS không thể đọc plaintext
- Netflix, YouTube Premium, Disney+ đều dùng mô hình này
- Widevine L1 (highest security): key xử lý trong TEE (Trusted Execution Environment)

**Điểm yếu:**
- Chỉ hoạt động cho audio/video content — không dùng được cho text/JSON câu hỏi
- Cần CDM → chỉ có trên browser desktop (Chrome, Firefox, Edge); mobile browser có thể thiếu
- Widevine là proprietary của Google — không open standard
- Overkill cho text-based exam

**Áp dụng thực tế:** Dùng EME cho phần nghe (Listening section) của IELTS/TOEIC — audio file mã hóa bằng Widevine, key chỉ release đúng giờ thi.

---

### 2.6 Segment-Level Encryption + Progressive Unlock (★★)

**Cách hoạt động:**

Thay vì unlock toàn bộ đề cùng lúc, server release key từng phần:

```
Đề thi 40 câu → chia 4 sections (10 câu/section)
Section key K1, K2, K3, K4 — mỗi key release sau khi thí sinh hoàn thành section trước
Hoặc: key K_t chỉ valid trong time window [t_start, t_end]
```

**Sức mạnh:**
- Giới hạn exposure: nếu key bị leak, chỉ lộ một phần đề
- Tương thích với thi theo phần (phần Listening xong → phần Reading)

**Điểm yếu:**
- Cần online để lấy key từng section → conflict với offline-first requirement
- Giải pháp: pre-fetch tất cả section keys nhưng encrypted bằng session key, giải mã dần

---

### 2.7 Content Security Policy + Subresource Integrity (★★ Defense-in-depth)

**Cách hoạt động:**

```html
<!-- Ngăn exfiltration qua XSS -->
Content-Security-Policy: default-src 'self'; connect-src https://api.examplatform.com; img-src 'self' data:; script-src 'self' 'sha256-[hash]'

<!-- Đảm bảo script không bị inject/thay thế -->
<script src="/exam-engine.js" integrity="sha384-[hash]" crossorigin="anonymous"></script>
```

**Sức mạnh:**
- Ngăn XSS attack inject script để exfiltrate câu hỏi
- SRI đảm bảo exam engine script không bị tamper
- Defense-in-depth: không phải primary security, nhưng ngăn vector tấn công phổ biến

**Điểm yếu:**
- Không bảo vệ được nếu attacker có physical access vào máy
- CSP có thể bị misconfigure → tạo false sense of security

---

## 3. Bảo mật Đáp án / Bài làm

### 3.1 HMAC-based Tamper Detection (★★★ Khuyến nghị)

**Cách hoạt động:**

Mỗi answer record được ký bằng HMAC trước khi lưu vào IndexedDB:

```javascript
// Khi lưu đáp án
const answerData = {
  question_id: "q_001",
  answer: "B",
  answered_at: "2026-04-11T09:15:30.000Z",
  sequence: 42
};

// Signing key = HMAC-SHA256(session_key, device_id + exam_id)
const hmac = await crypto.subtle.sign("HMAC", signingKey, encode(JSON.stringify(answerData)));

// Lưu vào IndexedDB
idb.put("answers", { ...answerData, integrity: toBase64(hmac) });

// Khi đọc lại / submit
const valid = await crypto.subtle.verify("HMAC", signingKey, hmac, encode(JSON.stringify(answerData)));
if (!valid) throw new IntegrityViolationError();
```

**Sức mạnh:**
- Bất kỳ thay đổi nào vào answer/timestamp/sequence đều vô hiệu hóa HMAC
- Sử dụng Web Crypto API — native browser, không cần library ngoài
- Nhẹ (microseconds per operation)

**Điểm yếu:**
- Signing key vẫn trong JS memory → attacker có thể lấy key và forge HMAC mới
- HMAC là symmetric: ai có key đều có thể tạo HMAC hợp lệ (khác digital signature)
- Giải pháp: signing key = derived từ server-issued session token, không hardcode

**Thực tế sử dụng:** AWS API Gateway, Stripe webhooks, và nhiều banking API dùng HMAC-SHA256 để verify message integrity.

---

### 3.2 Asymmetric Digital Signature (ECDSA) cho Non-repudiation (★★★)

**Cách hoạt động:**

```
Server tạo per-session key pair: (private_key_session, public_key_session)
private_key_session → gửi cho client trong secure channel (HTTPS, encrypted by session token)
public_key_session → lưu trên server

Client ký mỗi answer:
  signature = ECDSA.sign(private_key_session, hash(answer + timestamp + sequence))
  Lưu vào IndexedDB: { answer, timestamp, sequence, signature }

Khi server nhận bài:
  Server verify: ECDSA.verify(public_key_session, signature, hash(answer + timestamp + sequence))
  Nếu sai → reject, đánh dấu tamper attempt
```

**Sức mạnh:**
- Non-repudiation: thí sinh không thể phủ nhận đã nộp đáp án đó (khác HMAC)
- Asymmetric: server có thể verify mà không cần biết private key
- ECDSA (P-256) được Web Crypto API hỗ trợ native

**Điểm yếu:**
- Private key vẫn trong JS memory → advanced attacker có thể extract
- Key rotation phức tạp hơn HMAC
- Nếu private key bị steal → attacker có thể forge signature

**Thực tế sử dụng:** JWT ES256 signatures, document signing systems, code signing certificates.

---

### 3.3 AES-GCM Encryption của Answer Records (★★★)

**Cách hoạt động:**

Đáp án được mã hóa trước khi lưu vào IndexedDB:

```
Answer Encryption Key (AEK) = HKDF(session_key, "answer-encryption", exam_id)
Mỗi answer record: ciphertext = AES-256-GCM(AEK, plaintext_answer, iv=random, aad=question_id)
GCM authentication tag tích hợp → tự động detect tampering

Lưu vào IndexedDB: { question_id, ciphertext, iv, auth_tag }
```

AES-GCM cung cấp **authenticated encryption**: vừa confidentiality vừa integrity trong một bước.

**Sức mạnh:**
- Đáp án trong IndexedDB không đọc được mà không có AEK
- GCM auth tag: bất kỳ bit nào bị thay đổi → decrypt fails
- Additional Authenticated Data (AAD) ràng buộc ciphertext với câu hỏi cụ thể

**Điểm yếu:**
- AEK trong memory → cùng vấn đề với mọi client-side key
- Nếu attacker dump memory để lấy AEK → decrypt toàn bộ IndexedDB được

**Thực tế sử dụng:** Signal Protocol, WhatsApp end-to-end encryption, iCloud Keychain.

---

### 3.4 Merkle Tree Audit Trail (★★ Ít phổ biến nhưng mạnh)

**Cách hoạt động:**

```
Mỗi answer submission tạo một leaf node trong Merkle tree:
leaf_n = SHA256(answer_n + timestamp_n + sequence_n + leaf_{n-1}_hash)

Sau mỗi 10 answers (hoặc mỗi phút), gửi Merkle root lên server:
server lưu: { exam_session_id, merkle_root, timestamp, sequence_count }

Khi submit cuối:
Server rebuild Merkle tree từ submitted answers
So sánh với các intermediate roots đã lưu
Bất kỳ thay đổi nào ở answer cũ → root hiện tại khác root đã checkpoint
```

**Sức mạnh:**
- Immutable audit trail: không thể quay lại sửa đáp án cũ mà không bị phát hiện
- Efficient proof: có thể prove một answer cụ thể mà không cần gửi toàn bộ bài
- Tương tự cơ chế **Git commit history** hoặc **blockchain block hashing**

**Điểm yếu:**
- Cần online để sync Merkle root lên server → không hoàn toàn offline
- Giải pháp: buffer roots locally, sync khi có mạng
- Implementation phức tạp hơn HMAC đơn giản

**Thực tế sử dụng:** Bitcoin/Ethereum transaction verification, Certificate Transparency logs, Git object storage.

---

### 3.5 Sequence Number + Monotonic Counter (★★ Chống Replay Attack)

**Cách hoạt động:**

```
Mỗi answer event có sequence number tăng đơn điệu:
{ answer: "B", seq: 47, prev_seq: 46, session_id: "sess_xyz" }

Server maintain: max_received_seq per session
Khi nhận answer với seq <= max_received_seq → reject (replay/rollback attempt)

Kết hợp với HMAC: HMAC bao gồm seq → không thể thay seq mà không phá HMAC
```

**Sức mạnh:**
- Ngăn rollback attack: thí sinh không thể "undo" đáp án đã submit
- Ngăn replay attack: submit cùng packet cũ nhiều lần không được chấp nhận
- Đơn giản, overhead thấp

**Điểm yếu:**
- Nếu sequence không được ký (HMAC) → attacker có thể thay seq
- Cần server lưu trạng thái per-session → server stateful

---

### 3.6 Zero-Knowledge Proof (ZKP) cho Answer Verification (★ Nâng cao / Tương lai)

**Cách hoạt động:**

Ý tưởng: thí sinh prove rằng họ "có đáp án hợp lệ" mà không reveal đáp án cho server trong lúc thi:

```
Tình huống: thi viết luận, muốn prove bài đã hoàn thành (word count >= 150) mà không gửi nội dung
ZKP: client tạo proof π = ZKProve("essay has >= 150 words", private_witness=essay_content)
Server verify: ZKVerify(π, public_params) → true/false (không biết nội dung essay)
Sau khi thi kết thúc → gửi plaintext essay để chấm
```

Hoặc với multiple-choice:
```
Chứng minh: "tôi đã trả lời câu 1-40, không có câu nào skip" mà không lộ đáp án
→ Ngăn hệ thống chấm bị gian lận nhưng vẫn đảm bảo privacy trong lúc thi
```

**Sức mạnh:**
- Privacy-preserving: server không biết đáp án trong lúc thi, chỉ biết khi kết thúc
- Cryptographically sound: không thể fake proof
- zk-SNARK cho phép verify trong milliseconds dù proof generation tốn vài giây

**Điểm yếu:**
- Cực kỳ phức tạp để implement đúng
- zk-SNARK generation tốn CPU đáng kể trên browser
- Overkill cho hầu hết exam platform thực tế
- Library: snarkjs (JavaScript), Circom (circuit language)

**Thực tế sử dụng:** Zcash (private cryptocurrency transactions), ZKBAR-V (blockchain-based academic credential verification), Cloudflare Privacy Pass.

---

### 3.7 Server-Side Receive Log + Conflict Detection (★★★ Thực tế)

**Cách hoạt động:**

```
Client gửi answer event stream: { event_id, question_id, answer, timestamp, device_id }
Server ghi nhận MỌNG event → immutable append-only log
Nếu client gửi cùng question_id với answer khác nhau → conflict flag

Khi submit cuối:
  Server reconstruct "final answers" từ event log theo rule: last-write-wins hoặc server-timestamp-wins
  Client không thể "giả mạo" event đã ghi vì server log là authoritative
```

**Sức mạnh:**
- Ground truth là server log, không phải IndexedDB của client
- Offline period: client buffer events, sync khi có mạng; server merge với ordering
- Audit trail đầy đủ: biết thí sinh thay đổi đáp án câu nào, bao nhiêu lần, lúc nào

**Điểm yếu:**
- Cần reliable sync — nếu network mất hoàn toàn, cần reconciliation khi nộp
- Event log có thể lớn (đặc biệt với phần thi Writing/typing)

---

## 4. Kỹ thuật Nền tảng Dùng Chung

### 4.1 Web Crypto API — SubtleCrypto (Nền tảng)

Browser native API cho tất cả cryptographic operations:

```javascript
// Key derivation từ session token
const masterKey = await crypto.subtle.importKey("raw", sessionTokenBytes, "HKDF", false, ["deriveKey"]);

const cek = await crypto.subtle.deriveKey(
  { name: "HKDF", hash: "SHA-256", salt: examIdBytes, info: encode("content-encryption") },
  masterKey,
  { name: "AES-GCM", length: 256 },
  false, // extractable = false → key không export ra JS được
  ["encrypt", "decrypt"]
);
```

**Key insight:** Đặt `extractable: false` — key không thể được export ra bytes bởi JavaScript, ngay cả trong DevTools. Đây là tính năng quan trọng nhất của Web Crypto API cho security.

---

### 4.2 IndexedDB Attack Vectors — Cần Biết

Nghiên cứu cho thấy IndexedDB có các vulnerability sau:

| Attack | Cơ chế | Mitigation |
|---|---|---|
| Direct file read | SQLite files lưu trên disk không encrypted (Firefox, Chrome) | Encrypt tất cả values trước khi lưu |
| XSS exfiltration | Script inject → đọc IDB trong cùng origin | CSP strict, SRI, input sanitization |
| Forensic tool extraction | IDB file trên disk đọc offline | App-bound encryption (Chrome 127+) hoặc OS-level FDE |
| Private mode bypass | Nghiên cứu 2024: key trong memory có thể extract | Không lưu key vào persistent storage |
| DevTools | `indexedDB.open()` từ console | Không giúp được nếu data đã encrypted |

**Chrome App-Bound Encryption (2024):** Chrome mã hóa IDB data bằng key ràng buộc với Chrome app identity — tool bên ngoài không đọc được, nhưng Chrome extension và JS trong tab vẫn đọc được.

---

### 4.3 WebAuthn / FIDO2 — Device-Bound Session (★★)

**Cách hoạt động:**

```
Registration (trước khi thi):
  Thí sinh đăng ký thiết bị → browser tạo key pair trong TPM/Secure Enclave
  Private key KHÔNG BAO GIỜ rời hardware
  Public key lưu trên server

Khi bắt đầu thi:
  Server gửi challenge
  Browser ký challenge bằng device-bound private key (trong hardware)
  Server verify → xác nhận đúng thiết bị

Session key = derived từ WebAuthn assertion → ràng buộc với hardware device
```

**Sức mạnh:**
- Private key trong TPM/Secure Enclave → không thể extract dù có quyền admin
- Exam session bound to specific hardware → không thể chuyển sang máy khác
- Phishing-resistant: không thể fake domain

**Điểm yếu:**
- Cần enrollment trước → UX friction
- Nếu máy hỏng → thí sinh không thi được
- Không phải mọi device đều có TPM (cần Windows 11, hoặc recent iOS/Android)

**Thực tế sử dụng:** Google, Microsoft dùng hardware keys cho internal auth. Pearson VUE, Prometric đang thử nghiệm device-bound credentials.

---

### 4.4 Lockdown Browser / Kiosk Mode (★★★ Phổ biến nhất trong thực tế)

**Kỹ thuật Safe Exam Browser (SEB):**

```
SEB mở một Windows desktop riêng biệt
Block: Alt+F4, Ctrl+Alt+Del, Print Screen, Task Manager
Monitor: processes đang chạy → kill nếu không trong whitelist
Block: virtual machine detection (quét 400+ ứng dụng)
Block: screen sharing tools, remote access software
Hash-based configuration: config file được hash → server verify đúng SEB config
```

**Browser Fingerprinting của SEB:**
- SEB gửi custom HTTP header `X-SafeExamBrowser-RequestHash`
- Hash = SHA256(URL + SEB_config_key) → server verify client đang chạy đúng SEB config

**Giới hạn:** Thí sinh có thể dùng điện thoại chụp màn hình → cần giám thị hoặc AI proctoring.

**Thực tế sử dụng:** Moodle, Canvas, Blackboard tích hợp SEB. ETH Zurich, nhiều đại học châu Âu dùng.

---

### 4.5 Certificate Pinning / API Binding (★★)

**Cách hoạt động:**

Service Worker intercept mọi request → verify server certificate fingerprint:

```javascript
// Service Worker
self.addEventListener('fetch', async (event) => {
  const response = await fetch(event.request);
  // Với HTTP Public Key Pinning (deprecated) → dùng custom check
  // Verify response headers contain expected server signature
  const serverSig = response.headers.get('X-Server-Signature');
  if (!verifyServerSignature(serverSig, PINNED_PUBLIC_KEY)) {
    throw new SecurityError('Certificate mismatch');
  }
  return response;
});
```

**Lưu ý:** HTTP Public Key Pinning (HPKP) đã bị Chrome deprecate năm 2018 vì risk of bricking sites. Thay thế bằng **Certificate Transparency** logs.

**Thực tế áp dụng:** Mobile banking apps dùng certificate pinning trong native SDK. Cho web: chủ yếu dùng custom header signing thay vì cert pinning.

---

## 5. Kiến trúc Kết hợp Thực tế

### Recommended Stack cho IELTS/TOEIC Offline-First

```
┌─────────────────────────────────────────────────────────────────┐
│                    EXAM CONTENT SECURITY                        │
│                                                                 │
│  CDN/Server          Client (Browser)         KMS              │
│  ─────────           ──────────────           ─────            │
│  Encrypted exam   →  IndexedDB (ciphertext)   Holds CEK        │
│  (AES-256-GCM)       ↑                        ↑               │
│                      Pre-fetched early         |               │
│                                                |               │
│  T=exam_start: Client ──────────────────────→ KMS             │
│                      ← CEK (session-bound) ──                  │
│                      Decrypt in memory                         │
│                      Render questions                          │
│                      CEK cleared after parse                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                    ANSWER SECURITY                              │
│                                                                 │
│  Layer 1: AES-GCM encryption of each answer record             │
│           AEK = HKDF(session_token, "answer-enc", exam_id)     │
│           extractable: false → key không export được           │
│                                                                 │
│  Layer 2: HMAC-SHA256 integrity tag per answer                  │
│           Includes: answer + timestamp + sequence + question_id │
│                                                                 │
│  Layer 3: Sequence number + Merkle chain                        │
│           Sync Merkle root lên server mỗi 30s (online)         │
│           Buffer locally khi offline                            │
│                                                                 │
│  Layer 4: Server-side immutable event log (ground truth)       │
│           Client gửi answer events → server append-only log    │
│           Submit = server reconstruct từ log                    │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                 DEFENSE IN DEPTH                                │
│                                                                 │
│  - CSP strict: ngăn XSS exfiltration                           │
│  - SRI: đảm bảo exam engine script không bị tamper             │
│  - WebAuthn (optional): device binding                          │
│  - Lockdown browser (optional): kiosk mode                     │
│  - AI proctoring (optional): behavioral monitoring             │
└─────────────────────────────────────────────────────────────────┘
```

---

## 6. So sánh & Lựa chọn

### Exam Content Security

| Kỹ thuật | Offline-compatible | Implementation | Security Level | Thực tế sử dụng |
|---|---|---|---|---|
| AES-GCM + Time-locked key | Partial (cần online lấy key) | Trung bình | Cao | Netflix DRM model |
| Envelope Encryption | Partial | Cao | Rất cao | AWS KMS |
| Shamir Secret Sharing | Không (cần multi-server) | Rất cao | Rất cao | HashiCorp Vault |
| Time-Lock Puzzle | Hoàn toàn offline | Cao | Trung bình | Học thuật |
| EME/Widevine DRM | Partial | Thấp (API) | Rất cao | Netflix, YouTube |
| Segment-level unlock | Partial | Trung bình | Trung bình | Custom |

### Answer Security

| Kỹ thuật | Chống Tamper | Non-repudiation | Offline OK | Complexity |
|---|---|---|---|---|
| AES-GCM encryption | Cao (auth tag) | Không | Hoàn toàn | Thấp |
| HMAC integrity tag | Cao | Không | Hoàn toàn | Thấp |
| ECDSA signature | Cao | Có | Hoàn toàn | Trung bình |
| Merkle chain | Rất cao | Partial | Partial (sync roots) | Cao |
| Sequence counter | Trung bình | Không | Hoàn toàn | Thấp |
| ZKP | Rất cao | Có | Hoàn toàn | Rất cao |
| Server event log | Rất cao (ground truth) | Có | Partial (need sync) | Trung bình |

### Khuyến nghị cho production

**Content:** AES-GCM + Time-locked key từ KMS (đơn giản, proven, offline-compatible sau khi pre-fetch)

**Answers:** AES-GCM (confidentiality) + HMAC (integrity) + Sequence counter (anti-replay) + Server event log (authoritative ground truth)

**Defense-in-depth thêm:** CSP strict + SRI + WebAuthn (cho high-stakes exam)

---

## 7. Interview Quick Reference

**Q: Làm sao bảo vệ đề thi offline mà thí sinh không đọc được trước giờ?**

> Pre-fetch encrypted content (AES-256-GCM) vào IndexedDB. CEK giữ trên KMS. Đúng giờ thi, client authenticated request lấy CEK — verify session + timestamp. CEK dùng để decrypt trong memory, không persist. Tương tự Netflix DRM: CDN distribute ciphertext, License Server control key access.

**Q: Làm sao chống thí sinh sửa đáp án trong IndexedDB?**

> Ba lớp: (1) AES-GCM encrypt mỗi answer record — auth tag detect bất kỳ bit change. (2) HMAC-SHA256 với signing key derived từ session token — bao gồm answer + timestamp + sequence. (3) Server-side immutable event log là ground truth — client IndexedDB chỉ là cache, không authoritative.

**Q: Vì sao không dùng client-side key hoàn toàn?**

> Client-side key có thể bị extract từ JS memory hoặc memory dump. `extractable: false` trong Web Crypto giúp một phần, nhưng không chống được OS-level memory read. Giải pháp: key derivation từ server-issued token (short-lived), kết hợp device binding qua WebAuthn cho high-stakes scenarios.

**Q: Khác biệt giữa HMAC và ECDSA digital signature?**

> HMAC dùng symmetric key — ai có key đều ký được, dùng cho integrity check nội bộ. ECDSA dùng asymmetric key — private key ký, public key verify — đảm bảo non-repudiation. Cho exam: HMAC đủ nếu client-server trust, ECDSA cần thiết nếu muốn prove "thí sinh X đã nộp đáp án Y" trước tòa án.

**Q: Time-Lock Puzzle có thực tế không?**

> Không cho production web exam. CPU-dependent timing không reliable. Thực tế dùng server-controlled key release (time-locked key từ KMS) thay vì computational puzzle. Time-lock puzzle chỉ hữu ích khi không có server (e.g., distributed protocols).

---

*Research date: 2026-04-11*
