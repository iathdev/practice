# Q8: Why are controllers testable artifacts?
> **Dịch:** Tại sao Controller là thành phần dễ kiểm thử (testable)?

## Trả lời ngắn gọn
> Controller trong Spring MVC là **POJO** (Plain Old Java Object), không kế thừa từ Servlet. Nhờ **Dependency Injection**, các dependency có thể được mock, giúp test dễ dàng mà **không cần deploy** lên server.

## Cách nhớ
```
Struts Controller = Cần cả bếp (server) mới nấu được
Spring Controller = Nồi cơm điện (test được ở bất kỳ đâu)
```

## Tại sao Spring Controller dễ test?

### 1. Controller là POJO thường
```java
@RestController
public class UserController {
    private final UserService userService;

    // Dependency Injection - có thể mock được
    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }
}
```

### 2. Test với MockMvc (không cần server)
```java
@WebMvcTest(UserController.class)  // Chỉ load MVC layer
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;       // Mô phỏng HTTP request

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

### 3. Unit test thuần túy (không cần Spring)
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

## So sánh với Servlet truyền thống
```
Servlet: phải extend HttpServlet, cần ServletContext, không mock được
Spring:  POJO, inject dependency, mock dễ dàng
```

## Điểm quan trọng nhớ phỏng vấn
1. Controller là **POJO** - không phụ thuộc vào Servlet API
2. **DI** cho phép **mock** các dependency khi test
3. **MockMvc** test HTTP request mà **không cần deploy** server
4. `@WebMvcTest` chỉ load MVC layer -> test **nhanh**
