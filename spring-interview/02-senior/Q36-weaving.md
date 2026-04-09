# Q36: What is Weaving?
> **Dịch:** Weaving (dệt/đan xen) là gì?

## Tra loi ngan gon
> **Weaving** la qua trinh **ket noi** Aspect vao target object de tao proxy (advised object). Co 3 thoi diem weaving: **compile-time, load-time, runtime** (Spring dung runtime).

## Cach nho
```
Weaving = "May" Aspect vao code chinh
  Compile-time = May truoc khi mac (AspectJ compiler)
  Load-time    = May khi dang mac (ClassLoader)
  Runtime      = Khoac them ao ngoai (Spring AOP - PROXY)
```

## 3 loai Weaving

| Loai | Khi nao | Tool | Performance |
|------|---------|------|:-----------:|
| **Compile-time** | Khi compile | AspectJ compiler (ajc) | Nhanh nhat |
| **Load-time** | Khi load class | AspectJ LTW agent | Nhanh |
| **Runtime** | Khi chay | Spring AOP (JDK/CGLIB proxy) | Cham hon 1 chut |

## Spring AOP dung Runtime Weaving

```java
// Spring tao PROXY bao quanh bean
@Service
public class UserService {          // Bean goc
    @Transactional
    public void save(User user) { }
}

// Khi inject, Spring tra ve PROXY, khong phai bean goc:
// UserService$$EnhancerByCGLIB -> UserService
//
// Proxy.save() {
//     openTransaction();
//     target.save(user);  // goi bean goc
//     commitTransaction();
// }
```

### 2 loai Proxy trong Spring

```
JDK Dynamic Proxy:   Khi bean IMPLEMENT interface
CGLIB Proxy:         Khi bean KHONG co interface (tao subclass)

// Spring Boot mac dinh dung CGLIB (proxyTargetClass=true)
```

## Diem quan trong nho phong van
1. Weaving = ket noi Aspect vao target object
2. Spring AOP dung **runtime weaving** (proxy-based)
3. **JDK Proxy** cho interface, **CGLIB** cho class
4. Self-invocation khong hoat dong vi **khong qua proxy**
