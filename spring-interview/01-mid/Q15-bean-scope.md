# Q15: How do you define a bean scope?
> **Dịch:** Làm thế nào để định nghĩa Bean Scope?

## Tra loi ngan gon
> Bean scope dinh nghia **vong doi** va **so luong instance** cua bean. Khai bao bang `@Scope` annotation hoac trong XML config.

## Cach nho
```
Singleton  = 1 chiec xe bus (tat ca di chung 1 chiec)
Prototype  = Taxi (moi nguoi 1 chiec rieng)
Request    = Ly nuoc (moi khach 1 ly moi)
Session    = Ban an (1 nhom khach dung chung 1 ban)
```

## Cac loai scope

| Scope | So instance | Khi nao tao | Dung khi |
|-------|-------------|-------------|----------|
| **singleton** | 1 duy nhat | Khi container khoi dong | Stateless bean (mac dinh) |
| **prototype** | Moi lan 1 cai moi | Moi khi getBean() | Stateful bean |
| **request** | 1 / HTTP request | Moi request | Web - data theo request |
| **session** | 1 / HTTP session | Moi session | Web - data theo user session |
| **application** | 1 / ServletContext | Khi app khoi dong | Web - data chia se |
| **websocket** | 1 / WebSocket | Moi WS connection | WebSocket |

## Cach khai bao

### Annotation:
```java
// Singleton (mac dinh - khong can khai bao)
@Service
public class UserService { }

// Prototype
@Component
@Scope("prototype")
public class ShoppingCart { }

// Dung constant (an toan hon)
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

## Vi du minh hoa

```java
// SINGLETON - cung 1 instance
@Service // singleton mac dinh
public class UserService { }

UserService a = ctx.getBean(UserService.class);
UserService b = ctx.getBean(UserService.class);
System.out.println(a == b); // TRUE - cung 1 object

// PROTOTYPE - moi lan 1 instance moi
@Component
@Scope("prototype")
public class ShoppingCart { }

ShoppingCart a = ctx.getBean(ShoppingCart.class);
ShoppingCart b = ctx.getBean(ShoppingCart.class);
System.out.println(a == b); // FALSE - khac object
```

## Luu y: Singleton inject Prototype

```java
// VAN DE: Prototype bean trong Singleton se KHONG tao moi!
@Service // Singleton
public class OrderService {
    @Autowired
    private ShoppingCart cart; // Prototype - nhung chi inject 1 lan!
}

// GIAI PHAP: Dung @Lookup hoac ObjectFactory
@Service
public class OrderService {
    @Lookup
    public ShoppingCart getCart() { return null; } // Spring override method nay
}

// Hoac dung ObjectFactory / Provider
@Service
public class OrderService {
    @Autowired
    private ObjectFactory<ShoppingCart> cartFactory;

    public void process() {
        ShoppingCart cart = cartFactory.getObject(); // Moi lan 1 cart moi
    }
}
```

## Diem quan trong nho phong van
1. Mac dinh la **singleton** (1 instance duy nhat)
2. **request, session** chi dung trong **web application**
3. Prototype bean trong Singleton -> dung **@Lookup** hoac **ObjectFactory**
4. Singleton bean **khong nen** chua state (vi chia se giua cac thread)
