# Q2: How to validate form data in Spring Web MVC Framework?
> **Dịch:** Làm thế nào để validate dữ liệu form trong Spring Web MVC?

## Trả lời ngắn gọn
> Spring hỗ trợ validation qua **Bean Validation API** (JSR-303/380) với các annotation như `@NotNull`, `@Size`, `@Email`... kết hợp `@Valid` trong controller.

## Cách nhớ
```
Validation = Bảo vệ (check trước khi cho data vào hệ thống)
@Valid     = "Nhân viên bảo vệ" ở cửa controller
```

## Các bước thực hiện

### Bước 1: Thêm annotation vào Model
```java
public class UserForm {

    @NotBlank(message = "Tên không được để trống")
    @Size(min = 2, max = 50, message = "Tên từ 2-50 ký tự")
    private String name;

    @NotBlank(message = "Email không được để trống")
    @Email(message = "Email không hợp lệ")
    private String email;

    @Min(value = 18, message = "Phải từ 18 tuổi trở lên")
    @Max(value = 100, message = "Tuổi không hợp lệ")
    private int age;

    @Pattern(regexp = "^0[0-9]{9}$", message = "SĐT phải 10 số, bắt đầu bằng 0")
    private String phone;

    // getters, setters
}
```

### Bước 2: Dùng @Valid trong Controller
```java
@RestController
public class UserController {

    @PostMapping("/users")
    public ResponseEntity<?> createUser(@Valid @RequestBody UserForm form,
                                         BindingResult result) {
        // Kiểm tra có lỗi không
        if (result.hasErrors()) {
            List<String> errors = result.getFieldErrors()
                .stream()
                .map(e -> e.getField() + ": " + e.getDefaultMessage())
                .collect(Collectors.toList());
            return ResponseEntity.badRequest().body(errors);
        }

        // Logic tạo user...
        return ResponseEntity.ok("User created!");
    }
}
```

## Bảng các annotation validation thường dùng

| Annotation | Mục đích | Ví dụ |
|-----------|----------|-------|
| `@NotNull` | Không được null | `@NotNull String name` |
| `@NotBlank` | Không null, không rỗng, không chỉ space | `@NotBlank String name` |
| `@NotEmpty` | Không null, không rỗng (cho String/Collection) | `@NotEmpty List items` |
| `@Size` | Độ dài / kích thước | `@Size(min=2, max=50)` |
| `@Min` / `@Max` | Giá trị số nhỏ nhất / lớn nhất | `@Min(18)` |
| `@Email` | Định dạng email | `@Email String email` |
| `@Pattern` | Regex | `@Pattern(regexp="...")` |
| `@Past` / `@Future` | Ngày quá khứ / tương lai | `@Past LocalDate birthday` |

## Custom Validator (tự tạo)
```java
// 1. Tạo annotation
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PhoneValidator.class)
public @interface ValidPhone {
    String message() default "SĐT không hợp lệ";
    Class<?>[] groups() default {};
    Class<?>[] payload() default {};
}

// 2. Tạo validator class
public class PhoneValidator implements ConstraintValidator<ValidPhone, String> {
    @Override
    public boolean isValid(String phone, ConstraintValidatorContext ctx) {
        return phone != null && phone.matches("^0[0-9]{9}$");
    }
}

// 3. Sử dụng
public class UserForm {
    @ValidPhone
    private String phone;
}
```

## Điểm quan trọng nhớ phỏng vấn
1. Dùng **@Valid** (JSR-303) hoặc **@Validated** (Spring - hỗ trợ group)
2. **BindingResult** phải đặt **ngay sau** @Valid parameter
3. Spring Boot tự động có `spring-boot-starter-validation`
4. Có thể tạo **Custom Validator** cho logic phức tạp
