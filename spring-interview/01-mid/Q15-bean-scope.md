# Q15: How do you define a bean scope?
> **Dịch:** Làm thế nào để định nghĩa Bean Scope?

## Trả lời ngắn gọn
> Bean scope định nghĩa **vòng đời** và **số lượng instance** của bean. Khai báo bằng `@Scope` annotation hoặc trong XML config.

## Cách nhớ
```
Singleton  = 1 chiếc xe bus (tất cả đi chung 1 chiếc)
Prototype  = Taxi (mỗi người 1 chiếc riêng)
Request    = Ly nước (mỗi khách 1 ly mới)
Session    = Bàn ăn (1 nhóm khách dùng chung 1 bàn)
```

## Các loại scope

| Scope | Số instance | Khi nào tạo | Dùng khi |
|-------|-------------|-------------|----------|
| **singleton** | 1 duy nhất | Khi container khởi động | Stateless bean (mặc định) |
| **prototype** | Mỗi lần 1 cái mới | Mỗi khi getBean() | Stateful bean |
| **request** | 1 / HTTP request | Mỗi request | Web - data theo request |
| **session** | 1 / HTTP session | Mỗi session | Web - data theo user session |
| **application** | 1 / ServletContext | Khi app khởi động | Web - data chia sẻ |
| **websocket** | 1 / WebSocket | Mỗi WS connection | WebSocket |

## Cách khai báo

### Annotation:
```java
// Singleton (mặc định - không cần khai báo)
@Service
public class UserService { }

// Prototype
@Component
@Scope("prototype")
public class ShoppingCart { }

// Dùng constant (an toàn hơn)
@Component
@Scope(value = ConfigurableBeanFactory.SCOPE_PROTOTYPE)
public class ShoppingCart { }

// Request scope
@Component
@RequestScope  // Shortcut cho @Scope("request")
public class RequestData { }

// Session scope
@Component
@SessionScope  // Shortcut cho @Scope("session")
public class UserSession { }
```

### Java Config:
```java
@Configuration
public class AppConfig {

    @Bean
    @Scope("prototype")
    public ShoppingCart shoppingCart() {
        return new ShoppingCart();
    }
}
```

## Ví dụ minh họa

```java
// SINGLETON - cùng 1 instance
@Service // singleton mặc định
public class UserService { }

UserService a = ctx.getBean(UserService.class);
UserService b = ctx.getBean(UserService.class);
System.out.println(a == b); // TRUE - cùng 1 object

// PROTOTYPE - mỗi lần 1 instance mới
@Component
@Scope("prototype")
public class ShoppingCart { }

ShoppingCart a = ctx.getBean(ShoppingCart.class);
ShoppingCart b = ctx.getBean(ShoppingCart.class);
System.out.println(a == b); // FALSE - khác object
```

## Lưu ý: Singleton inject Prototype

```java
// VẤN ĐỀ: Prototype bean trong Singleton sẽ KHÔNG tạo mới!
@Service // Singleton
public class OrderService {
    @Autowired
    private ShoppingCart cart; // Prototype - nhưng chỉ inject 1 lần!
}

// GIẢI PHÁP: Dùng @Lookup hoặc ObjectFactory
@Service
public class OrderService {
    @Lookup
    public ShoppingCart getCart() { return null; } // Spring override method này
}

// Hoặc dùng ObjectFactory / Provider
@Service
public class OrderService {
    @Autowired
    private ObjectFactory<ShoppingCart> cartFactory;

    public void process() {
        ShoppingCart cart = cartFactory.getObject(); // Mỗi lần 1 cart mới
    }
}
```

## Điểm quan trọng nhớ phỏng vấn
1. Mặc định là **singleton** (1 instance duy nhất)
2. **request, session** chỉ dùng trong **web application**
3. Prototype bean trong Singleton -> dùng **@Lookup** hoặc **ObjectFactory**
4. Singleton bean **không nên** chứa state (vì chia sẻ giữa các thread)
