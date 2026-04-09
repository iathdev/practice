# Q30: What are the differences between @RequestParam and @PathVariable?
> **Dịch:** Sự khác nhau giữa @RequestParam và @PathVariable là gì?

## Tra loi ngan gon
> `@PathVariable` lay gia tri tu **URL path** (`/users/1`). `@RequestParam` lay gia tri tu **query string** (`/users?name=Thai`).

## Cach nho
```
URL: /users/1?name=Thai&age=25
          |         |       |
     @PathVariable  @RequestParam  @RequestParam
```

## So sanh

| | @PathVariable | @RequestParam |
|--|:---:|:---:|
| Lay tu | URL path | Query string |
| Vi du URL | `/users/1` | `/users?name=Thai` |
| Bat buoc | Co (mac dinh) | Co the optional |
| Dung cho | Resource ID | Filter, search, pagination |

## Vi du chi tiet

```java
@RestController
@RequestMapping("/api")
public class UserController {

    // @PathVariable - lay tu URL path
    // GET /api/users/1
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }

    // Nhieu PathVariable
    // GET /api/users/1/orders/5
    @GetMapping("/users/{userId}/orders/{orderId}")
    public Order getOrder(@PathVariable Long userId,
                          @PathVariable Long orderId) {
        return orderService.findByUserAndId(userId, orderId);
    }

    // @RequestParam - lay tu query string
    // GET /api/users?name=Thai&page=0&size=10
    @GetMapping("/users")
    public Page<User> searchUsers(
            @RequestParam String name,                           // bat buoc
            @RequestParam(defaultValue = "0") int page,          // co default
            @RequestParam(defaultValue = "10") int size,         // co default
            @RequestParam(required = false) String email) {      // optional
        return userService.search(name, email, page, size);
    }

    // Ket hop ca 2
    // GET /api/users/1/orders?status=PAID&sort=date
    @GetMapping("/users/{userId}/orders")
    public List<Order> getUserOrders(
            @PathVariable Long userId,                  // Tu path
            @RequestParam(defaultValue = "ALL") String status,  // Tu query
            @RequestParam(defaultValue = "date") String sort) { // Tu query
        return orderService.findByUser(userId, status, sort);
    }

    // @RequestParam voi List
    // GET /api/users?ids=1,2,3  hoac  /api/users?ids=1&ids=2&ids=3
    @GetMapping("/users/batch")
    public List<User> getUsersByIds(@RequestParam List<Long> ids) {
        return userService.findByIds(ids);
    }
}
```

## Khi nao dung cai nao?

```
@PathVariable:
  - Xac dinh 1 RESOURCE cu the: /users/{id}, /products/{slug}
  - RESTful URL design
  - Luon BAT BUOC

@RequestParam:
  - FILTER / SEARCH: /users?name=Thai&role=ADMIN
  - PAGINATION: /users?page=0&size=10
  - SORT: /users?sort=name,asc
  - Co the OPTIONAL (required=false, defaultValue)
```

## Diem quan trong nho phong van
1. `@PathVariable` = tu **URL path**, `@RequestParam` = tu **query string**
2. `@RequestParam(required = false)` hoac `defaultValue` cho optional
3. Ket hop duoc ca 2 trong 1 method
4. RESTful: dung `@PathVariable` cho ID, `@RequestParam` cho filter/sort/page
