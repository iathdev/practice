# Q16: How can you inject Java Collection in Spring?
> **Dịch:** Làm thế nào để inject Java Collection trong Spring?

## Tra loi ngan gon
> Spring ho tro inject **List, Set, Map, Properties** qua XML config hoac annotation. Voi annotation, Spring tu dong inject **tat ca bean cung type** vao List/Set, hoac dung `@Value` de inject tu properties file.

## Cach nho
```
Inject Collection = Bo tat ca dao cu cung loai vao 1 hop
List = Hop co thu tu
Set  = Hop khong trung lap
Map  = Hop co nhan ten
```

## Cach 1: Inject tat ca bean cung type vao List

```java
// Nhieu bean implement cung 1 interface
@Component
public class EmailNotifier implements Notifier {
    public void send(String msg) { /* gui email */ }
}

@Component
public class SmsNotifier implements Notifier {
    public void send(String msg) { /* gui SMS */ }
}

@Component
public class SlackNotifier implements Notifier {
    public void send(String msg) { /* gui Slack */ }
}

// Inject TAT CA vao List
@Service
public class NotificationService {
    @Autowired
    private List<Notifier> notifiers; // [EmailNotifier, SmsNotifier, SlackNotifier]

    public void notifyAll(String msg) {
        notifiers.forEach(n -> n.send(msg)); // Gui het!
    }
}
```

## Cach 2: Inject voi @Value tu properties

```yaml
# application.yml
app:
  allowed-origins: http://localhost:3000,http://example.com
  admin-emails:
    - admin@example.com
    - super@example.com
```

```java
@Component
public class AppConfig {
    // Inject List tu comma-separated string
    @Value("${app.allowed-origins}")
    private List<String> allowedOrigins;

    // Inject List tu YAML list
    @Value("${app.admin-emails}")
    private List<String> adminEmails;
}
```

## Cach 3: Inject Map

```java
// Map<String, Bean> - key la bean name, value la bean instance
@Service
public class PaymentService {
    @Autowired
    private Map<String, PaymentProcessor> processors;
    // {"creditCard": CreditCardProcessor, "paypal": PaypalProcessor}

    public void pay(String method, BigDecimal amount) {
        PaymentProcessor processor = processors.get(method);
        processor.process(amount);
    }
}
```

## Cach 4: Sap xep thu tu voi @Order

```java
@Component
@Order(1)  // Thu tu 1 (dau tien)
public class SecurityFilter implements Filter { }

@Component
@Order(2)  // Thu tu 2
public class LoggingFilter implements Filter { }

@Service
public class FilterChain {
    @Autowired
    private List<Filter> filters; // [SecurityFilter, LoggingFilter] - theo @Order
}
```

## Cach 5: XML config (cu nhung van gap)

```xml
<bean id="myBean" class="com.example.MyBean">
    <property name="nameList">
        <list>
            <value>Alice</value>
            <value>Bob</value>
        </list>
    </property>
    <property name="configMap">
        <map>
            <entry key="timeout" value="30"/>
            <entry key="retries" value="3"/>
        </map>
    </property>
</bean>
```

## Diem quan trong nho phong van
1. Inject `List<SomeInterface>` -> Spring inject **tat ca** bean implement interface do
2. Inject `Map<String, SomeInterface>` -> key = **bean name**, value = bean instance
3. Dung **@Order** hoac **@Priority** de sap xep thu tu trong List
4. `@Value` co the inject List tu **properties file**
