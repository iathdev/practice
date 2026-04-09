# Q13: How would you relate Spring MVC Framework to MVC architecture?
> **Dịch:** Spring MVC Framework liên hệ với kiến trúc MVC như thế nào?

## Trả lời ngắn gọn
> Spring MVC triển khai pattern MVC với: **Model** = data truyền qua Model/ModelAndView, **View** = template (Thymeleaf/JSP), **Controller** = class annotated với @Controller. **DispatcherServlet** là Front Controller điều phối tất cả.

## Cách nhớ
```
MVC = Nhà hàng
  Model      = Món ăn (data)
  View       = Bàn ăn, đĩa, chén (cách trình bày)
  Controller = Bồi bàn (nhận order, mang món ra)
  DispatcherServlet = Quản lý nhà hàng (phân công bồi bàn)
```

## Sơ đồ luồng xử lý

```
Client (Browser)
    |
    | 1. HTTP Request (GET /users)
    v
+-------------------+
| DispatcherServlet |  <-- Front Controller
+-------------------+
    |
    | 2. Tìm Controller phù hợp
    v
+-------------------+
| HandlerMapping    |  --> UserController.listUsers()
+-------------------+
    |
    | 3. Gọi Controller
    v
+-------------------+
| Controller        |  --> Xử lý logic, trả về Model + View name
| (@Controller)     |
+-------------------+
    |
    | 4. Trả về "userList" + data
    v
+-------------------+
| ViewResolver      |  --> Tìm file userList.html
+-------------------+
    |
    | 5. Render View với Model data
    v
+-------------------+
| View              |  --> Tạo HTML
| (Thymeleaf/JSP)   |
+-------------------+
    |
    | 6. HTTP Response (HTML)
    v
Client (Browser)
```

## Ví dụ code tương ứng

```java
// CONTROLLER - Nhận request, xử lý, trả về Model + View
@Controller
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping("/users")
    public String listUsers(Model model) {
        // MODEL - data truyền sang View
        List<User> users = userService.findAll();
        model.addAttribute("users", users);

        // Trả về tên VIEW
        return "userList"; // -> ViewResolver tìm userList.html
    }
}
```

```html
<!-- VIEW - Hiển thị data từ Model (Thymeleaf) -->
<!-- templates/userList.html -->
<html>
<body>
    <h1>User List</h1>
    <table>
        <tr th:each="user : ${users}">
            <td th:text="${user.name}">Name</td>
            <td th:text="${user.email}">Email</td>
        </tr>
    </table>
</body>
</html>
```

```java
// MODEL - Entity/DTO
public class User {
    private Long id;
    private String name;
    private String email;
    // getters, setters
}
```

## Mapping trong Spring MVC

| MVC Component | Spring Implementation |
|---------------|----------------------|
| Front Controller | DispatcherServlet |
| Controller | @Controller class |
| Model | Model, ModelAndView, @ModelAttribute |
| View | JSP, Thymeleaf, JSON (@ResponseBody) |
| Handler Mapping | @RequestMapping, @GetMapping... |
| View Resolver | InternalResourceViewResolver, ThymeleafViewResolver |

## Điểm quan trọng nhớ phỏng vấn
1. **DispatcherServlet** = Front Controller, điều phối tất cả request
2. **HandlerMapping** -> tìm Controller, **ViewResolver** -> tìm View
3. `@Controller` dùng View, `@RestController` trả JSON trực tiếp (không qua ViewResolver)
4. Spring MVC **tách biệt** 3 tầng rõ ràng: M - V - C
