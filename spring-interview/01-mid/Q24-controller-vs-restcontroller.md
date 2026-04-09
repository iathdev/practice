# Q24: Explain the difference between @Controller and @RestController
> **Dịch:** Giải thích sự khác nhau giữa @Controller và @RestController trong Spring

## Tra loi ngan gon
> `@Controller` tra ve **ten View** (HTML page). `@RestController` = `@Controller` + `@ResponseBody`, tra ve **data truc tiep** (JSON/XML).

## Cach nho
```
@Controller     = Nha sach (tra ve SACH = HTML page)
@RestController = May ATM (tra ve TIEN = JSON data)
```

## So sanh

### @Controller (tra ve View)
```java
@Controller
public class PageController {

    @GetMapping("/home")
    public String home(Model model) {
        model.addAttribute("name", "Thai");
        return "home"; // -> Tim file templates/home.html
    }

    // Muon tra JSON thi phai them @ResponseBody
    @GetMapping("/api/user")
    @ResponseBody  // Phai them tung method!
    public User getUser() {
        return new User("Thai");
    }
}
```

### @RestController (tra ve JSON)
```java
@RestController  // = @Controller + @ResponseBody cho TAT CA method
@RequestMapping("/api")
public class ApiController {

    @GetMapping("/users")
    public List<User> getUsers() {
        return userService.findAll(); // -> Tu dong tra JSON
    }

    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id); // -> Tu dong tra JSON
    }
}
```

## Bang so sanh

| | @Controller | @RestController |
|--|:-----------:|:---------------:|
| Tra ve | View name (String) | Object -> JSON |
| @ResponseBody | Phai them thu cong | Tu dong co san |
| Dung voi | Thymeleaf, JSP | REST API |
| ViewResolver | **Co** su dung | **KHONG** su dung |

## Khi nao dung cai nao?

```
Lam website (server-side rendering) -> @Controller + Thymeleaf
Lam REST API (cho frontend/mobile)  -> @RestController
Lam ca 2 trong 1 app                -> Tach rieng controller
```

## Diem quan trong nho phong van
1. `@RestController` = `@Controller` + `@ResponseBody`
2. `@Controller` can **ViewResolver**, `@RestController` thi **khong**
3. `@RestController` tu dong dung **Jackson** de serialize JSON
4. Co the ket hop: `@Controller` + `@ResponseBody` tren tung method
