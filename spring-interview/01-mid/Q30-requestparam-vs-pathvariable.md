# Q30: What are the differences between @RequestParam and @PathVariable?
> **Dịch:** Sự khác nhau giữa @RequestParam và @PathVariable là gì?

## Trả lời ngắn gọn
> `@PathVariable` lấy giá trị từ **URL path** (`/users/1`). `@RequestParam` lấy giá trị từ **query string** (`/users?name=Thai`).

## Cách nhớ
```
URL: /users/1?name=Thai&age=25
          |         |       |
     @PathVariable  @RequestParam  @RequestParam
```

## So sánh

| | @PathVariable | @RequestParam |
|--|:---:|:---:|
| Lấy từ | URL path | Query string |
| Ví dụ URL | `/users/1` | `/users?name=Thai` |
| Bắt buộc | Có (mặc định) | Có thể optional |
| Dùng cho | Resource ID | Filter, search, pagination |

## Ví dụ chi tiết

```java
@RestController
@RequestMapping("/api")
public class UserController {

    // @PathVariable - lấy từ URL path
    // GET /api/users/1
    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id);
    }

    // Nhiều PathVariable
    // GET /api/users/1/orders/5
    @GetMapping("/users/{userId}/orders/{orderId}")
    public Order getOrder(@PathVariable Long userId,
                          @PathVariable Long orderId) {
        return orderService.findByUserAndId(userId, orderId);
    }

    // @RequestParam - lấy từ query string
    // GET /api/users?name=Thai&page=0&size=10
    @GetMapping("/users")
    public Page<User> searchUsers(
            @RequestParam String name,                           // bắt buộc
            @RequestParam(defaultValue = "0") int page,          // có default
            @RequestParam(defaultValue = "10") int size,         // có default
            @RequestParam(required = false) String email) {      // optional
        return userService.search(name, email, page, size);
    }

    // Kết hợp cả 2
    // GET /api/users/1/orders?status=PAID&sort=date
    @GetMapping("/users/{userId}/orders")
    public List<Order> getUserOrders(
            @PathVariable Long userId,                  // Từ path
            @RequestParam(defaultValue = "ALL") String status,  // Từ query
            @RequestParam(defaultValue = "date") String sort) { // Từ query
        return orderService.findByUser(userId, status, sort);
    }

    // @RequestParam với List
    // GET /api/users?ids=1,2,3  hoặc  /api/users?ids=1&ids=2&ids=3
    @GetMapping("/users/batch")
    public List<User> getUsersByIds(@RequestParam List<Long> ids) {
        return userService.findByIds(ids);
    }
}
```

## Khi nào dùng cái nào?

```
@PathVariable:
  - Xác định 1 RESOURCE cụ thể: /users/{id}, /products/{slug}
  - RESTful URL design
  - Luôn BẮT BUỘC

@RequestParam:
  - FILTER / SEARCH: /users?name=Thai&role=ADMIN
  - PAGINATION: /users?page=0&size=10
  - SORT: /users?sort=name,asc
  - Có thể OPTIONAL (required=false, defaultValue)
```

## Điểm quan trọng nhớ phỏng vấn
1. `@PathVariable` = từ **URL path**, `@RequestParam` = từ **query string**
2. `@RequestParam(required = false)` hoặc `defaultValue` cho optional
3. Kết hợp được cả 2 trong 1 method
4. RESTful: dùng `@PathVariable` cho ID, `@RequestParam` cho filter/sort/page
