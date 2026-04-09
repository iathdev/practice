# Q2: How to validate form data in Spring Web MVC Framework?
> **Dịch:** Làm thế nào để validate dữ liệu form trong Spring Web MVC?

## Tra loi ngan gon
> Spring ho tro validation qua **Bean Validation API** (JSR-303/380) voi cac annotation nhu `@NotNull`, `@Size`, `@Email`... ket hop `@Valid` trong controller.

## Cach nho
```
Validation = Bao ve (check truoc khi cho data vao he thong)
@Valid     = "Nhan vien bao ve" o cua controller
```

## Cac buoc thuc hien

### Buoc 1: Them annotation vao Model
```java
public class UserForm {

    @NotBlank(message = "Ten khong duoc de trong")
    @Size(min = 2, max = 50, message = "Ten tu 2-50 ky tu")
    private String name;

    @NotBlank(message = "Email khong duoc de trong")
    @Email(message = "Email khong hop le")
    private String email;

    @Min(value = 18, message = "Phai tu 18 tuoi tro len")
    @Max(value = 100, message = "Tuoi khong hop le")
    private int age;

    @Pattern(regexp = "^0[0-9]{9}$", message = "SDT phai 10 so, bat dau bang 0")
    private String phone;

    // getters, setters
}
```

### Buoc 2: Dung @Valid trong Controller
```java
@RestController
public class UserController {

    @PostMapping("/users")
    public ResponseEntity<?> createUser(@Valid @RequestBody UserForm form,
                                         BindingResult result) {
        // Kiem tra co loi khong
        if (result.hasErrors()) {
            List<String> errors = result.getFieldErrors()
                .stream()
                .map(e -> e.getField() + ": " + e.getDefaultMessage())
                .collect(Collectors.toList());
            return ResponseEntity.badRequest().body(errors);
        }

        // Logic tao user...
        return ResponseEntity.ok("User created!");
    }
}
```

## Bang cac annotation validation thuong dung

| Annotation | Muc dich | Vi du |
|-----------|----------|-------|
| `@NotNull` | Khong duoc null | `@NotNull String name` |
| `@NotBlank` | Khong null, khong rong, khong chi space | `@NotBlank String name` |
| `@NotEmpty` | Khong null, khong rong (cho String/Collection) | `@NotEmpty List items` |
| `@Size` | Do dai / kich thuoc | `@Size(min=2, max=50)` |
| `@Min` / `@Max` | Gia tri so nho nhat / lon nhat | `@Min(18)` |
| `@Email` | Dinh dang email | `@Email String email` |
| `@Pattern` | Regex | `@Pattern(regexp="...")` |
| `@Past` / `@Future` | Ngay qua khu / tuong lai | `@Past LocalDate birthday` |

## Custom Validator (tu tao)
```java
// 1. Tao annotation
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PhoneValidator.class)
public @interface ValidPhone {
    String message() default "SDT khong hop le";
    Class<?>[] groups() default {};
    Class<?>[] payload() default {};
}

// 2. Tao validator class
public class PhoneValidator implements ConstraintValidator<ValidPhone, String> {
    @Override
    public boolean isValid(String phone, ConstraintValidatorContext ctx) {
        return phone != null && phone.matches("^0[0-9]{9}$");
    }
}

// 3. Su dung
public class UserForm {
    @ValidPhone
    private String phone;
}
```

## Diem quan trong nho phong van
1. Dung **@Valid** (JSR-303) hoac **@Validated** (Spring - ho tro group)
2. **BindingResult** phai dat **ngay sau** @Valid parameter
3. Spring Boot tu dong co `spring-boot-starter-validation`
4. Co the tao **Custom Validator** cho logic phuc tap
