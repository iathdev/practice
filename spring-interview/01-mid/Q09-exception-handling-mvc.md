# Q9: How to handle exceptions in Spring MVC Framework?
> **Dịch:** Xử lý ngoại lệ (exception) trong Spring MVC như thế nào?

## Tra loi ngan gon
> Spring MVC ho tro 3 cach xu ly exception:
> 1. **@ExceptionHandler** trong Controller (cuc bo)
> 2. **@ControllerAdvice** (toan cuc - KHUYEN DUNG)
> 3. **HandlerExceptionResolver** (customize)

## Cach nho
```
@ExceptionHandler   = Bao ve rieng (chi bao ve 1 phong)
@ControllerAdvice   = Bao ve toa nha (bao ve tat ca phong)
```

## Cach 1: @ExceptionHandler (trong 1 Controller)

```java
@RestController
public class UserController {

    @GetMapping("/users/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id)
            .orElseThrow(() -> new UserNotFoundException("User not found: " + id));
    }

    // Chi xu ly exception TRONG controller nay thoi
    @ExceptionHandler(UserNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(UserNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(404, ex.getMessage());
        return ResponseEntity.status(404).body(error);
    }
}
```

## Cach 2: @ControllerAdvice (TOAN CUC - KHUYEN DUNG)

```java
// Xu ly exception cho TAT CA controller
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

## Cach 3: Custom Exception voi @ResponseStatus

```java
@ResponseStatus(HttpStatus.NOT_FOUND)  // Tu dong tra ve 404
public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(String message) {
        super(message);
    }
}

// Khi throw exception nay, Spring tu dong tra ve HTTP 404
```

## Vi du day du: Cau truc Exception Handling tot

```java
// 1. Dinh nghia custom exceptions
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

// 3. Su dung trong Service
@Service
public class UserService {
    public User findById(Long id) {
        return userRepo.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User", id));
    }
}
```

## Diem quan trong nho phong van
1. **@ControllerAdvice** = xu ly exception **toan cuc** (best practice)
2. `@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`
3. Thu tu uu tien: **exception cu the** truoc, **Exception chung** sau
4. Nen tao **custom exception** rieng cho tung loai loi
