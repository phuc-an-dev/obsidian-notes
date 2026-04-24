---
created: 2026-04-20
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/configuration"
related:
  - "[[ConfigurationProperties in Spring Boot]]"
  - "[[Validated in Spring Boot]]"
---

# @Value in Spring Boot

## 1. What

`@Value` là annotation của Spring Framework (`org.springframework.beans.factory.annotation.Value`) dùng để **inject một giá trị đơn lẻ** vào field, constructor parameter, hoặc method parameter của Spring bean. Giá trị có thể đến từ ba nguồn: property placeholder (`${key}`), Spring Expression Language — SpEL (`#{expression}`), hoặc literal value. `@Value` được xử lý bởi `AutowiredAnnotationBeanPostProcessor` và hỗ trợ type conversion tự động từ String sang `int`, `boolean`, `List`, `Map`, v.v.

---

## 2. Why

Ứng dụng Spring Boot cần đọc config từ môi trường ngoài (environment variable, `application.properties`, `application.yml`) để tách cấu hình khỏi code — theo nguyên tắc 12-factor app. Thay vì hardcode giá trị trực tiếp trong Java:

```java
private int timeout = 5000;  // SAI — hardcode, không thể thay đổi theo môi trường
```

Developer cần cơ chế inject từ bên ngoài. `@Value` là giải pháp đơn giản nhất cho trường hợp inject một property đơn lẻ mà không cần tạo thêm class config riêng. Đặc biệt hữu ích khi:

- Bean chỉ cần một hoặc hai property từ config.
- Cần inject kết quả tính toán từ SpEL expression.
- Inject default value khi property không tồn tại.
- Inject toàn bộ nội dung file (`classpath:file.txt`) vào String.

---

## 3. Mental Model

Hãy tưởng tượng `@Value` như một **tờ phiếu yêu cầu cấp phát vật tư** trong kho:

Khi nhân viên kho (`BeanPostProcessor`) đọc phiếu `@Value("${app.timeout}")`, họ tra cứu mã vật tư `app.timeout` trong **bản kiểm kê kho** (`Environment` — tổng hợp từ `application.properties`, env var, system properties theo thứ tự ưu tiên). Nếu tìm thấy, họ lấy vật tư ra, **chuyển đổi sang đúng loại** (String sang int), và đặt vào đúng vị trí trong bean.

Nếu phiếu có ghi chú dự phòng `@Value("${app.timeout:5000}")`, nhân viên dùng hàng dự phòng `5000` khi kho hết mã đó.

`#{expression}` (SpEL) thì khác — giống như phiếu ghi công thức tính toán: "lấy vật tư A, nhân với 2, cộng với kích thước đơn hàng hiện tại". Nhân viên kho phải tính toán trước rồi mới giao.

---

## 4. Where it fits

```
[Nguồn config — theo thứ tự ưu tiên giảm dần]
  1. Command-line args          (--app.timeout=3000)
  2. OS environment variables   (APP_TIMEOUT=3000)
  3. application-{profile}.yml
  4. application.yml
  5. @PropertySource files
  6. Default values trong @Value

           |
           | Spring Environment abstraction
           v

[Spring Bean]
  @Value("${app.timeout:5000}")
  private int timeout;

  @Value("${app.name}")
  private String appName;

  @Value("#{systemProperties['java.version']}")  <- SpEL
  private String javaVersion;

           |
           | AutowiredAnnotationBeanPostProcessor
           v

[Runtime: field/constructor/method param đã được inject]
```

`@Value` hoạt động sau khi bean được khởi tạo (post-processing), không phải tại compile time.

---

## 5. When to use

- Bean chỉ cần **một hoặc hai property** — không đủ để tạo `@ConfigurationProperties` class riêng.
- Cần inject **default value** khi property không tồn tại: `@Value("${feature.enabled:false}")`.
- Cần inject **kết quả SpEL expression**: giá trị tính toán, gọi method của bean khác, đọc system property.
- Inject **nội dung file** từ classpath: `@Value("classpath:templates/email.html")` inject vào `Resource`.
- Inject **environment variable** đơn giản: `@Value("${SERVER_PORT:8080}")`.
- Prototype bean hoặc non-Spring class cần một giá trị config mà không muốn phụ thuộc `@ConfigurationProperties`.

---

## 6. When NOT to use

- **Nhóm từ 3 property trở lên có cùng prefix**: dùng `@ConfigurationProperties` — gọn hơn, type-safe hơn, IDE autocomplete, validation khi startup.
- **Type phức tạp cần nested structure**: `@Value` không bind được nested object — dùng `@ConfigurationProperties` với record lồng nhau.
- **Validation khi startup**: `@Value` không tích hợp với Bean Validation — không phát hiện lỗi config trước khi ứng dụng nhận traffic. `@ConfigurationProperties` + `@Validated` làm tốt hơn.
- **Field của `static` method hoặc `static` field**: `@Value` không inject vào `static` field — sẽ luôn là `null`. Phải dùng setter injection với static field (antipattern) hoặc `@ConfigurationProperties`.
- **Refactor thường xuyên**: tên key là magic string trong annotation — đổi tên property phải tìm-thay-thế thủ công, không có compile-time check.

---

## 7. Trade-offs

| Ưu điểm | Nhược điểm |
|---------|-----------|
| Đơn giản, ít boilerplate cho 1-2 property | Tên property là magic string — không compile-time safe |
| Hỗ trợ default value inline: `${key:default}` | Không validate khi startup — lỗi chỉ phát hiện lúc runtime |
| SpEL expression mạnh: tính toán, gọi method bean | Khó refactor — đổi tên key phải tìm tất cả `@Value` string |
| Inject được `Resource`, `List`, `Map`, primitives | Không IDE autocomplete cho property name |
| Hoạt động trên field, constructor, method param | Không inject được vào `static` field trực tiếp |
| Không cần class config riêng | Khó test — phải dùng `@TestPropertySource` hoặc `ReflectionTestUtils` |

---

## 8. Alternatives

| Giải pháp | Ưu điểm | Nhược điểm | Khi nào chọn |
|-----------|---------|-----------|-------------|
| `@ConfigurationProperties` | Type-safe, group config, validate startup, IDE autocomplete | Cần tạo class riêng | Nhóm 3+ property liên quan |
| `Environment.getProperty()` | Programmatic, linh hoạt, dynamic lookup | Verbose, không type-safe | Dynamic config lookup lúc runtime |
| Constructor injection + `@Value` | Immutable field, dễ test hơn field injection | Vẫn magic string | Khi ưu tiên immutability |
| Spring Cloud Config | Centralized, hot-reload, multi-environment | Cần infrastructure thêm | Microservices với nhiều service |

---

## 9. How

**Property placeholder cơ bản:**

```properties
# application.properties
app.name=My Application
app.timeout=5000
app.max-retries=3
app.feature-enabled=true
```

```java
@Component
public class AppConfig {

    @Value("${app.name}")
    private String appName;

    @Value("${app.timeout}")
    private int timeout;

    @Value("${app.max-retries}")
    private int maxRetries;

    @Value("${app.feature-enabled}")
    private boolean featureEnabled;
}
```

**Default value khi property không tồn tại:**

```java
@Value("${app.timeout:5000}")          // default 5000 nếu không có key
private int timeout;

@Value("${app.name:My App}")           // default string
private String appName;

@Value("${app.feature-enabled:false}") // default false
private boolean featureEnabled;
```

**Inject List và Map:**

```properties
app.allowed-origins=http://localhost:3000,http://localhost:4200
app.headers.content-type=application/json
app.headers.accept=application/json
```

```java
@Value("${app.allowed-origins}")
private List<String> allowedOrigins;  // Spring tự split bằng dấu phẩy

@Value("#{${app.headers}}")           // SpEL để parse Map
private Map<String, String> headers;
```

**Inject Resource (đọc file):**

```java
@Value("classpath:templates/welcome-email.html")
private Resource emailTemplate;

public String loadEmailTemplate() throws IOException {
    return emailTemplate.getContentAsString(StandardCharsets.UTF_8);
}
```

**SpEL expressions:**

```java
// System property
@Value("#{systemProperties['java.version']}")
private String javaVersion;

// Environment variable
@Value("#{environment['HOME']}")
private String homeDir;

// Gọi method của Spring bean khác
@Value("#{userService.getDefaultRole()}")
private String defaultRole;

// Biểu thức toán học và ternary
@Value("#{${app.timeout} * 2}")
private int doubleTimeout;

@Value("#{${app.feature-enabled} ? 'ENABLED' : 'DISABLED'}")
private String featureStatus;
```

**Constructor injection (khuyến nghị hơn field injection):**

```java
@Service
public class MailService {

    private final String smtpHost;
    private final int smtpPort;

    public MailService(
            @Value("${app.mail.host}") String smtpHost,
            @Value("${app.mail.port:587}") int smtpPort) {
        this.smtpHost = smtpHost;
        this.smtpPort = smtpPort;
    }
}
```

**Inject AWS Secrets Manager secret (Spring Cloud AWS):**

```java
// Secret được inject từ AWS Secrets Manager sau khi Spring Cloud AWS fetch
@Value("${/myapp/db-password}")
private String dbPassword;
```

**Test với `@Value`:**

```java
@SpringBootTest
@TestPropertySource(properties = {
    "app.name=Test App",
    "app.timeout=1000"
})
class AppConfigTest {

    @Autowired
    private AppConfig appConfig;

    @Test
    void shouldInjectProperties() {
        assertThat(appConfig.getAppName()).isEqualTo("Test App");
        assertThat(appConfig.getTimeout()).isEqualTo(1000);
    }
}
```

**Test unit không cần Spring context (dùng `ReflectionTestUtils`):**

```java
class MailServiceTest {

    private MailService mailService;

    @BeforeEach
    void setUp() {
        mailService = new MailService("smtp.test.com", 587);
        // Với constructor injection: truyền thẳng giá trị, không cần Spring
    }
}
```

---

## 10. Production concerns

**Thứ tự ưu tiên của property source:**
Spring Boot đọc property theo thứ tự ưu tiên giảm dần:
1. Command-line arguments (`--app.timeout=3000`).
2. `SPRING_APPLICATION_JSON` environment variable.
3. OS environment variables (`APP_TIMEOUT=3000`).
4. `application-{profile}.properties` / `.yml`.
5. `application.properties` / `.yml`.
6. Default value trong `@Value`.

Environment variable luôn override file config — quan trọng khi deploy lên container hay Kubernetes.

**Không inject vào `static` field:**
`@Value` trên `static` field không được Spring xử lý — field luôn là `null`. Đây là silent failure nguy hiểm.

```java
// SAI — static field, @Value không hoạt động
@Value("${app.name}")
private static String appName;  // luôn null
```

Fix: Dùng setter injection với lưu vào static field (không khuyến nghị, chỉ dùng khi thật sự cần):

```java
private static String appName;

@Value("${app.name}")
public void setAppName(String name) {
    AppConfig.appName = name;
}
```

Hoặc tốt hơn: dùng instance field và inject bean vào nơi cần.

**`@Value` trong `@Configuration` class:**
`@Value` hoạt động bình thường trong `@Configuration`. Tuy nhiên, khi dùng trong `@Bean` method parameter, Spring inject từ environment — không cần `@Autowired`:

```java
@Configuration
public class DataSourceConfig {

    @Bean
    public DataSource dataSource(
            @Value("${db.url}") String url,
            @Value("${db.username}") String username,
            @Value("${db.password}") String password) {
        // ...
    }
}
```

**Property không tồn tại và không có default:**
Nếu property không tồn tại và không có default value, Spring throw `IllegalArgumentException` khi khởi động với message: `Could not resolve placeholder '...' in value "${...}"`. Luôn set default value hoặc đảm bảo property tồn tại trong mọi environment.

**SpEL và security:**
Tránh build SpEL expression từ input của user — SpEL có thể thực thi arbitrary code. `@Value` với SpEL chỉ dùng cho static expression tại compile time, không phải dynamic expression lúc runtime.

---

## 11. Common mistakes

**Lỗi 1: Inject vào `static` field — luôn là `null`**

```java
@Component
public class AppConstants {

    @Value("${app.name}")
    private static String appName;  // SAI — luôn null

    public static String getAppName() {
        return appName;  // trả về null
    }
}
```

Fix: Dùng instance field hoặc dùng `@ConfigurationProperties`. Nếu bắt buộc cần static, inject qua setter.

**Lỗi 2: Dùng `@Value` trong class không được quản lý bởi Spring**

```java
public class EmailFormatter {  // SAI — không có @Component, @Service, v.v.

    @Value("${app.name}")
    private String appName;  // luôn null — Spring không xử lý class này

    public String format(String message) {
        return "[" + appName + "] " + message;  // NullPointerException
    }
}
```

Fix: Đăng ký class với Spring (`@Component`) hoặc inject giá trị qua constructor khi tạo object thủ công.

**Lỗi 3: Thiếu default value — ứng dụng crash khi deploy lên môi trường thiếu config**

```java
@Value("${app.feature-flag}")  // SAI nếu không phải mọi môi trường đều có key này
private boolean featureEnabled;
// IllegalArgumentException: Could not resolve placeholder 'app.feature-flag'
```

Fix: Luôn set default cho property tùy chọn:

```java
@Value("${app.feature-flag:false}")
private boolean featureEnabled;
```

**Lỗi 4: Nhầm `${}` với `#{}`**

```java
@Value("#{app.timeout}")   // SAI — SpEL expression, tìm bean tên "app" và gọi getTimeout()
private int timeout;

@Value("${app.timeout}")   // ĐÚNG — property placeholder, đọc từ Environment
private int timeout;
```

`${}` đọc từ Spring `Environment` (properties file, env var). `#{}` là SpEL — evaluate Java expression, có thể gọi method, truy cập bean. Nhầm hai cái này gây ra lỗi khó hiểu khi startup.

---

## 12. Sample project

**Bài tập: Feature Flag Service**

Xây dựng `FeatureFlagService` với các feature flag inject từ `application.properties`:

- `feature.dark-mode.enabled` (boolean, default `false`)
- `feature.max-upload-size-mb` (int, default `10`)
- `feature.allowed-roles` (List<String>, default `USER,ADMIN`)
- `feature.maintenance-message` (String, default `"Hệ thống đang bảo trì"`)
- `feature.api-rate-limit` (int, default `100`)

Hard constraints:
- Tất cả inject qua constructor (không phải field injection) để dễ test.
- Tất cả có default value phù hợp — không được crash khi không có key trong properties.
- Viết unit test không cần Spring context (không `@SpringBootTest`) bằng cách gọi constructor trực tiếp với giá trị test.
- Viết thêm một integration test `@SpringBootTest` với `@TestPropertySource` override hai property bất kỳ và verify.

---

## 13. Interview

**Core Q&A:**

Q: `@Value` hoạt động như thế nào bên dưới?
A: `@Value` được xử lý bởi `AutowiredAnnotationBeanPostProcessor` sau khi bean được khởi tạo. Post-processor đọc annotation, resolve placeholder `${}` qua `PropertySourcesPlaceholderConfigurer` — tra cứu trong `Environment` (tổng hợp từ tất cả property source theo thứ tự ưu tiên), rồi convert String sang kiểu đích qua `ConversionService`. SpEL `#{}` được evaluate bởi `ExpressionParser`. Kết quả được inject vào field/parameter trước khi bean được sử dụng.

Q: Thứ tự ưu tiên của property source trong Spring Boot là gì?
A: Từ cao đến thấp: command-line arguments, OS environment variables, `application-{profile}.yml`, `application.yml`, default value trong `@Value`. Environment variable luôn override file config — đây là cơ chế để override config khi deploy container mà không cần build lại image.

Q: Tại sao không nên dùng `@Value` trên `static` field?
A: `@Value` được inject bởi `BeanPostProcessor` vào instance của bean sau khi tạo. `static` field thuộc về class, không phải instance — post-processor không đụng đến. Field luôn là `null` hoặc giá trị mặc định của type. Không có error hay warning khi khởi động — là silent failure nguy hiểm.

Q: Khi nào dùng `@Value` và khi nào dùng `@ConfigurationProperties`?
A: `@Value` cho một hoặc hai property đơn lẻ, không liên quan nhóm, hoặc cần SpEL expression. `@ConfigurationProperties` cho nhóm ba property trở lên có cùng prefix, cần validation khi startup, cần IDE autocomplete, hoặc cần nested structure. Về lâu dài, `@ConfigurationProperties` dễ maintain hơn vì tập trung config vào một class và có type-safety.

Q: `${}` và `#{}` trong `@Value` khác nhau thế nào?
A: `${}` là property placeholder — đọc giá trị từ Spring `Environment` (properties file, env var, system properties). `#{}` là SpEL (Spring Expression Language) — evaluate Java expression tại runtime: tính toán, gọi method, truy cập Spring bean, đọc system property. Hai cú pháp có thể kết hợp: `@Value("#{'${app.name}'.toUpperCase()}")` — đọc property rồi gọi SpEL lên kết quả.

**Scenarios:**

Q: `@Value("${app.timeout}")` throw `IllegalArgumentException: Could not resolve placeholder` khi deploy lên staging nhưng local chạy tốt. Nguyên nhân và fix?
A: Nguyên nhân: property `app.timeout` tồn tại trong `application.properties` local nhưng không có trong staging environment (hoặc staging dùng `application-staging.properties` không có key này). Fix: (1) Thêm default value: `@Value("${app.timeout:5000}")` để không crash. (2) Đảm bảo key tồn tại trong tất cả môi trường — thêm vào `application.properties` gốc hoặc inject qua environment variable trên staging server.

Q: Bạn cần inject một List<String> từ property `app.cors.allowed-origins=http://localhost:3000,https://example.com`. `@Value` xử lý thế nào?
A: Spring Boot tự động split String bằng dấu phẩy khi target type là `List<String>` hoặc `String[]`: `@Value("${app.cors.allowed-origins}") List<String> allowedOrigins`. Kết quả là list `["http://localhost:3000", "https://example.com"]`. Lưu ý: nếu URL có dấu phẩy thì cần escape hoặc dùng YAML list syntax với `@ConfigurationProperties` thay thế.

---

## 14. References

- Spring Framework docs — @Value annotation: https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#beans-value-annotations
- Spring Boot docs — Externalized Configuration: https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config
- Spring Expression Language (SpEL) docs: https://docs.spring.io/spring-framework/docs/current/reference/html/core.html#expressions
- Spring Boot — Property Source Priority: https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config
- Baeldung — Spring @Value annotation: https://www.baeldung.com/spring-value-annotation

---

## 15. Real-world Code

- Spring Framework source `AutowiredAnnotationBeanPostProcessor` — hiểu cơ chế inject `@Value`: https://github.com/spring-projects/spring-framework/blob/main/spring-beans/src/main/java/org/springframework/beans/factory/annotation/AutowiredAnnotationBeanPostProcessor.java
- Spring Boot Autoconfigure — nhiều ví dụ `@Value` trong autoconfiguration classes: https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-autoconfigure
- Spring PetClinic — ví dụ inject property đơn giản trong project thực tế: https://github.com/spring-projects/spring-petclinic

---

## 16. Community

- Reddit: r/SpringBoot — tìm "@Value vs ConfigurationProperties"
- Stack Overflow tag: `spring-value`: https://stackoverflow.com/questions/tagged/spring-value
- Blog: Baeldung — "Spring @Value Annotation Guide": https://www.baeldung.com/spring-value-annotation
- Blog: reflectoring.io — "Configuring a Spring Boot Application": https://reflectoring.io/spring-boot-configuration-properties/
- Blog: Mkyong — "@Value examples in Spring Boot": https://mkyong.com/spring-boot/spring-boot-value-default-value/
- Stack Overflow — "Difference between @Value and @ConfigurationProperties": https://stackoverflow.com/questions/24392110/difference-between-value-and-configurationproperties-in-spring
