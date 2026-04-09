# Q24: Explain the difference between @Controller and @RestController
> **Dịch:** Giải thích sự khác nhau giữa @Controller và @RestController trong Spring

## Trả lời ngắn gọn
> `@Controller` trả về **tên View** (HTML page). `@RestController` = `@Controller` + `@ResponseBody`, trả về **data trực tiếp** (JSON/XML).

## Cách nhớ
```
@Controller     = Nhà sách (trả về SÁCH = HTML page)
@RestController = Máy ATM (trả về TIỀN = JSON data)
```

## So sánh

### @Controller (trả về View)
```java
@Controller
public class PageController {

    @GetMapping("/home")
    public String home(Model model) {
        model.addAttribute("name", "Thai");
        return "home"; // -> Tìm file templates/home.html
    }

    // Muốn trả JSON thì phải thêm @ResponseBody
    @GetMapping("/api/user")
    @ResponseBody  // Phải thêm từng method!
    public User getUser() {
        return new User("Thai");
    }
}
```

### @RestController (trả về JSON)
```java
@RestController  // = @Controller + @ResponseBody cho TẤT CẢ method
@RequestMapping("/api")
public class ApiController {

    @GetMapping("/users")
    public List<User> getUsers() {
        return userService.findAll(); // -> Tự động trả JSON
    }

    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id); // -> Tự động trả JSON
    }
}
```

## Bảng so sánh

| | @Controller | @RestController |
|--|:-----------:|:---------------:|
| Trả về | View name (String) | Object -> JSON |
| @ResponseBody | Phải thêm thủ công | Tự động có sẵn |
| Dùng với | Thymeleaf, JSP | REST API |
| ViewResolver | **Có** sử dụng | **KHÔNG** sử dụng |

## Khi nào dùng cái nào?

```
Làm website (server-side rendering) -> @Controller + Thymeleaf
Làm REST API (cho frontend/mobile)  -> @RestController
Làm cả 2 trong 1 app                -> Tách riêng controller
```

## Điểm quan trọng nhớ phỏng vấn
1. `@RestController` = `@Controller` + `@ResponseBody`
2. `@Controller` cần **ViewResolver**, `@RestController` thì **không**
3. `@RestController` tự động dùng **Jackson** để serialize JSON
4. Có thể kết hợp: `@Controller` + `@ResponseBody` trên từng method
