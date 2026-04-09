# Q9: How to handle exceptions in Spring MVC Framework?
> **Dịch:** Xử lý ngoại lệ (exception) trong Spring MVC như thế nào?

## Trả lời ngắn gọn
> Spring MVC hỗ trợ 3 cách xử lý exception:
> 1. **@ExceptionHandler** trong Controller (cục bộ)
> 2. **@ControllerAdvice** (toàn cục - KHUYÊN DÙNG)
> 3. **HandlerExceptionResolver** (customize)

## Cách nhớ
```
@ExceptionHandler   = Bảo vệ riêng (chỉ bảo vệ 1 phòng)
@ControllerAdvice   = Bảo vệ tòa nhà (bảo vệ tất cả phòng)
```

## Cách 1: @ExceptionHandler (trong 1 Controller)

```java
@RestController
public class UserController {

    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id)
            .orElseThrow(() -> new UserNotFoundException("User not found: " + id));
    }

    // Chỉ xử lý exception TRONG controller này thôi
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(UserNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(404, ex.getMessage());
        return ResponseEntity.status(404).body(error);
    }
}
```

## Cách 2: @ControllerAdvice (TOÀN CỤC - KHUYÊN DÙNG)

```java
// Xử lý exception cho TẤT CẢ controller
@RestControllerAdvice
public class GlobalExceptionHandler {

    // 404 - Not Found
    @ExceptionHandler(UserNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(UserNotFoundException ex) {
        return new ErrorResponse(404, ex.getMessage());
    }

    // 400 - Validation Error
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .toList();
        return new ErrorResponse(400, "Validation failed", errors);
    }

    // 500 - Catch-all
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleAll(Exception ex) {
        return new ErrorResponse(500, "Internal server error");
    }
}

// Error Response DTO
public record ErrorResponse(int status, String message, List<String> details) {
    public ErrorResponse(int status, String message) {
        this(status, message, null);
    }
}
```

## Cách 3: Custom Exception với @ResponseStatus

```java
@ResponseStatus(HttpStatus.NOT_FOUND)  // Tự động trả về 404
public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(String message) {
        super(message);
    }
}

// Khi throw exception này, Spring tự động trả về HTTP 404
```

## Ví dụ đầy đủ: Cấu trúc Exception Handling tốt

```java
// 1. Định nghĩa custom exceptions
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String resource, Long id) {
        super(resource + " not found with id: " + id);
    }
}

public class BusinessException extends RuntimeException {
    private final String errorCode;
    public BusinessException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }
}

// 2. Global handler
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(404)
            .body(new ErrorResponse(404, ex.getMessage()));
    }

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusiness(BusinessException ex) {
        return ResponseEntity.status(422)
            .body(new ErrorResponse(422, ex.getMessage()));
    }
}

// 3. Sử dụng trong Service
@Service
public class UserService {
    public User findById(Long id) {
        return userRepo.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User", id));
    }
}
```

## Điểm quan trọng nhớ phỏng vấn
1. **@ControllerAdvice** = xử lý exception **toàn cục** (best practice)
2. `@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`
3. Thứ tự ưu tiên: **exception cụ thể** trước, **Exception chung** sau
4. Nên tạo **custom exception** riêng cho từng loại lỗi
