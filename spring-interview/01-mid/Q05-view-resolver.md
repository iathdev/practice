# Q5: How is the right View chosen when it comes to the rendering phase?
> **Dịch:** View được chọn như thế nào trong giai đoạn rendering?

## Trả lời ngắn gọn
> Spring MVC sử dụng **ViewResolver** để chuyển đổi tên view (String) thành đối tượng View cụ thể. ViewResolver sẽ tìm view phù hợp dựa trên tên logic và cấu hình (prefix/suffix).

## Cách nhớ
```
Controller trả về: "home"
ViewResolver dịch: "home" --> /WEB-INF/views/home.jsp

Giống như: Bạn nói "Cafe" --> GPS tìm ra "Cafe ABC, 123 Nguyễn Huệ"
```

## Quá trình chọn View

```
Client Request
     |
     v
DispatcherServlet
     |
     v
Controller --> trả về "userList" (tên logic)
     |
     v
ViewResolver --> tìm view phù hợp
     |
     v
View (JSP/Thymeleaf/...) --> render HTML
     |
     v
Client Response (HTML)
```

## Các loại ViewResolver phổ biến

### 1. InternalResourceViewResolver (JSP)
```java
@Configuration
public class WebConfig {
    @Bean
    public ViewResolver viewResolver() {
        InternalResourceViewResolver resolver = new InternalResourceViewResolver();
        resolver.setPrefix("/WEB-INF/views/");
        resolver.setSuffix(".jsp");
        return resolver;
    }
}

// Controller trả về "home" --> /WEB-INF/views/home.jsp
// Controller trả về "user/list" --> /WEB-INF/views/user/list.jsp
```

### 2. ThymeleafViewResolver (Thymeleaf)
```java
// Spring Boot tự cấu hình
// Controller trả về "home" --> templates/home.html

@Controller
public class HomeController {
    @GetMapping("/")
    public String home(Model model) {
        model.addAttribute("message", "Hello World");
        return "home"; // ViewResolver sẽ tìm templates/home.html
    }
}
```

### 3. Nhiều ViewResolver (chain)
```java
@Bean
public ViewResolver jspViewResolver() {
    InternalResourceViewResolver resolver = new InternalResourceViewResolver();
    resolver.setPrefix("/WEB-INF/views/");
    resolver.setSuffix(".jsp");
    resolver.setOrder(2); // Thứ tự ưu tiên (số nhỏ = ưu tiên cao)
    return resolver;
}

@Bean
public ViewResolver thymeleafViewResolver() {
    ThymeleafViewResolver resolver = new ThymeleafViewResolver();
    resolver.setOrder(1); // Ưu tiên cao hơn
    return resolver;
}
// Spring sẽ thử Thymeleaf trước, nếu không tìm thấy -> thử JSP
```

## Các kiểu trả về của Controller

```java
@Controller
public class MyController {

    // 1. Trả về tên View (dùng ViewResolver)
    @GetMapping("/page1")
    public String page1() {
        return "page1"; // ViewResolver sẽ dịch thành file cụ thể
    }

    // 2. Redirect
    @GetMapping("/old-url")
    public String redirect() {
        return "redirect:/new-url"; // HTTP 302 redirect
    }

    // 3. Forward
    @GetMapping("/forward")
    public String forward() {
        return "forward:/other-page"; // Server-side forward
    }

    // 4. ModelAndView
    @GetMapping("/page2")
    public ModelAndView page2() {
        ModelAndView mav = new ModelAndView("page2");
        mav.addObject("name", "Thai");
        return mav;
    }
}
```

## Điểm quan trọng nhớ phỏng vấn
1. **ViewResolver** chuyển tên logic -> View cụ thể
2. Có thể **chain** nhiều ViewResolver với thứ tự ưu tiên (`setOrder`)
3. `@RestController` / `@ResponseBody` **KHÔNG** dùng ViewResolver (trả JSON trực tiếp)
4. Spring Boot tự cấu hình ThymeleafViewResolver nếu có dependency
