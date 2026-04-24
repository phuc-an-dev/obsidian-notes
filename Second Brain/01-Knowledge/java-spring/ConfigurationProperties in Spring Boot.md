---
created: 2026-04-20
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/configuration"
related:
  - "[[Validated in Spring Boot]]"
  - "[[spring-cloud-starter-aws]]"
---

# @ConfigurationProperties in Spring Boot

## 1. What

`@ConfigurationProperties(prefix = "...")` là annotation của Spring Boot dùng để **bind toàn bộ một nhóm config properties** từ `application.properties` / `application.yml` vào một Java class hoặc record theo kiểu type-safe. Thay vì inject từng property riêng lẻ bằng `@Value("${app.timeout}")`, bạn khai báo một class với các field tương ứng và Spring Boot tự động map tất cả property có cùng prefix vào class đó. Kể từ Spring Boot 2.2+, hỗ trợ binding vào `record` (immutable) thay vì chỉ mutable class.

---

## 2. Why

Khi ứng dụng có nhiều config liên quan đến nhau (ví dụ: database pool có 10 property, mail server có 8 property, rate limiter có 5 property), dùng `@Value` dẫn đến:

- Mỗi property phải inject riêng một field — 10 property = 10 `@Value` field rải rác.
- Không có validation kiểu type-safe tại compile time: `@Value("${app.timeout}")` luôn nhận `String`, phải tự parse sang `int` hay `Duration`.
- Refactoring tên property nguy hiểm: đổi tên trong `.properties` mà quên đổi trong `@Value` string — chỉ phát hiện lúc runtime.
- Không có IDE autocomplete cho property name khi viết `@Value`.
- Khó test: phải mock `Environment` hoặc dùng `@TestPropertySource` với từng property riêng lẻ.

`@ConfigurationProperties` giải quyết:
- Nhóm tất cả property liên quan vào một class — dễ đọc, dễ tìm.
- Spring Boot tự convert type: `String -> Integer`, `String -> Duration`, `String -> List`, `String -> Map`.
- Kết hợp với `@Validated` để validate constraint (`@NotNull`, `@Min`, `@Max`) khi application khởi động — fail-fast thay vì lỗi lúc runtime.
- IDE (IntelliJ) sinh autocomplete và documentation hint cho property name khi thêm `spring-boot-configuration-processor`.

---

## 3. Mental Model

Hãy tưởng tượng bạn đang điền **mẫu đơn nhập học** cho sinh viên:

`@Value` giống như điền từng ô riêng lẻ: nhân viên phòng đào tạo hỏi "họ tên?", bạn trả lời. Rồi hỏi "ngày sinh?", bạn trả lời. Rồi hỏi "mã sinh viên?", bạn trả lời. 20 câu hỏi = 20 lần hỏi-đáp riêng lẻ.

`@ConfigurationProperties` giống như đưa cả **tờ mẫu đơn đã điền đầy đủ** cho phòng đào tạo. Phòng đào tạo có template biết trước: ô nào tương ứng với field nào, kiểu dữ liệu nào, ô nào bắt buộc. Họ tự đọc và map toàn bộ tờ đơn một lần, báo lỗi ngay nếu ô bắt buộc để trống hay điền sai kiểu — không đợi đến khi xử lý hồ sơ mới phát hiện.

---

## 4. Where it fits

```
application.yml
┌─────────────────────────────┐
│ app:                        │
│   mail:                     │
│     host: smtp.example.com  │
│     port: 587               │
│     timeout: 5s             │
│     from: noreply@example   │
└─────────────────────────────┘
           |
           | Spring Boot Binder
           v
┌──────────────────────────────────────┐
│ @ConfigurationProperties("app.mail") │
│ record MailProperties(               │
│   String host,                       │
│   int port,                          │
│   Duration timeout,                  │
│   String from                        │
│ ) {}                                 │
└──────────────────────────────────────┘
           |
           | @Autowired / constructor injection
           v
┌────────────────────┐
│ MailService        │
│ (uses properties)  │
└────────────────────┘
```

`@ConfigurationProperties` nằm ở tầng infrastructure config, không chứa business logic, chỉ là data holder cho configuration.

---

## 5. When to use

- Nhóm từ 3 property trở lên có cùng prefix và logic liên quan với nhau.
- Cần **type-safe binding**: `Duration`, `DataSize`, `List<String>`, `Map<String, String>`, enum.
- Cần **validation khi startup**: `@NotBlank host`, `@Min(1) @Max(65535) port` — fail-fast trước khi ứng dụng nhận traffic.
- Config của một external integration: database pool, mail server, S3 bucket, payment gateway — mỗi integration có một `@ConfigurationProperties` class riêng.
- Muốn IDE autocomplete và documentation hint khi viết `application.properties` (cần `spring-boot-configuration-processor`).
- Muốn dễ test: inject `@ConfigurationProperties` class vào unit test trực tiếp, không phụ thuộc `Environment`.

---

## 6. When NOT to use

- **Một property đơn lẻ không liên quan nhóm**: `@Value("${server.port}")` gọn hơn, không cần tạo cả class.
- **Property thay đổi lúc runtime** (dynamic config): `@ConfigurationProperties` bind một lần khi khởi động. Nếu cần hot-reload config, cần kết hợp với Spring Cloud Config + `@RefreshScope`, hoặc dùng mechanism khác (database-backed config).
- **Secret cần encrypt**: không nên để secret dạng plaintext trong `application.properties`. Dùng AWS Secrets Manager, HashiCorp Vault, hay Kubernetes Secret — inject qua environment variable, không qua `@ConfigurationProperties` file tĩnh.
- **Config logic phức tạp**: nếu property cần transform, merge, hoặc compute phức tạp từ nhiều nguồn, dùng `@Bean` method trong `@Configuration` class để có full control.

---

## 7. Trade-offs

| Ưu điểm | Nhược điểm |
|---------|-----------|
| Nhóm config liên quan — dễ đọc, dễ tìm | Cần tạo thêm class/record — overhead nhỏ so với `@Value` |
| Type conversion tự động (Duration, DataSize, List, Map) | Binding error chỉ phát hiện lúc startup — không phải compile time |
| Validation khi khởi động với `@Validated` — fail-fast | Cần `@EnableConfigurationProperties` hoặc `@ConfigurationPropertiesScan` nếu không dùng `@Component` |
| IDE autocomplete với `spring-boot-configuration-processor` | Cần dependency processor thêm vào build |
| Dễ test: inject class trực tiếp, không cần mock `Environment` | Immutable record binding yêu cầu Spring Boot 2.2+ và constructor binding |
| Tên property relaxed binding: `camelCase`, `kebab-case`, `UPPER_CASE` đều bind được | Khó trace khi binding fail — error message đôi khi không rõ ràng |

---

## 8. Alternatives

| Giải pháp | Ưu điểm | Nhược điểm | Khi nào chọn |
|-----------|---------|-----------|-------------|
| `@Value("${...}")` | Đơn giản, ít boilerplate cho 1-2 property | Không type-safe, khó refactor, không validate khi startup | Property đơn lẻ, không thuộc nhóm |
| `Environment.getProperty()` | Programmatic, linh hoạt | Verbose, không type-safe, khó test | Dynamic property lookup lúc runtime |
| Spring Cloud Config + `@RefreshScope` | Hỗ trợ hot-reload config | Cần infrastructure thêm (Config Server) | Dynamic config trong microservices |
| Externalized config via env var + `@Value` | Đơn giản cho container/k8s | Không group được, khó validate | 12-factor app với ít config |

---

## 9. How

**Dependency (optional — chỉ cần cho IDE autocomplete):**

```xml
<dependency>
  <groupId>org.springframework.boot</groupId>
  <artifactId>spring-boot-configuration-processor</artifactId>
  <optional>true</optional>
</dependency>
```

**Cách 1 — Dùng record (Spring Boot 2.6+, khuyến nghị):**

```yaml
# application.yml
app:
  mail:
    host: smtp.example.com
    port: 587
    timeout: 5s
    username: noreply@example.com
    retry-count: 3
    allowed-domains:
      - example.com
      - example.org
```

```java
@ConfigurationProperties(prefix = "app.mail")
public record MailProperties(
    @NotBlank String host,
    @Min(1) @Max(65535) int port,
    @NotNull Duration timeout,
    String username,
    @Min(0) @Max(10) int retryCount,
    List<String> allowedDomains
) {}
```

```java
// Đăng ký trong main class hoặc @Configuration
@SpringBootApplication
@ConfigurationPropertiesScan   // scan tất cả @ConfigurationProperties trong package
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

**Cách 2 — Dùng mutable class với `@Component`:**

```java
@Component
@ConfigurationProperties(prefix = "app.mail")
@Validated
public class MailProperties {

    @NotBlank
    private String host;

    @Min(1) @Max(65535)
    private int port = 587;  // default value

    @NotNull
    private Duration timeout = Duration.ofSeconds(5);

    // Getter và Setter bắt buộc với mutable class
    public String getHost() { return host; }
    public void setHost(String host) { this.host = host; }
    public int getPort() { return port; }
    public void setPort(int port) { this.port = port; }
    // ...
}
```

**Cách 3 — Dùng `@Bean` + `@ConfigurationProperties` (không cần `@Component`):**

```java
@Configuration
public class AppConfig {

    @Bean
    @ConfigurationProperties(prefix = "app.mail")
    public MailProperties mailProperties() {
        return new MailProperties();
    }
}
```

**Inject vào service:**

```java
@Service
@RequiredArgsConstructor
public class MailService {

    private final MailProperties mailProperties;

    public void sendWelcomeEmail(String recipient) {
        log.info("Sending via {}:{} with timeout {}",
            mailProperties.host(),
            mailProperties.port(),
            mailProperties.timeout());
        // gửi mail...
    }
}
```

**Nested properties:**

```yaml
app:
  database:
    primary:
      url: jdbc:postgresql://localhost:5432/main
      pool:
        max-size: 20
        min-idle: 5
        connection-timeout: 30s
    readonly:
      url: jdbc:postgresql://replica:5432/main
      pool:
        max-size: 10
        min-idle: 2
```

```java
@ConfigurationProperties(prefix = "app.database")
public record DatabaseProperties(
    DataSourceConfig primary,
    DataSourceConfig readonly
) {
    public record DataSourceConfig(
        String url,
        PoolConfig pool
    ) {}

    public record PoolConfig(
        int maxSize,
        int minIdle,
        Duration connectionTimeout
    ) {}
}
```

**Unit test:**

```java
@SpringBootTest
@TestPropertySource(properties = {
    "app.mail.host=test-smtp.example.com",
    "app.mail.port=587",
    "app.mail.timeout=10s",
    "app.mail.username=test@example.com",
    "app.mail.retry-count=2"
})
class MailPropertiesTest {

    @Autowired
    private MailProperties mailProperties;

    @Test
    void shouldBindPropertiesCorrectly() {
        assertThat(mailProperties.host()).isEqualTo("test-smtp.example.com");
        assertThat(mailProperties.port()).isEqualTo(587);
        assertThat(mailProperties.timeout()).isEqualTo(Duration.ofSeconds(10));
        assertThat(mailProperties.retryCount()).isEqualTo(2);
    }
}
```

---

## 10. Production concerns

**Validation khi startup:**
Kết hợp `@Validated` với `@ConfigurationProperties` để validate toàn bộ config khi ứng dụng khởi động. Nếu config thiếu hoặc sai, ứng dụng fail ngay với message rõ ràng — tốt hơn nhiều so với `NullPointerException` lúc request đầu tiên tới.

```java
@ConfigurationProperties(prefix = "app.payment")
@Validated
public record PaymentProperties(
    @NotBlank String apiKey,
    @NotNull @Positive Duration requestTimeout,
    @Min(1) @Max(5) int maxRetries
) {}
```

**Secret management:**
Không hardcode secret trong `application.properties` commit vào git. Pattern khuyến nghị:
- Inject qua environment variable: `app.payment.api-key=${PAYMENT_API_KEY}`.
- Dùng AWS Secrets Manager / Parameter Store (với Spring Cloud AWS).
- Kubernetes Secret mount vào environment variable.

**Relaxed binding:**
Spring Boot tự động match các variant tên property:
- `app.mail.retry-count` (kebab-case, khuyến nghị trong `.yml`)
- `app.mail.retryCount` (camelCase)
- `APP_MAIL_RETRY_COUNT` (UPPER_SNAKE_CASE, dùng cho env var)

Tất cả đều bind vào field `retryCount`. Không cần nhất quán giữa `.yml` và Java field name.

**Configuration metadata (IDE support):**
Thêm `spring-boot-configuration-processor` vào build để sinh file `META-INF/spring-configuration-metadata.json`. IntelliJ và VS Code dùng file này để:
- Autocomplete tên property khi viết `application.yml`.
- Hiển thị type và description cho từng property.
- Cảnh báo property không tồn tại.

**Nhiều profile:**

```yaml
# application-dev.yml
app:
  mail:
    host: localhost
    port: 1025  # MailHog local

# application-prod.yml
app:
  mail:
    host: smtp.sendgrid.net
    port: 465
```

Spring Boot tự merge và override theo profile active. `@ConfigurationProperties` class nhận giá trị đúng theo profile.

---

## 11. Common mistakes

**Lỗi 1: Dùng record nhưng quên đăng ký bean**

```java
@ConfigurationProperties(prefix = "app.mail")
public record MailProperties(String host, int port) {}
// Không có @Component, không có @ConfigurationPropertiesScan
// -> NoSuchBeanDefinitionException khi inject
```

Fix: Thêm `@ConfigurationPropertiesScan` vào main class, hoặc thêm `@EnableConfigurationProperties(MailProperties.class)` vào `@Configuration` class:

```java
@SpringBootApplication
@ConfigurationPropertiesScan
public class Application { ... }
```

**Lỗi 2: Dùng mutable class nhưng quên Setter**

```java
@ConfigurationProperties(prefix = "app.mail")
public class MailProperties {
    private String host;
    // Quên setter!
    public String getHost() { return host; }
}
// host luôn là null vì Spring không thể set giá trị
```

Fix: Với mutable class, Spring Boot dùng setter-based binding — phải có setter cho mọi field cần bind. Hoặc chuyển sang record để dùng constructor binding.

**Lỗi 3: Đặt secret trực tiếp trong `application.properties` và commit vào git**

```properties
# SAI — secret lộ vào git history
app.payment.api-key=sk_live_abc123supersecret
```

Fix: Dùng environment variable placeholder và inject từ ngoài:

```properties
# ĐÚNG — chỉ lưu template, không lưu giá trị thật
app.payment.api-key=${PAYMENT_API_KEY}
```

**Lỗi 4: Thiếu `@Validated` dẫn đến lỗi chỉ phát hiện lúc runtime**

```java
@ConfigurationProperties(prefix = "app.mail")
public record MailProperties(
    @NotBlank String host,  // constraint có nhưng không được kiểm tra
    int port
) {}
// host có thể là null hoặc blank, chỉ biết khi MailService thực sự dùng
```

Fix: Thêm `@Validated` vào class hoặc khai báo riêng:

```java
@ConfigurationProperties(prefix = "app.mail")
@Validated
public record MailProperties(@NotBlank String host, int port) {}
```

---

## 12. Sample project

**Bài tập: Multi-integration config cho ứng dụng e-commerce**

Tạo `@ConfigurationProperties` cho ba integration riêng biệt:

1. `app.payment` — `apiKey`, `baseUrl`, `requestTimeout` (Duration), `maxRetries` (int 1–5).
2. `app.storage` — `bucketName`, `region`, `presignedUrlTtl` (Duration), `maxFileSizeMb` (DataSize).
3. `app.notification.email` — `host`, `port`, `from`, `useTls` (boolean).

Hard constraints:
- Tất cả dùng record (không phải mutable class).
- Tất cả có `@Validated` với ít nhất hai constraint mỗi class.
- Viết `application.yml` với giá trị hợp lệ cho cả ba.
- Viết một `@SpringBootTest` slice (`@ConfigurationPropertiesTest`) verify binding đúng cho từng class.
- `apiKey` trong payment phải inject từ environment variable, không hardcode trong `.yml`.

---

## 13. Interview

**Core Q&A:**

Q: `@ConfigurationProperties` khác `@Value` như thế nào?
A: `@Value` inject từng property đơn lẻ qua string key — không type-safe, không group được, khó refactor. `@ConfigurationProperties` bind cả nhóm property có cùng prefix vào một class — type-safe (auto convert Duration, List, Map), có thể validate khi startup với `@Validated`, dễ test hơn, và IDE hỗ trợ autocomplete. Dùng `@Value` cho một property đơn lẻ, `@ConfigurationProperties` cho nhóm config liên quan.

Q: Relaxed binding là gì?
A: Spring Boot tự động match nhiều variant tên property vào cùng một Java field. Field `retryCount` sẽ nhận giá trị từ `retry-count` (kebab-case trong yml), `retryCount` (camelCase), hay `RETRY_COUNT` (env var). Không cần nhất quán tên giữa `.yml` và Java field — Spring tự normalize.

Q: Khi nào dùng record, khi nào dùng mutable class với `@ConfigurationProperties`?
A: Với Spring Boot 2.6+ và Java 16+, ưu tiên record — immutable, gọn hơn, không cần setter. Dùng mutable class khi: cần default value phức tạp (khó đặt default trong record constructor), cần Spring Boot 2.4 trở xuống (constructor binding record chưa stable), hoặc framework đang dùng yêu cầu JavaBean convention.

Q: Làm thế nào để `@ConfigurationProperties` validate config khi startup?
A: Thêm `@Validated` vào class `@ConfigurationProperties` và gắn constraint annotation (`@NotBlank`, `@NotNull`, `@Min`, `@Max`) lên từng field hoặc record component. Spring Boot chạy Bean Validation khi bind config — nếu fail, ứng dụng không start và in ra error message rõ ràng với tên property vi phạm.

Q: `@ConfigurationPropertiesScan` và `@EnableConfigurationProperties` khác nhau thế nào?
A: `@ConfigurationPropertiesScan` tự động scan và đăng ký tất cả class có `@ConfigurationProperties` trong package chỉ định (hoặc package của main class). `@EnableConfigurationProperties(Foo.class)` đăng ký chỉ định từng class cụ thể. `@ConfigurationPropertiesScan` tiện hơn khi có nhiều `@ConfigurationProperties` class. `@EnableConfigurationProperties` rõ ràng hơn khi muốn control chính xác class nào được đăng ký.

**Scenarios:**

Q: Application khởi động thành công nhưng `MailProperties.host()` trả về `null` dù đã khai báo trong `application.yml`. Debug thế nào?
A: Kiểm tra theo thứ tự: (1) Prefix trong `@ConfigurationProperties` có khớp chính xác với key trong yml không — sai dấu chấm hay typo. (2) Bean có được đăng ký không — thiếu `@ConfigurationPropertiesScan` hoặc `@EnableConfigurationProperties`. (3) Với mutable class: có setter cho field `host` không. (4) Profile: yml đang dùng có phải profile khác không. (5) Thêm `@PostConstruct` log `mailProperties.host()` để xác nhận giá trị thực tế nhận được.

Q: Cần đọc một `List<String>` từ config, ví dụ danh sách domain được phép. Viết yml và class thế nào?
A: Trong yml: `app.security.allowed-domains: [example.com, example.org]` hoặc dạng block list. Trong record: `List<String> allowedDomains`. Spring Boot tự convert YAML list sang `List<String>`. Với relaxed binding, `allowed-domains` trong yml map sang `allowedDomains` trong Java. Thêm `@NotEmpty` nếu list không được rỗng.

---

## 14. References

- Spring Boot docs — Externalized Configuration: https://docs.spring.io/spring-boot/docs/current/reference/html/application-properties.html
- Spring Boot docs — Type-safe Configuration Properties: https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config.typesafe-configuration-properties
- Spring Boot docs — Relaxed Binding: https://docs.spring.io/spring-boot/docs/current/reference/html/features.html#features.external-config.typesafe-configuration-properties.relaxed-binding
- Spring Boot Configuration Processor: https://docs.spring.io/spring-boot/docs/current/reference/html/configuration-metadata.html
- Baeldung — Guide to @ConfigurationProperties: https://www.baeldung.com/configuration-properties-in-spring-boot

---

## 15. Real-world Code

- Spring Boot Autoconfigure source — hàng trăm ví dụ `@ConfigurationProperties` thực tế từ team Spring: https://github.com/spring-projects/spring-boot/tree/main/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure
- Ví dụ: `DataSourceProperties`, `MailProperties`, `RedisProperties` trong spring-boot-autoconfigure — cách team Spring tổ chức config class production.
- Spring PetClinic — ví dụ đơn giản hơn với config properties: https://github.com/spring-projects/spring-petclinic

---

## 16. Community

- Reddit: r/SpringBoot — tìm "ConfigurationProperties best practices"
- Stack Overflow tag: `spring-boot-configuration`: https://stackoverflow.com/questions/tagged/spring-boot-configuration
- Blog: Baeldung — "@ConfigurationProperties in Spring Boot": https://www.baeldung.com/configuration-properties-in-spring-boot
- Blog: reflectoring.io — "Spring Boot @ConfigurationProperties": https://reflectoring.io/spring-boot-configuration-properties/
- Talk: Spring I/O — "Externalized Configuration Deep Dive": tìm trên YouTube Spring I/O channel
- Blog: Philip Riecks — "Testing @ConfigurationProperties in Spring Boot": https://rieckpil.de/testing-spring-boot-configuration-properties/
