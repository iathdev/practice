# Q8: Why are controllers testable artifacts?
> **Dịch:** Tại sao Controller là thành phần dễ kiểm thử (testable)?

## Tra loi ngan gon
> Controller trong Spring MVC la **POJO** (Plain Old Java Object), khong ke thua tu Servlet. Nho **Dependency Injection**, cac dependency co the duoc mock, giup test de dang ma **khong can deploy** len server.

## Cach nho
```
Struts Controller = Can ca bep (server) moi nau duoc
Spring Controller = Noi com dien (test duoc o bat ky dau)
```

## Tai sao Spring Controller de test?

### 1. Controller la POJO thuong
```java
@RestController
public class UserController {
    private final UserService userService;

    // Dependency Injection - co the mock duoc
    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }
}
```

### 2. Test voi MockMvc (khong can server)
```java
@WebMvcTest(UserController.class)  // Chi load MVC layer
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;       // Mo phong HTTP request

    @MockBean
    private UserService userService; // Mock dependency

    @Test
    void shouldReturnUser() throws Exception {
        // Given
        User user = new User(1L, "Thai");
        when(userService.findById(1L)).thenReturn(user);

        // When & Then
        mockMvc.perform(get("/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.name").value("Thai"));
    }
}
```

### 3. Unit test thuan tuy (khong can Spring)
```java
class UserControllerUnitTest {

    private UserService userService = mock(UserService.class);
    private UserController controller = new UserController(userService);

    @Test
    void shouldReturnUser() {
        // Given
        when(userService.findById(1L)).thenReturn(new User(1L, "Thai"));

        // When
        User result = controller.getUser(1L);

        // Then
        assertEquals("Thai", result.getName());
    }
}
```

## So sanh voi Servlet truyen thong
```
Servlet: phai extend HttpServlet, can ServletContext, khong mock duoc
Spring:  POJO, inject dependency, mock de dang
```

## Diem quan trong nho phong van
1. Controller la **POJO** - khong phu thuoc vao Servlet API
2. **DI** cho phep **mock** cac dependency khi test
3. **MockMvc** test HTTP request ma **khong can deploy** server
4. `@WebMvcTest` chi load MVC layer -> test **nhanh**
