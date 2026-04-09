# Q13: How would you relate Spring MVC Framework to MVC architecture?
> **Dịch:** Spring MVC Framework liên hệ với kiến trúc MVC như thế nào?

## Tra loi ngan gon
> Spring MVC trien khai pattern MVC voi: **Model** = data truyen qua Model/ModelAndView, **View** = template (Thymeleaf/JSP), **Controller** = class annotated voi @Controller. **DispatcherServlet** la Front Controller dieu phoi tat ca.

## Cach nho
```
MVC = Nha hang
  Model      = Mon an (data)
  View       = Ban an, dia, chen (cach trinh bay)
  Controller = Boi ban (nhan order, mang mon ra)
  DispatcherServlet = Quan ly nha hang (phan cong boi ban)
```

## So do luong xu ly

```
Client (Browser)
    |
    | 1. HTTP Request (GET /users)
    v
+-------------------+
| DispatcherServlet |  <-- Front Controller
+-------------------+
    |
    | 2. Tim Controller phu hop
    v
+-------------------+
| HandlerMapping    |  --> UserController.listUsers()
+-------------------+
    |
    | 3. Goi Controller
    v
+-------------------+
| Controller        |  --> Xu ly logic, tra ve Model + View name
| (@Controller)     |
+-------------------+
    |
    | 4. Tra ve "userList" + data
    v
+-------------------+
| ViewResolver      |  --> Tim file userList.html
+-------------------+
    |
    | 5. Render View voi Model data
    v
+-------------------+
| View              |  --> Tao HTML
| (Thymeleaf/JSP)   |
+-------------------+
    |
    | 6. HTTP Response (HTML)
    v
Client (Browser)
```

## Vi du code tuong ung

```java
// CONTROLLER - Nhan request, xu ly, tra ve Model + View
@Controller
public class UserController {

    @Autowired
    private UserService userService;

    @GetMapping("/users")
    public String listUsers(Model model) {
        // MODEL - data truyen sang View
        List<User> users = userService.findAll();
        model.addAttribute("users", users);

        // Tra ve ten VIEW
        return "userList"; // -> ViewResolver tim userList.html
    }
}
```

```html
<!-- VIEW - Hien thi data tu Model (Thymeleaf) -->
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

## Diem quan trong nho phong van
1. **DispatcherServlet** = Front Controller, dieu phoi tat ca request
2. **HandlerMapping** -> tim Controller, **ViewResolver** -> tim View
3. `@Controller` dung View, `@RestController` tra JSON truc tiep (khong qua ViewResolver)
4. Spring MVC **tach biet** 3 tang ro rang: M - V - C
