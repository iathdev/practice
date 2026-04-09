# Q19: Can we send an Object as the response of Controller handler method?
> **Dịch:** Có thể trả về Object làm response từ method của Controller không?

## Tra loi ngan gon
> **Co!** Dung `@ResponseBody` hoac `@RestController`, Spring se tu dong chuyen doi (serialize) Object thanh **JSON/XML** qua **HttpMessageConverter** (mac dinh la Jackson).

## Cach nho
```
@Controller    = Tra ve ten View (HTML)
@RestController = Tra ve Object -> tu dong chuyen JSON
```

## Vi du

### Tra ve Object (JSON)
```java
@RestController // = @Controller + @ResponseBody
@RequestMapping("/api")
public class UserController {

    // Tra ve 1 object -> JSON
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return new User(1L, "Thai", "thai@email.com");
    }
    // Response: {"id": 1, "name": "Thai", "email": "thai@email.com"}

    // Tra ve List -> JSON array
    @GetMapping("/users")
    public List<User> getUsers() {
        return List.of(
            new User(1L, "Thai", "thai@email.com"),
            new User(2L, "Dong", "dong@email.com")
        );
    }
    // Response: [{"id": 1, ...}, {"id": 2, ...}]

    // Tra ve ResponseEntity (kem HTTP status, headers)
    @GetMapping("/users/{id}/detail")
    public ResponseEntity<User> getUserDetail(@PathVariable Long id) {
        User user = userService.findById(id);
        if (user == null) {
            return ResponseEntity.notFound().build(); // 404
        }
        return ResponseEntity.ok(user); // 200 + JSON
    }
}
```

### Qua trinh chuyen doi
```
Object (Java) --> HttpMessageConverter --> JSON (Response)
                  (Jackson mac dinh)

User{id=1, name="Thai"} --> {"id": 1, "name": "Thai"}
```

### Customize JSON output
```java
public class User {
    private Long id;

    @JsonProperty("full_name")  // Doi ten field trong JSON
    private String name;

    @JsonIgnore                 // An field, khong tra ve
    private String password;

    @JsonFormat(pattern = "dd/MM/yyyy")  // Format ngay
    private LocalDate birthday;
}
// Output: {"id": 1, "full_name": "Thai", "birthday": "09/04/2026"}
```

## Diem quan trong nho phong van
1. `@RestController` tu dong serialize Object -> JSON
2. **Jackson** la thu vien mac dinh de chuyen doi JSON
3. Dung **ResponseEntity** khi can control HTTP status va headers
4. Customize voi `@JsonProperty`, `@JsonIgnore`, `@JsonFormat`
