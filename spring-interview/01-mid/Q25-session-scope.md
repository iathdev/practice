# Q25: What is the purpose of the session scope?
> **Dịch:** Mục đích của Session Scope là gì?

## Trả lời ngắn gọn
> **Session scope** tạo **1 instance bean riêng** cho mỗi HTTP session của user. Bean tồn tại suốt session và bị hủy khi session hết hạn. Dùng để lưu thông tin **riêng của từng user** như giỏ hàng, preferences.

## Cách nhớ
```
Singleton = TV ở sảnh (tất cả khách xem chung)
Session   = Remote TV trong phòng (mỗi khách 1 cái riêng)
Request   = Khăn mặt (dùng 1 lần rồi bỏ)
```

## Ví dụ thực tế: Shopping Cart

```java
@Component
@SessionScope  // Mỗi user có 1 ShoppingCart riêng
public class ShoppingCart {
    private List<Item> items = new ArrayList<>();

    public void addItem(Item item) {
        items.add(item);
    }

    public List<Item> getItems() {
        return items;
    }

    public BigDecimal getTotal() {
        return items.stream()
            .map(Item::getPrice)
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}

// Controller
@RestController
public class CartController {
    @Autowired
    private ShoppingCart cart; // Mỗi user nhận cart RIÊNG của mình

    @PostMapping("/cart/add")
    public ShoppingCart addToCart(@RequestBody Item item) {
        cart.addItem(item);  // User A thêm -> chỉ có trong cart User A
        return cart;
    }

    @GetMapping("/cart")
    public ShoppingCart viewCart() {
        return cart; // Trả về cart của user HIỆN TẠI
    }
}
```

## Các web scope

| Scope | Vòng đời | Use case |
|-------|---------|----------|
| **request** | 1 HTTP request | Request-specific data, form data |
| **session** | 1 HTTP session | Shopping cart, user preferences |
| **application** | 1 ServletContext | Shared config, app-wide counter |

## Lưu ý: Inject session-scoped bean vào singleton

```java
// Spring tự động tạo PROXY để xử lý
@Service // Singleton
public class OrderService {
    @Autowired
    private ShoppingCart cart; // Session scope -> Spring inject PROXY
    // Proxy sẽ delegate đến đúng cart của session hiện tại
}

// Hoặc khai báo rõ proxyMode
@Component
@Scope(value = "session", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class ShoppingCart { }
```

## Điểm quan trọng nhớ phỏng vấn
1. Session scope = **1 bean / 1 user session**
2. Dùng cho: **giỏ hàng, user preferences, wizard multi-step form**
3. Inject vào singleton -> Spring dùng **scoped proxy** tự động
4. Chỉ hoạt động trong **web application**
