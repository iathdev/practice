# Q5: How is the right View chosen when it comes to the rendering phase?
> **Dịch:** View được chọn như thế nào trong giai đoạn rendering?

## Tra loi ngan gon
> Spring MVC su dung **ViewResolver** de chuyen doi ten view (String) thanh doi tuong View cu the. ViewResolver se tim view phu hop dua tren ten logic va cau hinh (prefix/suffix).

## Cach nho
```
Controller tra ve: "home"
ViewResolver dich: "home" --> /WEB-INF/views/home.jsp

Giong nhu: Ban noi "Cafe" --> GPS tim ra "Cafe ABC, 123 Nguyen Hue"
```

## Qua trinh chon View

```
Client Request
     |
     v
DispatcherServlet
     |
     v
Controller --> tra ve "userList" (ten logic)
     |
     v
ViewResolver --> tim view phu hop
     |
     v
View (JSP/Thymeleaf/...) --> render HTML
     |
     v
Client Response (HTML)
```

## Cac loai ViewResolver pho bien

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

// Controller tra ve "home" --> /WEB-INF/views/home.jsp
// Controller tra ve "user/list" --> /WEB-INF/views/user/list.jsp
```

### 2. ThymeleafViewResolver (Thymeleaf)
```java
// Spring Boot tu cau hinh
// Controller tra ve "home" --> templates/home.html

@Controller
public class HomeController {
    @GetMapping("/")
    public String home(Model model) {
        model.addAttribute("message", "Hello World");
        return "home"; // ViewResolver se tim templates/home.html
    }
}
```

### 3. Nhieu ViewResolver (chain)
```java
@Bean
public ViewResolver jspViewResolver() {
    InternalResourceViewResolver resolver = new InternalResourceViewResolver();
    resolver.setPrefix("/WEB-INF/views/");
    resolver.setSuffix(".jsp");
    resolver.setOrder(2); // Thu tu uu tien (so nho = uu tien cao)
    return resolver;
}

@Bean
public ViewResolver thymeleafViewResolver() {
    ThymeleafViewResolver resolver = new ThymeleafViewResolver();
    resolver.setOrder(1); // Uu tien cao hon
    return resolver;
}
// Spring se thu Thymeleaf truoc, neu khong tim thay -> thu JSP
```

## Cac kieu tra ve cua Controller

```java
@Controller
public class MyController {

    // 1. Tra ve ten View (dung ViewResolver)
    @GetMapping("/page1")
    public String page1() {
        return "page1"; // ViewResolver se dich thanh file cu the
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

## Diem quan trong nho phong van
1. **ViewResolver** chuyen ten logic -> View cu the
2. Co the **chain** nhieu ViewResolver voi thu tu uu tien (`setOrder`)
3. `@RestController` / `@ResponseBody` **KHONG** dung ViewResolver (tra JSON truc tiep)
4. Spring Boot tu cau hinh ThymeleafViewResolver neu co dependency
