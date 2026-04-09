# Q23: What do you mean by Auto Wiring?
> **Dịch:** Auto Wiring (tự động nối dây) nghĩa là gì?

## Tra loi ngan gon
> Autowiring la kha nang Spring **tu dong tim va inject** dependency vao bean ma developer khong can chi dinh cu the. Spring dua vao **type**, **name**, hoac **constructor** de tim bean phu hop.

(Xem chi tiet o Q14)

## 5 che do Autowiring (XML)

| Mode | Mo ta | Vi du |
|------|-------|-------|
| **no** | Khong autowire (mac dinh XML) | Phai khai bao thu cong |
| **byName** | Tim bean theo ten property | property `userRepo` -> bean id `userRepo` |
| **byType** | Tim bean theo kieu du lieu | property `UserRepository` -> bean kieu `UserRepository` |
| **constructor** | Nhu byType nhung qua constructor | |
| **autodetect** | Thu constructor truoc, roi byType | (bo tu Spring 4) |

## Trong thuc te (annotation)

```java
@Service
public class OrderService {
    // Spring tu dong tim bean kieu OrderRepository
    // va inject vao day (byType)
    private final OrderRepository orderRepo;

    public OrderService(OrderRepository orderRepo) {
        this.orderRepo = orderRepo;
    }
}
```

## Diem quan trong nho phong van
1. Autowiring = Spring **tu dong** tim va inject dependency
2. Annotation-based: mac dinh theo **byType**
3. Nhieu bean cung type -> `@Qualifier` hoac `@Primary`
4. Constructor injection voi 1 constructor -> `@Autowired` **tu dong** (khong can ghi)
