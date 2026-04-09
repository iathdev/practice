# Q23: What do you mean by Auto Wiring?
> **Dịch:** Auto Wiring (tự động nối dây) nghĩa là gì?

## Trả lời ngắn gọn
> Autowiring là khả năng Spring **tự động tìm và inject** dependency vào bean mà developer không cần chỉ định cụ thể. Spring dựa vào **type**, **name**, hoặc **constructor** để tìm bean phù hợp.

(Xem chi tiết ở Q14)

## 5 chế độ Autowiring (XML)

| Mode | Mô tả | Ví dụ |
|------|-------|-------|
| **no** | Không autowire (mặc định XML) | Phải khai báo thủ công |
| **byName** | Tìm bean theo tên property | property `userRepo` -> bean id `userRepo` |
| **byType** | Tìm bean theo kiểu dữ liệu | property `UserRepository` -> bean kiểu `UserRepository` |
| **constructor** | Như byType nhưng qua constructor | |
| **autodetect** | Thử constructor trước, rồi byType | (bỏ từ Spring 4) |

## Trong thực tế (annotation)

```java
@Service
public class OrderService {
    // Spring tự động tìm bean kiểu OrderRepository
    // và inject vào đây (byType)
    private final OrderRepository orderRepo;

    public OrderService(OrderRepository orderRepo) {
        this.orderRepo = orderRepo;
    }
}
```

## Điểm quan trọng nhớ phỏng vấn
1. Autowiring = Spring **tự động** tìm và inject dependency
2. Annotation-based: mặc định theo **byType**
3. Nhiều bean cùng type -> `@Qualifier` hoặc `@Primary`
4. Constructor injection với 1 constructor -> `@Autowired` **tự động** (không cần ghi)
