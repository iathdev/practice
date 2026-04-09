# Q27: What is Controller in Spring MVC framework?
> **Dịch:** Controller trong Spring MVC Framework là gì?

## Tra loi ngan gon
> **Controller** la component xu ly HTTP request tu client, goi business logic (Service), va tra ve response (View hoac JSON). Trong Spring MVC, controller duoc danh dau bang `@Controller` hoac `@RestController`.

## Cach nho
```
Controller = Boi ban nha hang
1. Nhan order (HTTP Request)
2. Chuyen cho bep (Service layer)
3. Mang mon ra (Response)
```

## Vi du day du

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    // GET /api/users
    @GetMapping
    public List<User> list() {
        return userService.findAll();
    }

    // GET /api/users/1
    @GetMapping("/{id}")
    public ResponseEntity<User> getById(@PathVariable Long id) {
        return userService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }

    // POST /api/users
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public User create(@Valid @RequestBody UserDto dto) {
        return userService.create(dto);
    }

    // PUT /api/users/1
    @PutMapping("/{id}")
    public User update(@PathVariable Long id, @Valid @RequestBody UserDto dto) {
        return userService.update(id, dto);
    }

    // DELETE /api/users/1
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

## Controller CHI nen lam gi?
```
CO nen:                           KHONG nen:
- Nhan request                    - Viet business logic
- Validate input (@Valid)         - Truy cap database truc tiep
- Goi Service                     - Xu ly phuc tap
- Tra ve response                 - Giu state
```

## Diem quan trong nho phong van
1. Controller la **entry point** cua request
2. Nen **mong** (thin controller) - logic o **Service layer**
3. `@Controller` -> View, `@RestController` -> JSON
4. Dung `ResponseEntity` khi can control **status code va headers**
