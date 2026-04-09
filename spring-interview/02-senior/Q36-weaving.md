# Q36: What is Weaving?
> **Dịch:** Weaving (dệt/đan xen) là gì?

## Trả lời ngắn gọn
> **Weaving** là quá trình **kết nối** Aspect vào target object để tạo proxy (advised object). Có 3 thời điểm weaving: **compile-time, load-time, runtime** (Spring dùng runtime).

## Cách nhớ
```
Weaving = "May" Aspect vào code chính
  Compile-time = May trước khi mặc (AspectJ compiler)
  Load-time    = May khi đang mặc (ClassLoader)
  Runtime      = Khoác thêm áo ngoài (Spring AOP - PROXY)
```

## 3 loại Weaving

| Loại | Khi nào | Tool | Performance |
|------|---------|------|:-----------:|
| **Compile-time** | Khi compile | AspectJ compiler (ajc) | Nhanh nhất |
| **Load-time** | Khi load class | AspectJ LTW agent | Nhanh |
| **Runtime** | Khi chạy | Spring AOP (JDK/CGLIB proxy) | Chậm hơn 1 chút |

## Spring AOP dùng Runtime Weaving

```java
// Spring tạo PROXY bao quanh bean
@Service
public class UserService {          // Bean gốc
    @Transactional
    public void save(User user) { }
}

// Khi inject, Spring trả về PROXY, không phải bean gốc:
// UserService$$EnhancerByCGLIB -> UserService
//
// Proxy.save() {
//     openTransaction();
//     target.save(user);  // gọi bean gốc
//     commitTransaction();
// }
```

### 2 loại Proxy trong Spring

```
JDK Dynamic Proxy:   Khi bean IMPLEMENT interface
CGLIB Proxy:         Khi bean KHÔNG có interface (tạo subclass)

// Spring Boot mặc định dùng CGLIB (proxyTargetClass=true)
```

## Điểm quan trọng nhớ phỏng vấn
1. Weaving = kết nối Aspect vào target object
2. Spring AOP dùng **runtime weaving** (proxy-based)
3. **JDK Proxy** cho interface, **CGLIB** cho class
4. Self-invocation không hoạt động vì **không qua proxy**
