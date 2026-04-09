# Q48: What is the default scope in the web context?
> **Dịch:** Scope mặc định trong Web Context là gì?

# Q49: What are the limitations with autowiring?
> **Dịch:** Những hạn chế của Autowiring là gì?

# Q50: What are different Modes of auto wiring?
> **Dịch:** Các chế độ (mode) Auto Wiring khác nhau là gì?

# Q51: What bean scopes does Spring support?
> **Dịch:** Spring hỗ trợ những Bean Scope nào?

---

## Q48: Default Scope in Web Context

### Trả lời ngắn gọn
> Mặc định vẫn là **singleton** - cả trong web context lẫn non-web context. Nhưng web context có thêm các scope: **request, session, application, websocket**.

---

## Q49: Limitations of Autowiring

### Trả lời ngắn gọn
> Autowiring có nhiều hạn chế cần lưu ý.

```
1. AMBIGUITY (Nhập nhằng)
   - Nhiều bean cùng type -> NoUniqueBeanDefinitionException
   - Giải pháp: @Qualifier, @Primary

2. KHÔNG OVERRIDE được
   - Autowiring không cho phép set giá trị primitive (String, int)
   - Phải dùng @Value

3. KHÓ DEBUG
   - Dependency không rõ ràng (đặc biệt field injection)
   - Khó biết bean nào được inject

4. KHÔNG DÙNG CHO SIMPLE TYPES
   - Không autowire được String, int, boolean
   - Chỉ autowire được bean

5. DOCUMENTATION
   - Không có tài liệu nào chỉ ra dependency (với field injection)
   - Constructor injection rõ ràng hơn

Ví dụ vấn đề:
@Service
public class PaymentService {
    @Autowired
    private PaymentGateway gateway; // Gateway nào? Stripe? PayPal? Không biết!
}

// Fix:
@Service
public class PaymentService {
    @Autowired
    @Qualifier("stripeGateway") // Rõ ràng!
    private PaymentGateway gateway;
}
```

---

## Q50: Autowiring Modes

### Trả lời ngắn gọn (XML modes)

| Mode | Mô tả |
|------|-------|
| **no** | Không autowire (mặc định XML) |
| **byName** | Tìm bean có ID trùng với tên property |
| **byType** | Tìm bean có TYPE trùng với kiểu property |
| **constructor** | Như byType nhưng qua constructor |
| **autodetect** | Thử constructor, rồi byType (bỏ từ Spring 4) |

```
Trong annotation-based (thực tế):
@Autowired mặc định là byType
Nếu nhiều bean cùng type -> dùng @Qualifier (byName)
```

---

## Q51: Bean Scopes

### Bảng tổng hợp tất cả scope

| Scope | Mô tả | Ví dụ |
|-------|-------|-------|
| **singleton** | 1 instance / container (MẶC ĐỊNH) | Service, Repository |
| **prototype** | Instance mới mỗi lần getBean() | Stateful objects |
| **request** | 1 instance / HTTP request | Request data |
| **session** | 1 instance / HTTP session | Shopping cart |
| **application** | 1 instance / ServletContext | Shared config |
| **websocket** | 1 instance / WebSocket session | WS connection data |

```java
@Component
@Scope("singleton")     // Mặc định, không cần ghi
public class SingletonBean { }

@Component
@Scope("prototype")
public class PrototypeBean { }

@Component
@RequestScope           // = @Scope("request")
public class RequestBean { }

@Component
@SessionScope           // = @Scope("session")
public class SessionBean { }
```

## Điểm quan trọng nhớ phỏng vấn
1. Default scope = **singleton** (kể cả web context)
2. Autowiring hạn chế: **ambiguity, không primitive, khó debug**
3. Annotation-based autowiring mặc định **byType**
4. Web scopes (request/session) cần **web application context**
