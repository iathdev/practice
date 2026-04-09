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

### Tra loi ngan gon
> Mac dinh van la **singleton** - ca trong web context lan non-web context. Nhung web context co them cac scope: **request, session, application, websocket**.

---

## Q49: Limitations of Autowiring

### Tra loi ngan gon
> Autowiring co nhieu han che can luu y.

```
1. AMBIGUITY (Nhap nhang)
   - Nhieu bean cung type -> NoUniqueBeanDefinitionException
   - Giai phap: @Qualifier, @Primary

2. KHONG OVERRIDE duoc
   - Autowiring khong cho phep set gia tri primitive (String, int)
   - Phai dung @Value

3. KHO DEBUG
   - Dependency khong ro rang (dac biet field injection)
   - Kho biet bean nao duoc inject

4. KHONG DUNG CHO SIMPLE TYPES
   - Khong autowire duoc String, int, boolean
   - Chi autowire duoc bean

5. DOCUMENTATION
   - Khong co tai lieu nao chi ra dependency (voi field injection)
   - Constructor injection ro rang hon

Vi du van de:
@Service
public class PaymentService {
    @Autowired
    private PaymentGateway gateway; // Gateway nao? Stripe? PayPal? Khong biet!
}

// Fix:
@Service
public class PaymentService {
    @Autowired
    @Qualifier("stripeGateway") // Ro rang!
    private PaymentGateway gateway;
}
```

---

## Q50: Autowiring Modes

### Tra loi ngan gon (XML modes)

| Mode | Mo ta |
|------|-------|
| **no** | Khong autowire (mac dinh XML) |
| **byName** | Tim bean co ID trung voi ten property |
| **byType** | Tim bean co TYPE trung voi kieu property |
| **constructor** | Nhu byType nhung qua constructor |
| **autodetect** | Thu constructor, roi byType (bo tu Spring 4) |

```
Trong annotation-based (thuc te):
@Autowired mac dinh la byType
Neu nhieu bean cung type -> dung @Qualifier (byName)
```

---

## Q51: Bean Scopes

### Bang tong hop tat ca scope

| Scope | Mo ta | Vi du |
|-------|-------|-------|
| **singleton** | 1 instance / container (MAC DINH) | Service, Repository |
| **prototype** | Instance moi moi lan getBean() | Stateful objects |
| **request** | 1 instance / HTTP request | Request data |
| **session** | 1 instance / HTTP session | Shopping cart |
| **application** | 1 instance / ServletContext | Shared config |
| **websocket** | 1 instance / WebSocket session | WS connection data |

```java
@Component
@Scope("singleton")     // Mac dinh, khong can ghi
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

## Diem quan trong nho phong van
1. Default scope = **singleton** (ke ca web context)
2. Autowiring han che: **ambiguity, khong primitive, kho debug**
3. Annotation-based autowiring mac dinh **byType**
4. Web scopes (request/session) can **web application context**
