# Q10: What are the types of the transaction management Spring supports?
> **Dịch:** Spring hỗ trợ những loại quản lý transaction nào?

## Tra loi ngan gon
> Spring ho tro **2 loai** quan ly transaction:
> 1. **Programmatic** - code thu cong trong method
> 2. **Declarative** - dung annotation `@Transactional` (KHUYEN DUNG)

## Cach nho
```
Programmatic  = Lai xe so san (tu sang so, dap con, nha con)
Declarative   = Lai xe so tu dong (chi can dap ga, xe tu lam)
```

## Loai 1: Programmatic Transaction (thu cong)

```java
@Service
public class OrderService {
    @Autowired
    private TransactionTemplate transactionTemplate;

    public void createOrder(Order order) {
        transactionTemplate.execute(status -> {
            try {
                orderRepo.save(order);
                paymentService.charge(order);
                inventoryService.reduce(order);
                return null; // commit
            } catch (Exception e) {
                status.setRollbackOnly(); // rollback
                throw e;
            }
        });
    }
}
```
- Uu: Kiem soat chi tiet
- Nhuoc: Code dai, kho bao tri

## Loai 2: Declarative Transaction (annotation - KHUYEN DUNG)

```java
@Service
public class OrderService {

    @Transactional  // Chi can 1 annotation!
    public void createOrder(Order order) {
        orderRepo.save(order);        // Thanh cong
        paymentService.charge(order); // Thanh cong
        inventoryService.reduce(order); // THAT BAI -> Rollback TAT CA!
    }
}
```
- Uu: Don gian, sach se, de bao tri
- Nhuoc: It kiem soat chi tiet hon

## Cac thuoc tinh cua @Transactional

```java
@Transactional(
    propagation = Propagation.REQUIRED,      // Mac dinh
    isolation = Isolation.READ_COMMITTED,    // Muc co lap
    timeout = 30,                            // Timeout (giay)
    readOnly = false,                        // Doc/ghi
    rollbackFor = Exception.class,           // Rollback khi nao
    noRollbackFor = MailException.class      // Khong rollback khi nao
)
public void myMethod() { }
```

## Propagation (Lan truyen transaction)

```
REQUIRED (mac dinh)
  A() goi B(): B dung chung transaction voi A
  |-------- Transaction A ---------|
       |--- B dung chung ---|

REQUIRES_NEW
  A() goi B(): B tao transaction MOI, tam dung A
  |-------- Transaction A (tam dung) ---------|
       |--- Transaction B (moi) ---|

NESTED
  A() goi B(): B tao savepoint trong A
  |-------- Transaction A ---------|
       |--- Savepoint B ---|
```

| Propagation | Mo ta |
|-------------|-------|
| REQUIRED | Dung transaction hien tai, tao moi neu chua co |
| REQUIRES_NEW | Luon tao transaction moi |
| SUPPORTS | Co transaction thi dung, khong co thi thoi |
| NOT_SUPPORTED | Khong dung transaction |
| MANDATORY | BAT BUOC phai co transaction san |
| NEVER | BAT BUOC khong co transaction |
| NESTED | Tao savepoint (nested) |

## Luu y quan trong

```java
// SAI - Tu goi noi bo, @Transactional KHONG hoat dong!
@Service
public class UserService {
    public void methodA() {
        this.methodB(); // Khong qua proxy -> @Transactional bi bo qua!
    }

    @Transactional
    public void methodB() { }
}

// DUNG - Goi tu bean khac
@Service
public class OrderService {
    @Autowired
    private UserService userService;

    public void process() {
        userService.methodB(); // Qua proxy -> @Transactional hoat dong!
    }
}
```

## Diem quan trong nho phong van
1. **Declarative** (@Transactional) la best practice
2. Mac dinh chi **rollback RuntimeException** (unchecked), muon rollback tat ca: `rollbackFor = Exception.class`
3. **Self-invocation** khong hoat dong vi bypass proxy
4. `@Transactional(readOnly = true)` cho cac method chi doc -> **tang performance**
