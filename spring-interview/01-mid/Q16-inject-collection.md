# Q16: How can you inject Java Collection in Spring?
> **Dịch:** Làm thế nào để inject Java Collection trong Spring?

## Trả lời ngắn gọn
> Spring hỗ trợ inject **List, Set, Map, Properties** qua XML config hoặc annotation. Với annotation, Spring tự động inject **tất cả bean cùng type** vào List/Set, hoặc dùng `@Value` để inject từ properties file.

## Cách nhớ
```
Inject Collection = Bỏ tất cả đạo cụ cùng loại vào 1 hộp
List = Hộp có thứ tự
Set  = Hộp không trùng lặp
Map  = Hộp có nhãn tên
```

## Cách 1: Inject tất cả bean cùng type vào List

```java
// Nhiều bean implement cùng 1 interface
@Component
public class EmailNotifier implements Notifier {
    public void send(String msg) { /* gửi email */ }
}

@Component
public class SmsNotifier implements Notifier {
    public void send(String msg) { /* gửi SMS */ }
}

@Component
public class SlackNotifier implements Notifier {
    public void send(String msg) { /* gửi Slack */ }
}

// Inject TẤT CẢ vào List
@Service
public class NotificationService {
    @Autowired
    private List<Notifier> notifiers; // [EmailNotifier, SmsNotifier, SlackNotifier]

    public void notifyAll(String msg) {
        notifiers.forEach(n -> n.send(msg)); // Gửi hết!
    }
}
```

## Cách 2: Inject với @Value từ properties

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
    // Inject List từ comma-separated string
    @Value("${app.allowed-origins}")
    private List<String> allowedOrigins;

    // Inject List từ YAML list
    @Value("${app.admin-emails}")
    private List<String> adminEmails;
}
```

## Cách 3: Inject Map

```java
// Map<String, Bean> - key là bean name, value là bean instance
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

## Cách 4: Sắp xếp thứ tự với @Order

```java
@Component
@Order(1)  // Thứ tự 1 (đầu tiên)
public class SecurityFilter implements Filter { }

@Component
@Order(2)  // Thứ tự 2
public class LoggingFilter implements Filter { }

@Service
public class FilterChain {
    @Autowired
    private List<Filter> filters; // [SecurityFilter, LoggingFilter] - theo @Order
}
```

## Cách 5: XML config (cũ nhưng vẫn gặp)

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

## Điểm quan trọng nhớ phỏng vấn
1. Inject `List<SomeInterface>` -> Spring inject **tất cả** bean implement interface đó
2. Inject `Map<String, SomeInterface>` -> key = **bean name**, value = bean instance
3. Dùng **@Order** hoặc **@Priority** để sắp xếp thứ tự trong List
4. `@Value` có thể inject List từ **properties file**
