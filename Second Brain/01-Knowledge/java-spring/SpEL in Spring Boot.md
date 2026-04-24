---
created: 2026-04-20
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/configuration"
related:
  - "[[Value in Spring Boot]]"
  - "[[ConfigurationProperties in Spring Boot]]"
---

# SpEL in Spring Boot

## 1. What

SpEL (Spring Expression Language) là ngôn ngữ biểu thức mạnh mẽ của Spring Framework, cho phép **tính toán expression tại runtime** trong ngữ cảnh của Spring container. SpEL được dùng qua cú pháp `#{...}` trong annotation như `@Value`, `@Cacheable`, `@PreAuthorize`, `@ConditionalOnExpression`, và trong XML config. SpEL hỗ trợ: truy cập property và method của bean, arithmetic và logical operators, collection projection và selection, ternary operator, regular expression matching, và gọi constructor. SpEL được xử lý bởi `SpelExpressionParser` và có thể dùng độc lập ngoài annotation.

---

## 2. Why

Spring beans đôi khi cần các giá trị không phải là hằng số hay property đơn giản, mà cần tính toán, phụ thuộc vào trạng thái runtime, hay kết hợp từ nhiều nguồn. Trước SpEL, những trường hợp này buộc phải viết Java code:

- Inject giá trị là kết quả gọi method của bean khác — không thể dùng `@Value("${...}")` thuần.
- Cấu hình cache key là biểu thức kết hợp nhiều tham số method: `#user.id + '_' + #region`.
- Phân quyền dựa trên đặc điểm của đối tượng đang được truy cập: `#entity.ownerId == authentication.id`.
- Inject giá trị tính toán từ system property: `#{systemProperties['cpu.count'] * 2}`.

SpEL cho phép nhúng những biểu thức này trực tiếp vào annotation mà không cần viết thêm Java method hoặc config class.

---

## 3. Mental Model

Hãy tưởng tượng Spring container như một **tòa nhà văn phòng lớn**. Mỗi Spring bean là một phòng ban — có tên, có nhân viên (method), có tài liệu (property).

`@Value("${app.timeout}")` giống như gọi điện thoại đến **tổng đài thông tin** (Environment): "cho tôi số nội bộ của phòng `app.timeout`". Tổng đài tra sổ danh bạ (properties file, env var) và đọc lại số.

`@Value("#{...}")` (SpEL) giống như nhờ một **trợ lý thông minh** có thể:
- Đi đến phòng `userService` và hỏi kết quả (`#{userService.currentRole}`).
- Tính toán: "lấy số từ tổng đài nhân đôi rồi trả cho tôi" (`#{${app.count} * 2}`).
- Ra quyết định theo điều kiện: `#{${app.isPeak} ? 'A' : 'B'}`.

Trợ lý SpEL biết đường đi trong toàn bộ tòa nhà và có thể thực hiện tính toán — không chỉ đơn giản đọc danh bạ.

---

## 4. Where it fits

```
[SpEL dùng trong annotation]

@Value("#{...}")                     <- inject vào field/constructor
@Cacheable(key = "#{...}")           <- tạo cache key từ tham số method
@PreAuthorize("#{...}")              <- kiểm tra quyền trước khi vào method
@ConditionalOnExpression("#{...}")   <- bật/tắt bean theo điều kiện
@Scheduled(cron = "#{...}")          <- cron expression từ property

          |
          v
[SpelExpressionParser — xử lý tại runtime]
          |
          v
[EvaluationContext — cung cấp bean, biến, function]
  - StandardEvaluationContext: truy cập bean, type, system
  - SimpleEvaluationContext: giới hạn scope, an toàn hơn

          |
          v
[Kết quả — inject vào bean / cache key / authorization decision]
```

SpEL nằm ở tầng cross-cutting, xuất hiện trong nhiều module của Spring: Core, Security, Cache, Data, Batch.

---

## 5. When to use

- **`@Value` với tính toán**: inject giá trị không phải là hằng số — tính toán từ property khác, kết quả gọi method.
- **Cache key phức tạp**: `@Cacheable(key = "#userId + ':' + #productId")` — key là tổ hợp tham số.
- **Method Security**: `@PreAuthorize("hasRole('ADMIN') or #order.customerId == authentication.name")` — kiểm tra quyền kết hợp với đặc điểm object.
- **Conditional bean**: `@ConditionalOnExpression("'${app.mode}' == 'production'")` — chỉ tạo bean theo điều kiện.
- **Dynamic cron**: `@Scheduled(cron = "#{@appProperties.cronExpression}")` — đọc cron từ property thay vì hardcode.
- **Spring Data query**: `@Query("select u from User u where u.email = ?#{[0]}")` — kết hợp JPQL với SpEL.

---

## 6. When NOT to use

- **Logic nghiệp vụ phức tạp**: SpEL expression dài hơn một dòng là dấu hiệu nên viết Java method thay thế — dễ test, dễ debug, IDE hỗ trợ.
- **Dữ liệu đầu vào của người dùng**: `StandardEvaluationContext` có thể thực thi arbitrary Java — là lỗ hổng bảo mật nghiêm trọng (SpEL injection) nếu cho phép user kiểm soát expression. Dùng `SimpleEvaluationContext` khi expression đến từ nguồn không tin cậy.
- **Thay thế toàn bộ logic Java**: SpEL không có type checking, không có IDE refactor support, không có compile-time error.
- **Hot path hiệu năng cao**: SpEL parse và xử lý có overhead so với native Java. Cache expression đã compile nếu dùng lặp lại nhiều lần.
- **Thay thế `@ConfigurationProperties`**: nếu chỉ cần đọc property, dùng `${...}` hoặc `@ConfigurationProperties` — đơn giản và rõ ràng hơn.

---

## 7. Trade-offs

| Ưu điểm | Nhược điểm |
|---------|-----------|
| Biểu đạt logic động trong annotation mà không cần Java method riêng | Magic string — không có compile-time check, IDE không refactor được |
| Tích hợp tất cả module Spring: Value, Security, Cache, Data, Batch | Khó debug khi expression sai — error message đôi khi không rõ ràng |
| Truy cập được toàn bộ Spring context: bean, environment, system | Nguy hiểm nếu xử lý user input với `StandardEvaluationContext` |
| Hỗ trợ collection projection, selection, aggregation | Không có type safety — lỗi type chỉ phát hiện lúc runtime |
| Có thể dùng độc lập ngoài annotation (programmatic API) | Khó test riêng biệt — phải có Spring context |
| Kết hợp `${}` và `#{}` trong cùng expression | Performance thấp hơn native Java trong hot path |

---

## 8. Alternatives

| Giải pháp | Ưu điểm | Nhược điểm | Khi nào chọn |
|-----------|---------|-----------|-------------|
| Java method thông thường | Type-safe, IDE support, dễ test | Không dùng được trong annotation trực tiếp | Logic phức tạp hơn một dòng |
| `${...}` property placeholder | Đơn giản, rõ ràng | Chỉ đọc được giá trị tĩnh từ Environment | Chỉ cần inject giá trị config |
| `@ConfigurationProperties` | Type-safe, IDE autocomplete, validate startup | Cần tạo class riêng | Nhóm config liên quan |
| Custom `PermissionEvaluator` (Security) | Type-safe, testable, tái sử dụng | Cần implement interface | Logic phân quyền phức tạp |

---

## 9. How

**Property và method access:**

```java
@Component
public class AppBean {
    public String getEnvironment()  { return "production"; }
    public int getMaxConnections()  { return 100; }
}
```

```java
@Service
public class SomeService {

    // Gọi method của Spring bean khác
    @Value("#{appBean.environment}")
    private String env;

    // Kết hợp ${} và #{}: đọc property rồi áp dụng SpEL
    @Value("#{${app.pool.size} * 2}")
    private int doublePoolSize;

    // Ternary operator
    @Value("#{${app.debug} ? 'DEBUG' : 'INFO'}")
    private String logLevel;
}
```

**System properties và type references:**

```java
// System property
@Value("#{systemProperties['java.version']}")
private String javaVersion;

// Số processor
@Value("#{T(Runtime).getRuntime().availableProcessors()}")
private int processorCount;

// Hằng số từ class
@Value("#{T(java.lang.Math).PI}")
private double pi;
```

**String operations:**

```java
// Uppercase từ property
@Value("#{'${app.name}'.toUpperCase()}")
private String appNameUpper;

// Concatenate
@Value("#{'${app.prefix}' + '-' + '${app.name}'}")
private String fullName;

// Regex match — trả về boolean
@Value("#{'${app.env}' matches 'prod.*'}")
private boolean isProduction;
```

**Collection projection và selection:**

```java
@Component
public class DataSource {
    public List<String> getActiveUsers() {
        return List.of("alice", "bob", "charlie", "admin");
    }
}
```

```java
// Selection: lọc phần tử thỏa điều kiện — ?[condition]
@Value("#{dataSource.activeUsers.?[length() > 4]}")
private List<String> longNameUsers;  // ["alice", "charlie", "admin"]

// Projection: transform từng phần tử — ![expression]
@Value("#{dataSource.activeUsers.![toUpperCase()]}")
private List<String> upperNames;  // ["ALICE", "BOB", "CHARLIE", "ADMIN"]

// First matching — ^[condition]
@Value("#{dataSource.activeUsers.^[startsWith('a')]}")
private String firstA;  // "alice"

// Last matching — $[condition]
@Value("#{dataSource.activeUsers.$[startsWith('a')]}")
private String lastA;   // "admin"
```

**Cache key với `@Cacheable`:**

```java
@Service
public class ProductService {

    // Cache key là tổ hợp nhiều tham số
    @Cacheable(value = "products", key = "#category + ':' + #page + ':' + #size")
    public List<Product> findByCategory(String category, int page, int size) { ... }

    // Cache key từ property của object tham số
    @Cacheable(value = "products", key = "#filter.category + ':' + #filter.brand")
    public List<Product> findByFilter(ProductFilter filter) { ... }

    // Conditional cache: chỉ cache nếu result không rỗng
    @Cacheable(value = "products", key = "#id", unless = "#result == null")
    public Product findById(Long id) { ... }

    // Cache evict theo điều kiện
    @CacheEvict(value = "products", key = "#product.category",
                condition = "#product.active == true")
    public void updateProduct(Product product) { ... }
}
```

**Method Security với `@PreAuthorize`:**

```java
@Service
public class OrderService {

    // Chỉ ADMIN hoặc chủ sở hữu mới được xem
    @PreAuthorize("hasRole('ADMIN') or #order.customerId == authentication.name")
    public Order getOrder(Order order) { ... }

    // Kiểm tra tham số trực tiếp
    @PreAuthorize("hasRole('USER') and #userId == authentication.principal.id")
    public UserProfile getProfile(Long userId) { ... }

    // PostAuthorize — kiểm tra sau khi method chạy, trước khi return
    @PostAuthorize("returnObject.ownerId == authentication.name")
    public Document getDocument(Long docId) { ... }
}
```

**Dynamic `@Scheduled`:**

```java
@ConfigurationProperties(prefix = "app.scheduler")
public record SchedulerProperties(String reportCron, String cleanupCron) {}

@Component
@RequiredArgsConstructor
public class ScheduledTasks {

    @Scheduled(cron = "#{@schedulerProperties.reportCron}")
    public void generateReport() { ... }

    @Scheduled(cron = "#{@schedulerProperties.cleanupCron}")
    public void cleanup() { ... }
}
```

**Programmatic SpEL — dùng ngoài annotation:**

```java
@Service
public class DynamicExpressionService {

    // SimpleEvaluationContext — an toàn hơn, không cho phép gọi arbitrary Java class
    public Object process(String expression, Map<String, Object> variables) {
        ExpressionParser parser = new SpelExpressionParser();
        SimpleEvaluationContext context = SimpleEvaluationContext
            .forReadOnlyDataBinding()
            .build();

        variables.forEach(context::setVariable);

        return parser.parseExpression(expression).getValue(context);
    }
}

// Dùng:
service.process("#name.toUpperCase()", Map.of("name", "alice"));  // "ALICE"
```

**Cache compiled expression:**

```java
@Service
public class ExpressionService {

    // Parse một lần, tái dùng nhiều lần
    private final Expression nameExpr =
        new SpelExpressionParser().parseExpression("name.toUpperCase()");

    public String extractName(User user) {
        EvaluationContext ctx = SimpleEvaluationContext
            .forReadOnlyDataBinding()
            .withRootObject(user)
            .build();
        return nameExpr.getValue(ctx, String.class);
    }
}
```

---

## 10. Production concerns

**Bảo mật — SpEL injection:**
`StandardEvaluationContext` cho phép truy cập toàn bộ Java class thông qua `T(ClassName)`. Nếu expression đến từ user input và được xử lý với `StandardEvaluationContext`, attacker có thể thực thi arbitrary Java code — đây là lỗ hổng SpEL injection, tương tự SQL injection nhưng nguy hiểm hơn (có thể leo thang lên RCE). Đây là class vulnerabilities đã có CVE được ghi nhận trong Spring.

Fix: Luôn dùng `SimpleEvaluationContext` khi expression đến từ input không tin cậy:

```java
SimpleEvaluationContext context = SimpleEvaluationContext
    .forReadOnlyDataBinding()
    .build();
```

**Cache compiled expression trong hot path:**
Mỗi lần gọi `parser.parseExpression(str)` đều parse string — overhead không cần thiết nếu expression không thay đổi. Parse một lần khi khởi tạo bean, cache `Expression` object, tái dùng nhiều lần.

**Null safety trong cache key:**
Nếu tham số method có thể là `null`, cache key expression sẽ throw `NullPointerException`. Dùng safe navigation operator `?.` và Elvis operator `?:`:

```java
@Cacheable(key = "#filter?.category ?: 'all'")
public List<Product> find(ProductFilter filter) { ... }
```

**`@PreAuthorize` performance:**
`@PreAuthorize` chạy trên mọi method call. Expression gọi nhiều service sẽ tăng latency. Tách logic phức tạp ra custom `PermissionEvaluator` để cache và test độc lập.

**SpEL trong Spring Data `@Query`:**
`?#{[0]}` trong JPQL query thay thế tham số positional. `?#{principal.username}` inject authenticated user vào query trực tiếp — tiện nhưng cần cẩn thận không để lọt filter, gây data leak cross-tenant.

---

## 11. Common mistakes

**Lỗi 1: Nhầm `${}` với `#{}`**

```java
@Value("#{app.timeout}")   // SAI — SpEL tìm bean tên "app" và gọi getTimeout()
private int timeout;       // NoSuchBeanDefinitionException hoặc giá trị sai

@Value("${app.timeout}")   // ĐÚNG — đọc property từ Environment
private int timeout;
```

`${}` đọc từ Spring `Environment`. `#{}` xử lý SpEL expression. Để đọc property trong SpEL phải kết hợp: `@Value("#{${app.timeout}}")` hoặc `@Value("#{environment['app.timeout']}")`.

**Lỗi 2: Xử lý user input với `StandardEvaluationContext` — lỗ hổng bảo mật nghiêm trọng**

```java
// SAI — không bao giờ xử lý user-provided expression với StandardEvaluationContext
@PostMapping("/calculate")
public String calculate(@RequestBody String expression) {
    return parser.parseExpression(expression)
        .getValue(new StandardEvaluationContext(), String.class);
    // Attacker có thể gọi arbitrary Java class qua T(...) syntax
}
```

Fix: Dùng `SimpleEvaluationContext` với `forReadOnlyDataBinding()`. Nếu không cần tính năng này, xóa hoàn toàn.

**Lỗi 3: Không dùng safe navigation operator khi property có thể null**

```java
// SAI — NullPointerException nếu address là null
@Cacheable(key = "#user.address.city")
public List<Order> findOrders(User user) { ... }
```

Fix:

```java
// ĐÚNG — trả về "unknown" nếu address hoặc city là null
@Cacheable(key = "#user.address?.city ?: 'unknown'")
public List<Order> findOrders(User user) { ... }
```

**Lỗi 4: SpEL expression quá phức tạp trong `@PreAuthorize` — không thể test**

```java
// SAI — logic ẩn trong string, không test được
@PreAuthorize("hasRole('ADMIN') or " +
    "(hasRole('MANAGER') and #order.department == authentication.principal.department " +
    "and #order.amount < 50000)")
public void approveOrder(Order order) { ... }
```

Fix: Tách ra custom evaluator:

```java
// ĐÚNG — ngắn gọn, delegate sang Java method có thể test
@PreAuthorize("hasRole('ADMIN') or @orderPermissions.canApprove(authentication, #order)")
public void approveOrder(Order order) { ... }
```

---

## 12. Sample project

**Bài tập: Dynamic Permission System với SpEL**

Xây dựng REST API quản lý `Document` với phân quyền động:

1. `GET /documents/{id}` — chỉ owner hoặc ADMIN được xem. Dùng `@PostAuthorize("returnObject.ownerId == authentication.name or hasRole('ADMIN')")`.
2. `PUT /documents/{id}` — chỉ owner trong cùng department được sửa. Dùng `@PreAuthorize` với custom `PermissionEvaluator`.
3. `GET /documents` — dùng `@Cacheable` với key là `"#page + ':' + #size + ':' + authentication.name"`.
4. `DELETE /documents/{id}` — chỉ ADMIN.

Hard constraints:
- Permission logic phức tạp hơn role check đơn giản phải nằm trong `DocumentPermissionEvaluator`, không inline SpEL string.
- Viết unit test cho `DocumentPermissionEvaluator` không cần Spring context.
- Cache evict khi document bị update hoặc delete.
- Mọi programmatic SpEL phải dùng `SimpleEvaluationContext`, không phải `StandardEvaluationContext`.

---

## 13. Interview

**Core Q&A:**

Q: SpEL là gì và dùng ở đâu trong Spring?
A: SpEL (Spring Expression Language) là ngôn ngữ biểu thức của Spring cho phép xử lý expression động tại runtime trong ngữ cảnh Spring container. Dùng trong: `@Value("#{...}")` inject giá trị tính toán, `@Cacheable(key = "#{...}")` tạo cache key, `@PreAuthorize("#{...}")` phân quyền động, `@ConditionalOnExpression` bật/tắt bean, `@Scheduled(cron = "#{...}")` đọc cron từ config, Spring Data `@Query` inject vào JPQL.

Q: `${}` và `#{}` trong `@Value` khác nhau thế nào?
A: `${}` là property placeholder — đọc giá trị từ Spring `Environment` (properties file, env var, system properties). `#{}` là SpEL — xử lý Java expression: gọi method, truy cập bean, tính toán, regex match, collection filter. Hai cú pháp có thể kết hợp: `@Value("#{${app.count} * 2}")` — đọc property `app.count` từ Environment rồi nhân đôi qua SpEL.

Q: `StandardEvaluationContext` và `SimpleEvaluationContext` khác nhau thế nào?
A: `StandardEvaluationContext` cho phép truy cập toàn bộ Java class qua `T(ClassName)` và gọi bất kỳ method nào — mạnh nhưng nguy hiểm nếu xử lý user input (SpEL injection). `SimpleEvaluationContext` giới hạn scope: không cho `T(...)` type references, không gọi constructor tùy tiện — an toàn cho expression đến từ input bên ngoài. Dùng `StandardEvaluationContext` cho annotation trong code. Dùng `SimpleEvaluationContext` khi expression đến từ input không tin cậy.

Q: Safe navigation operator `?.` trong SpEL dùng khi nào?
A: Khi truy cập property trên object có thể là `null`, `?.` tránh `NullPointerException` bằng cách trả về `null` thay vì throw. `#user.address?.city` trả về `null` nếu `address` là null. Thường kết hợp với Elvis operator `?:` để cung cấp default: `#user.address?.city ?: 'unknown'`.

Q: Tại sao không nên viết SpEL phức tạp inline trong `@PreAuthorize`?
A: SpEL trong `@PreAuthorize` là magic string — không có compile-time check, không có IDE navigation, không có type safety, không test được riêng biệt. Fix: tách logic phức tạp ra `PermissionEvaluator` với method Java thông thường — testable, readable, có thể cache kết quả. Giữ SpEL trong `@PreAuthorize` ngắn gọn: chỉ role check đơn giản hoặc gọi một custom evaluator method.

**Scenarios:**

Q: `@Cacheable(key = "#filter.category + ':' + #filter.brand")` throw `NullPointerException` khi `filter.brand` là null. Fix thế nào?
A: Dùng safe navigation và Elvis: `key = "#filter.category + ':' + (#filter.brand ?: 'all')"`. Trả về chuỗi `'all'` khi `brand` là null, đảm bảo cache key luôn là String hợp lệ và không xảy ra collision.

Q: Bạn cần tạo cache key từ object phức tạp có nhiều field. SpEL string dài hay cách khác?
A: Với nhiều field, tốt hơn là implement `hashCode()` cẩn thận trên object và dùng `key = "#filter.hashCode()"`, hoặc tạo custom `KeyGenerator` bean. SpEL string dài dễ bị thiếu field khi object thay đổi sau này — nếu thêm field mới mà quên cập nhật SpEL, cache sẽ collision sai một cách im lặng.

---

## 14. References

- Spring Framework docs — Spring Expression Language: https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#expressions
- Spring Framework docs — SpEL Language Reference: https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#expressions-language-ref
- Spring Security docs — Method Security với SpEL: https://docs.spring.io/spring-security/reference/servlet/authorization/method-security.html
- Spring Cache docs — Cache SpEL context variables: https://docs.spring.io/spring-framework/docs/current/reference/html/integration.html#cache-spel-context
- Baeldung — Spring Expression Language Guide: https://www.baeldung.com/spring-expression-language

---

## 15. Real-world Code

- Spring Framework source — `SpelExpressionParser` và evaluation context: https://github.com/spring-projects/spring-framework/tree/main/spring-expression/src/main/java/org/springframework/expression/spel
- Spring Security source — `@PreAuthorize` processing với SpEL: https://github.com/spring-projects/spring-security/tree/main/core/src/main/java/org/springframework/security/access/prepost
- Spring Data source — `@Query` với SpEL extension: https://github.com/spring-projects/spring-data-commons/tree/main/src/main/java/org/springframework/data/repository/query

---

## 16. Community

- Reddit: r/SpringBoot — tìm "SpEL expressions" và "PreAuthorize SpEL"
- Stack Overflow tag: `spring-el`: https://stackoverflow.com/questions/tagged/spring-el
- Blog: Baeldung — "Spring Expression Language Guide": https://www.baeldung.com/spring-expression-language
- Blog: reflectoring.io — "Spring Security Method Security": https://reflectoring.io/spring-security-method-security/
- Talk: "Spring Security Method Security" tại Spring I/O conference trên YouTube
