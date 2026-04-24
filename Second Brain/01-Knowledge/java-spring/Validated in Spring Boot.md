---
created: 2026-04-20
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/error-handling"
related:
  - "[[RestClient in Spring Boot]]"
---

# @Validated in Spring Boot

## 1. What

`@Validated` là annotation của Spring Framework (`org.springframework.validation.annotation.Validated`), dùng để kích hoạt **Bean Validation** (JSR-380) tại tầng service hoặc controller. Khác với `@Valid` của Jakarta Validation, `@Validated` hỗ trợ thêm hai tính năng mà `@Valid` không có: **Validation Groups** (chạy tập con rule validation tùy theo context) và **method-level validation** (validate tham số đầu vào và giá trị trả về của bất kỳ Spring bean method nào, không chỉ controller). `@Validated` là Spring-specific wrapper bao quanh cơ chế chuẩn của Hibernate Validator.

---

## 2. Why

`@Valid` của Jakarta Bean Validation chỉ hoạt động tốt khi validate toàn bộ object tại controller layer. Hai vấn đề phổ biến nó không giải quyết được:

**Vấn đề 1 — Context-dependent validation:**
Khi tạo user (`POST /users`), `password` là bắt buộc. Khi cập nhật user (`PUT /users/{id}`), `password` là optional. Nhưng cả hai dùng cùng class `UserRequest`. Với `@Valid`, không thể phân biệt rule nào áp dụng cho context nào.

**Vấn đề 2 — Service-layer validation:**
`@Valid` chỉ được Spring MVC kích hoạt tự động tại controller. Nếu một service method nhận tham số và cần validate, phải tự gọi `validator.validate()` thủ công. Không có cơ chế declarative.

`@Validated` giải quyết cả hai:
- **Validation Groups**: gắn group interface vào constraint annotation, rồi chỉ định group khi dùng `@Validated` — chỉ rule thuộc group đó được chạy.
- **Method validation via AOP**: đặt `@Validated` lên class Spring bean, framework tự intercept method call và validate tham số/return value qua AOP proxy.

---

## 3. Mental Model

Hãy tưởng tượng một **bộ quy tắc kiểm tra hành lý** tại sân bay:

`@Valid` giống như bộ quy tắc mặc định áp dụng cho tất cả hành khách: cân nặng tối đa 23kg, không có vật cấm. Áp dụng một lần, cho tất cả mọi người, tại một điểm kiểm tra duy nhất (security gate = controller).

`@Validated` giống như hệ thống kiểm tra theo **hạng vé**:
- Economy class: chỉ cần kiểm tra 23kg và vật cấm.
- Business class: thêm kiểm tra hành lý xách tay riêng.
- VIP lounge entry: chỉ kiểm tra thẻ VIP, không quan tâm hành lý.

Mỗi **điểm kiểm tra** (controller, service, method nào đó) có thể chọn bộ quy tắc phù hợp với context của nó — thay vì luôn chạy toàn bộ quy tắc.

---

## 4. Where it fits

```
HTTP Request
      |
      v
[Controller Layer]
  @Validated(OnCreate.class)  <- chỉ validate rules trong group OnCreate
  public ResponseEntity<?> createUser(@RequestBody @Validated(OnCreate.class) UserRequest req)
      |
      | (sau khi validate ở controller)
      v
[Service Layer]
  @Validated  <- kích hoạt method-level validation cho toàn bộ class
  public class UserService {
      public void updateEmail(@Email String email)  <- validate tham số method
  }
      |
      v
[AOP Proxy - MethodValidationInterceptor]
  Validate trước khi method thực thi, throw ConstraintViolationException nếu fail
      |
      v
[Business Logic]
```

`@Validated` hoạt động qua Spring AOP proxy, nên chỉ có tác dụng với Spring-managed beans và external method calls (không phải self-invocation trong cùng class).

---

## 5. When to use

- Cần **Validation Groups**: cùng DTO nhưng rule validate khác nhau tùy theo context (create vs update, draft vs publish).
- Cần **service-layer validation**: validate tham số của method trong `@Service` hoặc `@Component` mà không muốn lặp lại validation logic ở mọi caller.
- Validate **primitive types và String** trực tiếp làm tham số method: `@Email String email`, `@NotNull Long id`, `@Min(1) int pageSize`.
- Validate **return value** của method để đảm bảo response contract.
- Muốn validation logic gần với business rule hơn là tầng HTTP.

---

## 6. When NOT to use

- **Self-invocation trong cùng class**: `@Validated` dùng Spring AOP proxy, method A trong class X gọi method B trong cùng class X sẽ bypass proxy — validation không chạy. Phải inject service vào chính nó hoặc tách ra class khác.
- **Validate logic phức tạp phụ thuộc database** (ví dụ: kiểm tra email đã tồn tại chưa): đây là business validation, không phải bean validation. Đặt vào service method, throw custom exception.
- **Khi `@Valid` đã đủ**: nếu chỉ cần validate DTO đơn giản tại controller và không cần group, dùng `@Valid` — ít boilerplate hơn.
- **Trên `final` class hoặc method**: Spring AOP proxy dùng CGLIB, không wrap được `final` class/method. Validation sẽ silently không chạy.

---

## 7. Trade-offs

| Ưu điểm | Nhược điểm |
|---------|-----------|
| Validation Groups cho phép tái sử dụng DTO với rule khác nhau | Group interfaces thêm boilerplate (phải định nghĩa interface rỗng) |
| Method-level validation declarative, không cần code thủ công | AOP proxy — self-invocation bypass validation silently |
| Validate tham số primitive trực tiếp trên method signature | `ConstraintViolationException` (service layer) khác `MethodArgumentNotValidException` (controller) — cần xử lý riêng |
| Tích hợp tự nhiên với Spring Boot, không cần config thêm | Khó debug khi validation không chạy (proxy issue, final class, v.v.) |
| Kết hợp được với custom constraint annotation | Không hoạt động với method gọi nội bộ trong cùng bean |

---

## 8. Alternatives

| Giải pháp | Ưu điểm | Nhược điểm | Khi nào chọn |
|-----------|---------|-----------|-------------|
| `@Valid` (Jakarta) | Không cần config, đơn giản hơn | Không có groups, chỉ hoạt động tại controller MVC | Validate DTO đơn giản tại controller, không cần group |
| Manual validation (`Validator.validate()`) | Full control, hoạt động ở mọi nơi kể cả self-invocation | Verbose, phải tự inject `Validator`, tự handle result | Khi AOP proxy không phù hợp |
| Custom service method + exception | Rõ ràng, dễ đọc, không phụ thuộc annotation | Không tái sử dụng được, phải viết lại cho mỗi class | Business validation phức tạp (DB check, cross-field) |
| Precondition libraries (Guava, Apache Commons) | Lightweight, không cần Spring | Không tích hợp với Spring error handling | Non-Spring code, utility class |

---

## 9. How

**Dependency (đã có sẵn trong Spring Boot Web):**

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

**Bước 1 — Định nghĩa Validation Groups:**

```java
public interface OnCreate {}
public interface OnUpdate {}
```

**Bước 2 — Gắn group vào constraint trên DTO:**

```java
public class UserRequest {

    @Null(groups = OnCreate.class)       // id phải null khi tạo mới
    @NotNull(groups = OnUpdate.class)    // id phải có khi cập nhật
    private Long id;

    @NotBlank(groups = {OnCreate.class, OnUpdate.class})
    @Size(min = 2, max = 100)
    private String name;

    @NotBlank(groups = OnCreate.class)   // password bắt buộc khi tạo
    @Size(min = 8)
    private String password;

    @NotBlank
    @Email
    private String email;
}
```

**Bước 3 — Dùng `@Validated` với group tại controller:**

```java
@RestController
@RequestMapping("/users")
@RequiredArgsConstructor
public class UserController {

    private final UserService userService;

    @PostMapping
    public ResponseEntity<UserResponse> createUser(
            @RequestBody @Validated(OnCreate.class) UserRequest request) {
        return ResponseEntity.ok(userService.create(request));
    }

    @PutMapping("/{id}")
    public ResponseEntity<UserResponse> updateUser(
            @PathVariable Long id,
            @RequestBody @Validated(OnUpdate.class) UserRequest request) {
        return ResponseEntity.ok(userService.update(id, request));
    }
}
```

**Bước 4 — Method-level validation tại Service:**

```java
@Service
@Validated   // <-- kích hoạt method-level validation cho toàn bộ class
@RequiredArgsConstructor
public class UserService {

    // Validate tham số primitive trực tiếp
    public UserResponse findById(@NotNull @Positive Long id) {
        return userRepository.findById(id)
            .map(UserResponse::from)
            .orElseThrow(() -> new UserNotFoundException(id));
    }

    // Validate object tham số
    public UserResponse create(@Valid UserRequest request) {
        // @Valid ở đây trigger validation của UserRequest
        User user = User.from(request);
        return UserResponse.from(userRepository.save(user));
    }
}
```

**Bước 5 — Global exception handler:**

```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    // Lỗi từ controller (@RequestBody @Validated)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleMethodArgumentNotValid(
            MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(fe -> fe.getField() + ": " + fe.getDefaultMessage())
            .toList();
        return ResponseEntity.badRequest().body(new ErrorResponse("Validation failed", errors));
    }

    // Lỗi từ service layer (@Validated trên class + constraint trên tham số)
    @ExceptionHandler(ConstraintViolationException.class)
    public ResponseEntity<ErrorResponse> handleConstraintViolation(
            ConstraintViolationException ex) {
        List<String> errors = ex.getConstraintViolations()
            .stream()
            .map(cv -> cv.getPropertyPath() + ": " + cv.getMessage())
            .toList();
        return ResponseEntity.badRequest().body(new ErrorResponse("Validation failed", errors));
    }
}
```

---

## 10. Production concerns

**Exception type khác nhau theo layer:**
- Controller (`@RequestBody @Validated`): throw `MethodArgumentNotValidException`.
- Service method (`@Validated` trên class + constraint trên param): throw `ConstraintViolationException`.
- Phải handle cả hai trong `@RestControllerAdvice`, không thể dùng chung một handler.

**Self-invocation không được validate:**
Method A trong class X gọi trực tiếp method B trong cùng class X — Spring AOP proxy không intercept được. Validation annotation trên method B silently bị bỏ qua. Phát hiện lỗi này khó vì không có exception, chỉ là validation không chạy.

**Spring Boot 3.x yêu cầu `MethodValidationPostProcessor`:**
Trong Spring Boot 3.x với Spring Framework 6, `@Validated` trên class tự động đăng ký `MethodValidationInterceptor`. Nhưng nếu dùng Spring Boot 2.x, phải đăng ký bean `MethodValidationPostProcessor` thủ công trong config.

**Performance:**
Validation qua AOP có overhead nhỏ cho mỗi method call do reflection và proxy invocation. Với hot path có throughput cao (hàng nghìn call/giây), cân nhắc validate ở boundary (controller) thay vì từng service method call.

**Avoid validating in repository layer:**
Không đặt `@Validated` lên `@Repository` interface — JPA repositories dùng proxy phức tạp, có thể gây conflict. Validation thuộc về service layer trở lên.

---

## 11. Common mistakes

**Lỗi 1: Dùng `@Validated` trên class nhưng quên annotate tham số method**

```java
@Service
@Validated   // đúng — kích hoạt AOP
public class UserService {
    public UserResponse findById(Long id) {  // SAI — thiếu @NotNull trên id
        // validation không chạy vì không có constraint
    }
}
```

Fix: Luôn gắn constraint annotation lên tham số method khi muốn validate:

```java
public UserResponse findById(@NotNull @Positive Long id) { ... }
```

**Lỗi 2: Gọi method nội bộ trong cùng class và kỳ vọng validation chạy**

```java
@Service
@Validated
public class OrderService {

    public void processOrder(String orderId) {
        validateAndFetch(orderId);  // SAI — self-invocation, bypass AOP proxy
    }

    public Order validateAndFetch(@NotBlank String orderId) {
        return orderRepository.findById(orderId).orElseThrow();
    }
}
```

Fix: Tách thành class riêng để Spring proxy wrap được, hoặc inject `OrderService` vào chính nó qua `@Autowired` (ít dùng, code smell), hoặc dùng `Validator` thủ công.

**Lỗi 3: Thiếu `ConstraintViolationException` handler trong `@RestControllerAdvice`**

Validation fail ở service layer throw `ConstraintViolationException` nhưng `@RestControllerAdvice` chỉ có handler cho `MethodArgumentNotValidException`. Kết quả: client nhận HTTP 500 thay vì 400.

Fix: Luôn handle cả hai exception type trong global handler như ví dụ ở section 9.

**Lỗi 4: Dùng Validation Group nhưng quên chỉ định group mặc định**

```java
public class UserRequest {
    @NotBlank                          // không thuộc group nào -> chạy ở Default group
    private String name;

    @NotBlank(groups = OnCreate.class) // chỉ chạy ở OnCreate group
    private String password;
}
```

Khi gọi `@Validated(OnCreate.class)`, chỉ rule trong `OnCreate` group chạy. Rule `@NotBlank` trên `name` (thuộc `Default` group) KHÔNG chạy. Nếu muốn cả hai, phải dùng `@Validated({Default.class, OnCreate.class})`.

---

## 12. Sample project

**Bài tập: CRUD API với Create/Update validation groups**

Xây dựng REST API quản lý `Product` với hai endpoint:
- `POST /products` — tạo mới: `name` bắt buộc, `id` phải null, `price` bắt buộc và > 0.
- `PUT /products/{id}` — cập nhật: `id` bắt buộc, `name` optional (chỉ update nếu có), `price` nếu có phải > 0.

Hard constraints:
- Dùng chung một class `ProductRequest` cho cả hai endpoint với Validation Groups.
- Service layer có method `findById(@NotNull @Positive Long id)` với `@Validated` trên class.
- `@RestControllerAdvice` phải handle cả `MethodArgumentNotValidException` và `ConstraintViolationException`, trả về cùng một format error response.
- Viết unit test cho validator behavior: verify create endpoint reject request thiếu `name`, verify update endpoint accept request thiếu `name`.

---

## 13. Interview

**Core Q&A:**

Q: `@Validated` khác `@Valid` như thế nào?
A: `@Valid` là annotation chuẩn của Jakarta Bean Validation, chỉ trigger validation của object tại tầng mà Spring MVC tự động xử lý (controller). Không hỗ trợ Validation Groups, không validate method parameter ngoài controller. `@Validated` là Spring extension, hỗ trợ Validation Groups (chỉ chạy subset rule theo context) và method-level validation qua AOP (áp dụng cho bất kỳ Spring bean nào, không chỉ controller).

Q: Validation Groups là gì và khi nào dùng?
A: Validation Groups là cơ chế nhóm các constraint annotation theo interface marker. Khi gọi validation, chỉ định group nào cần chạy — chỉ constraint thuộc group đó được kích hoạt. Dùng khi cùng DTO cần rule validate khác nhau tùy context: tạo mới (`id` phải null) vs cập nhật (`id` bắt buộc), draft vs published state, v.v.

Q: Tại sao `@Validated` trên service method đôi khi không hoạt động?
A: Ba lý do phổ biến: (1) Self-invocation — method A gọi method B trong cùng class, bypass Spring AOP proxy. (2) Class hoặc method là `final` — CGLIB proxy không thể wrap. (3) Quên annotate tham số method với constraint annotation — `@Validated` chỉ kích hoạt cơ chế, còn cần constraint cụ thể trên tham số.

Q: Khi validation fail ở service layer với `@Validated`, exception nào được throw?
A: `ConstraintViolationException` (từ `jakarta.validation`), không phải `MethodArgumentNotValidException`. Đây là điểm khác biệt quan trọng so với validation ở controller layer. Cần handle cả hai loại trong global exception handler.

Q: Cần validate một object có nested object bên trong. Làm thế nào?
A: Gắn `@Valid` lên field là nested object trong DTO. Hibernate Validator sẽ cascade validation xuống object con. Ví dụ: `UserRequest` có field `@Valid AddressRequest address` — constraint trên `AddressRequest` sẽ được validate khi `UserRequest` được validate.

**Scenarios:**

Q: API tạo đơn hàng cần validate: khi tạo mới thì `customerId` bắt buộc, khi admin override thì `customerId` optional nhưng `adminNote` bắt buộc. Dùng `@Validated` thiết kế thế nào?
A: Định nghĩa hai group: `OnCustomerCreate` và `OnAdminCreate`. Trên `OrderRequest`: `@NotNull(groups = OnCustomerCreate.class) Long customerId` và `@NotBlank(groups = OnAdminCreate.class) String adminNote`. Controller endpoint tạo thường dùng `@Validated(OnCustomerCreate.class)`, endpoint admin dùng `@Validated(OnAdminCreate.class)`.

Q: Bạn đặt `@Validated` trên `UserService` và `@NotNull` trên tham số `findById(Long id)`, nhưng validation không chạy khi truyền `null`. Debug thế nào?
A: Kiểm tra theo thứ tự: (1) Xác nhận `spring-boot-starter-validation` trong classpath. (2) Kiểm tra `UserService` có phải Spring bean không (`@Service`, `@Component`). (3) Kiểm tra method có bị gọi qua self-invocation không. (4) Kiểm tra class hoặc method có phải `final` không. (5) Thêm breakpoint trong `MethodValidationInterceptor` để xác nhận interceptor có được invoke không.

---

## 14. References

- Spring Framework docs — Validation: https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#validation-beanvalidation
- Jakarta Bean Validation spec (JSR-380): https://beanvalidation.org/2.0/spec/
- Hibernate Validator docs (reference implementation): https://docs.jboss.org/hibernate/stable/validator/reference/en-US/html_single/
- Spring Boot docs — Validation: https://docs.spring.io/spring-boot/docs/current/reference/html/io.html#io.validation
- Baeldung — Spring @Validated vs @Valid: https://www.baeldung.com/spring-valid-vs-validated

---

## 15. Real-world Code

- Spring Framework source `MethodValidationInterceptor` — hiểu cơ chế AOP intercept: https://github.com/spring-projects/spring-framework/blob/main/spring-context/src/main/java/org/springframework/validation/beanvalidation/MethodValidationInterceptor.java
- Spring PetClinic — ví dụ validation trong project thực tế của Spring team: https://github.com/spring-projects/spring-petclinic
- Tìm trong các Spring Boot sample projects trên GitHub với keyword `@Validated groups` để xem cách team production tổ chức validation group interfaces.

---

## 16. Community

- Reddit: r/SpringBoot — tìm "@Validated @Valid difference"
- Stack Overflow: https://stackoverflow.com/questions/tagged/spring-validation — nhiều Q&A về `@Validated` vs `@Valid` và method validation
- Blog: Baeldung — "Difference Between @Valid and @Validated in Spring": https://www.baeldung.com/spring-valid-vs-validated
- Blog: reflectoring.io — "Bean Validation with Spring Boot": https://reflectoring.io/bean-validation-with-spring-boot/
- Talk: tìm "Spring Bean Validation" tại Spring I/O và SpringOne conference trên YouTube
