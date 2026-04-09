# Q25: What is the purpose of the session scope?
> **Dịch:** Mục đích của Session Scope là gì?

## Tra loi ngan gon
> **Session scope** tao **1 instance bean rieng** cho moi HTTP session cua user. Bean ton tai suot session va bi huy khi session het han. Dung de luu thong tin **rieng cua tung user** nhu gio hang, preferences.

## Cach nho
```
Singleton = TV o sanh (tat ca khach xem chung)
Session   = Remote TV trong phong (moi khach 1 cai rieng)
Request   = Khan mat (dung 1 lan roi bo)
```

## Vi du thuc te: Shopping Cart

```java
@Component
@SessionScope  // Moi user co 1 ShoppingCart rieng
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
    private ShoppingCart cart; // Moi user nhan cart RIENG cua minh

    @PostMapping("/cart/add")
    public ShoppingCart addToCart(@RequestBody Item item) {
        cart.addItem(item);  // User A them -> chi co trong cart User A
        return cart;
    }

    @GetMapping("/cart")
    public ShoppingCart viewCart() {
        return cart; // Tra ve cart cua user HIEN TAI
    }
}
```

## Cac web scope

| Scope | Vong doi | Use case |
|-------|---------|----------|
| **request** | 1 HTTP request | Request-specific data, form data |
| **session** | 1 HTTP session | Shopping cart, user preferences |
| **application** | 1 ServletContext | Shared config, app-wide counter |

## Luu y: Inject session-scoped bean vao singleton

```java
// Spring tu dong tao PROXY de xu ly
@Service // Singleton
public class OrderService {
    @Autowired
    private ShoppingCart cart; // Session scope -> Spring inject PROXY
    // Proxy se delegate den dung cart cua session hien tai
}

// Hoac khai bao ro proxyMode
@Component
@Scope(value = "session", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class ShoppingCart { }
```

## Diem quan trong nho phong van
1. Session scope = **1 bean / 1 user session**
2. Dung cho: **gio hang, user preferences, wizard multi-step form**
3. Inject vao singleton -> Spring dung **scoped proxy** tu dong
4. Chi hoat dong trong **web application**
