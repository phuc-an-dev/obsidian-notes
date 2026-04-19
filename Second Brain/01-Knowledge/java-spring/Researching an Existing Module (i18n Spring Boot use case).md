---
created: 2026-04-17
tags:
  - "#type/tutorial"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/i18n"
related: "[[EventListener in Spring Boot]]"
---

# Researching an Existing Module (i18n Spring Boot use case)

## 1. What

Đây là một tutorial về **quy trình đọc hiểu và nghiên cứu một Spring Boot module có sẵn** — code mà bạn không tự viết, có thể từ developer khác, từ thư viện, hoặc từ phiên bản trước của bản thân. Sử dụng module i18n (internationalization) trong Spring Boot làm case study cụ thể. Kết quả mong đợi: sau khi áp dụng quy trình này, bạn có thể giải thích module đó hoạt động thế nào, biết chính xác sửa ở đâu khi có bug, và mở rộng nó mà không làm hỏng hành vi cũ.

## 2. Why

Đọc code người khác không có hướng dẫn thường dẫn đến những tình huống sau:

- Mở IDE, mở từng file theo alphabet, đọc xong vẫn không hiểu module làm gì tổng thể.
- Hiểu từng dòng code nhưng không hiểu **tại sao** module đó tồn tại và vấn đề gì nó giải quyết.
- Sửa bug nhưng sửa sai chỗ vì không biết entry point đâu là quan trọng.
- Sau 2 giờ research, đóng màn hình lại trắng tay — không có gì để lại.
- Phát sinh "assumption" sai (tưởng rằng module làm X nhưng thực ra làm Y) và chỉ phát hiện khi deploy.

Quy trình dưới đây là một khung có cấu trúc để biến "research lan man" thành "research có mục tiêu" — tiết kiệm thời gian, giảm rủi ro, tạo ra kiến thức tái sử dụng được.

## 3. Mental Model

Nghiên cứu module là giống như được bàn giao căn hộ trong tòa nhà mới mà bạn chưa bao giờ vào.

- **Đừng làm**: mở tất cả các tủ, phá tất cả các ổ cắm, đóng mở từng cửa sổ ngay khi bước vào.
- **Làm đúng**: đi từ ngoài vào trong. Nhìn tổng thể tòa nhà trước (mặt tiền — API, README, JIRA ticket). Xem sơ đồ tầng và vị trí căn hộ (architecture — module này nằm ở tầng nào, ai gọi nó). Rồi mới vào từng phòng với mục đích cụ thể (implementation — nơi chứa logic chính). Cuối cùng kiểm tra có thể thay cấu trúc chịu lực không trước khi sửa tường (viết smoke test trước khi refactor).

Tường chịu lực trong Spring Boot: các `@Configuration` class, các `@Bean` chính, các `@ConditionalOn*` điều kiện, các auto-configuration entry trong `spring.factories` hoặc `AutoConfiguration.imports`.

## 4. Where It Fits

```
[JIRA ticket / Bug report / Task description]
    |
    v
Bước 1: Xác định mục tiêu và phạm vi research
    |
    v
Bước 2: Tìm entry point (config, bean, annotation)
    |
    v
Bước 3: Vẽ data flow (text diagram, không cần tool)
    |
    v
Bước 4: Đọc implementation theo flow, ghi chép song song
    |
    v
Bước 5: Viết smoke test để verify
    |
    v
[Có thể extend / fix / refactor an toàn]
```

Module i18n nằm ở vị trí như sau trong kiến trúc Spring Boot:

```
HTTP Request (header: Accept-Language: vi)
    |
    v
LocaleChangeInterceptor (nếu cấu hình)  <-- entry point 1
    |
    v
LocaleResolver (resolve Locale từ request)  <-- entry point 2
    |
    v
Controller / @ExceptionHandler  <-- nơi gọi getMessage()
    |
    v
MessageSource  <-- core bean  <-- entry point 3
    |
    v
messages_vi.properties  <-- file chứa chuỗi dịch
    |
    v
Response body (tên / thông báo đã được dịch)
```

## 5. When to Use

- Được giao task liên quan đến một tính năng mình chưa bao giờ dùng hoặc chưa đọc code.
- Cần fix bug trong module người khác viết mà không có documentation.
- Onboarding vào project mới: phải hiểu module i18n, auth, caching, hoặc bất kỳ module infrastructure nào trước khi code.
- Cần extend hoạt động của một module (thêm locale mới, thêm loại source cho message) mà không được phá vỡ hành vi cũ.
- Bị mời vào code review cho một PR lớn — cần hiểu context trước khi review có chất lượng.

## 6. When NOT to Use

- Module quá nhỏ (1 file, < 100 dòng) — đọc thẳng, không cần quy trình 5 bước.
- Bạn tự viết module này trong 6 tháng trước và vẫn nhớ rõ — phí thời gian.
- Đang trong bug-critical hotfix — quy trình này dành cho "hiểu đúng", không phải "sửa nhanh". Trong hotfix, trace stack trace là ưu tiên số 1.

## 7. Trade-offs

| Pros | Cons |
|---|---|
| Hiểu đúng mục đích trước khi đọc code — tránh hiểu lầm | Tốn thêm 30-60 phút setup trước khi viết code |
| Có thể giải thích lại module cho người khác sau research | Cần phạm vi research rõ ràng — nếu không, dễ rơi vào "rabbit hole" |
| Giảm rủi ro break existing behavior khi sửa | Các module không có documentation khó áp dụng bước 1 |
| Tạo ra Obsidian note tái sử dụng được cho lần sau | Quy trình chỉ work tốt nếu có test coverage để verify |

## 8. Alternatives

| Approach | Khi nào chọn |
|---|---|
| Đọc code top-down từ đầu file | Module < 3 file, ít dependency |
| Hỏi trực tiếp tác giả | Tác giả còn trong team, sẵn sàng review cùng |
| Đọc test trước (test-first reading) | Project có test coverage tốt (> 70%) |
| Đọc commit history (git log -p) | Cần hiểu tại sao code được viết thế này, không phải thế kia |
| **Quy trình 5 bước dưới đây** | Module lớn, nhiều file, ít hoặc không có tài liệu |

## 9. How: Quy trình 5 bước nghiên cứu i18n module

### Bước 1 — Xác định mục tiêu trước khi mở IDE

Trả lời 3 câu hỏi này TRƯỚC khi đọc bất kỳ dòng code nào:

```
1. Module này làm gì?
   Trả lời: i18n module dịch message/nội dung theo ngôn ngữ của người dùng.

2. Tại sao dự án cần nó?
   Trả lời: App hỗ trợ cả tiếng Việt và tiếng Anh — API phải trả về error message
   đúng ngôn ngữ trong Accept-Language header.

3. Task cụ thể của mình là gì?
   Trả lời: Fix bug — API luôn trả tiếng Anh dù client gửi Accept-Language: vi.
```

Nếu không trả lời được câu 1 và 2 — đọc JIRA ticket kỹ hơn, xem README, hoặc hỏi đồng nghiệp trước.

---

### Bước 2 — Tìm entry point (không đọc file theo alphabet)

**Với i18n trong Spring Boot**, có 3 entry point chính:

**Entry point A: Auto-configuration**

```
src/
  resources/
    META-INF/
      spring/
        org.springframework.boot.autoconfigure.AutoConfiguration.imports
          -> tìm dòng có chữ "MessageSource" hoặc "Locale"
```

Spring Boot tự động cấu hình `MessageSource` qua `MessageSourceAutoConfiguration` nếu có file `messages.properties` trong classpath. Tìm class này để hiểu default behavior:

```java
// Trong Spring Boot source (reference):
@Configuration(proxyBeanMethods = false)
@ConditionalOnMissingBean(name = AbstractApplicationContext.MESSAGE_SOURCE_BEAN_NAME,
                          search = SearchStrategy.CURRENT)
@AutoConfigureOrder(Ordered.HIGHEST_PRECEDENCE)
public class MessageSourceAutoConfiguration {

    @Bean
    public MessageSource messageSource(MessageSourceProperties properties) {
        ResourceBundleMessageSource messageSource = new ResourceBundleMessageSource();
        messageSource.setBasenames(StringUtils.toStringArray(properties.getBasename()));
        messageSource.setDefaultEncoding(properties.getEncoding().name());
        messageSource.setFallbackToSystemLocale(properties.isFallbackToSystemLocale());
        return messageSource;
    }
}
```

**Entry point B: Custom config class**

Sử dụng IntelliJ để tìm:
```
1. Ctrl+Shift+F (Find in Files)
2. Search: "MessageSource" hoặc "LocaleResolver"
3. Filter: file type *.java
4. Tìm class có annotation @Configuration hoặc @Bean
```

**Entry point C: application.yml**

```yaml
# Tìm các key này trong application.yml hoặc application.properties:
spring:
  messages:
    basename: messages          # tên file .properties
    encoding: UTF-8
    fallback-to-system-locale: false
    cache-duration: 3600        # cache thời gian, -1 là không cache
```

**Checklist tìm entry point i18n:**

- [ ] Có class `@Configuration` định nghĩa bean `MessageSource` không?
- [ ] Có bean `LocaleResolver` không? (nếu không có, Spring dùng `AcceptHeaderLocaleResolver` mặc định)
- [ ] Có `LocaleChangeInterceptor` được đăng ký vào `WebMvcConfigurer.addInterceptors()` không?
- [ ] File `messages.properties`, `messages_vi.properties`, `messages_en.properties` nằm ở đâu?
- [ ] `spring.messages.basename` trong config là gì?

---

### Bước 3 — Vẽ data flow bằng tay

Sau khi tìm entry points, trace luồng xử lý từ request đến response. KHÔNG dùng tool — vẽ bằng text ở Obsidian:

**Flow i18n:**

```
Client gửi request:
  GET /api/users/99
  Header: Accept-Language: vi

Step 1: LocaleChangeInterceptor.preHandle()
  - Đọc param "lang" từ query string (nếu có: /api/users/99?lang=vi)
  - Gọi localeResolver.setLocale(request, response, locale)
  - Nếu không có LocaleChangeInterceptor -> skip step này

Step 2: LocaleResolver.resolveLocale(request)
  - AcceptHeaderLocaleResolver: đọc header "Accept-Language: vi"
  - -> trả về Locale.forLanguageTag("vi") = Locale("vi")
  - SessionLocaleResolver: đọc từ session, fallback về Accept-Language
  - CookieLocaleResolver: đọc từ cookie "locale"

Step 3: LocaleContextHolder.getLocale()
  - Spring MVC lưu Locale vào ThreadLocal sau khi resolve
  - Bất kỳ code nào gọi LocaleContextHolder.getLocale() sẽ lấy được "vi"

Step 4: messageSource.getMessage("error.userNotFound", null, locale)
  - MessageSource đọc file messages_vi.properties
  - Tìm key "error.userNotFound"
  - Trả về: "Người dùng không tồn tại"
  - Nếu không tìm thấy -> fallback về messages.properties (tiếng Anh)
  - Nếu không có fallback -> throw NoSuchMessageException

Step 5: Response
  {
    "message": "Người dùng không tồn tại"
  }
```

Ghi ra Obsidian hoặc giấy trước khi đọc implementation. Nếu step nào bạn không chắc — đánh dấu "?" để xác nhận sau.

---

### Bước 4 — Đọc implementation theo flow, ghi chép song song

Đọc theo flow đã vẽ, KHÔNG lan man sang các file không nằm trong flow.

**Các class cần đọc theo thứ tự:**

**A. Đọc `LocaleResolver` bean**

```java
// AcceptHeaderLocaleResolver — mặc định khi không define bean tùy chỉnh
public class AcceptHeaderLocaleResolver implements LocaleContextResolver {

    @Override
    public Locale resolveLocale(HttpServletRequest request) {
        // Đọc Accept-Language header
        List<Locale.LanguageRange> ranges = Locale.LanguageRange
            .parse(request.getHeader("Accept-Language"));
        List<Locale> supportedLocales = getSupportedLocales(); // từ config
        return Locale.lookup(ranges, supportedLocales);
    }
}

// SessionLocaleResolver — lưu locale vào HttpSession
// Phù hợp cho web app có login state
// Khi user thay đổi ngôn ngữ, locale được save vào session và giữ cho lần đăng nhập tiếp theo

// CookieLocaleResolver — lưu locale vào cookie
// Phù hợp cho single-page app (SPA) tương tác với REST API
```

**B. Đọc `MessageSource` và cách nó tìm file**

```java
// ResourceBundleMessageSource — đọc từ classpath
messageSource.setBasenames("messages", "validation-messages");
// Tìm theo thứ tự: messages_vi_VN.properties -> messages_vi.properties -> messages.properties

// ReloadableResourceBundleMessageSource — hỗ trợ reload mà không cần restart
messageSource.setBasenames("classpath:messages/messages");
messageSource.setCacheSeconds(60); // reload mỗi 60 giây — tốt cho dev

// Cách gọi:
String msg = messageSource.getMessage(
    "key.name",          // message key
    new Object[]{"param1", "param2"},  // tham số cho placeholder {0}, {1}
    locale               // ngôn ngữ cần
);
```

**C. Đọc `MessageSourceAccessor` (nếu có dùng)**

```java
// MessageSourceAccessor là wrapper tiện lợi cho MessageSource
// Nó tự động lấy Locale từ LocaleContextHolder
@Bean
public MessageSourceAccessor messageSourceAccessor(MessageSource messageSource) {
    return new MessageSourceAccessor(messageSource);
}

// Sử dụng:
String msg = accessor.getMessage("key.name");  // Locale lấy từ ThreadLocal tự động
```

**D. Đọc nơi gọi getMessage() trong code dự án**

Dùng IntelliJ:
```
1. Ctrl+Shift+F -> search "getMessage("
2. Hoặc search "messageSource.getMessage"
3. Hoặc search ".getMessage(" trong package nội dung exception handler
```

Thường gặp trong:
```java
@ExceptionHandler(UserNotFoundException.class)
public ResponseEntity<ErrorResponse> handleUserNotFound(
        UserNotFoundException ex, Locale locale) {
    String message = messageSource.getMessage(
        "error.userNotFound",
        new Object[]{ex.getUserId()},
        locale
    );
    return ResponseEntity.status(404).body(new ErrorResponse(message));
}
```

**Lưu ý quan trọng:** `locale` parameter trong `@ExceptionHandler` được Spring inject tự động từ request — không cần lấy từ `LocaleContextHolder` thủ công.

**E. IntelliJ navigation tips**

| Thao tác | Phím tắt |
|---|---|
| Nhảy đến khai báo / implementation | Ctrl+Click hoặc Ctrl+B |
| Tìm tất cả nơi dùng (Usages) | Alt+F7 |
| Xem Structure (methods, fields) | Ctrl+F12 |
| Tìm file theo tên | Ctrl+Shift+N |
| Tìm class theo tên | Ctrl+N |
| Tìm symbol (method, variable) trong project | Ctrl+Shift+Alt+N |
| Xem hierarchy của class | Ctrl+H |
| Tìm tất cả config annotation | Ctrl+Shift+F + filter @Configuration |

---

### Bước 5 — Viết smoke test để verify hiểu đúng

Sau khi đọc, viết một test để xác nhận bạn hiểu đúng:

```java
// I18nSmokeTest.java
@SpringBootTest
class I18nSmokeTest {

    @Autowired
    private MessageSource messageSource;

    @Autowired
    private MockMvc mockMvc;

    // Test 1: MessageSource trả đúng chuỗi Việt
    @Test
    void shouldReturnVietnameseMessageForViLocale() {
        String msg = messageSource.getMessage(
            "error.userNotFound",
            new Object[]{42L},
            new Locale("vi")
        );
        assertThat(msg).isNotBlank();
        // Nếu biết giá trị cụ thể:
        assertThat(msg).contains("42"); // placeholder được replace
    }

    // Test 2: Kiểm tra API trả về message đúng ngôn ngữ
    @Test
    void apiShouldReturnVietnamesMessageWhenHeaderIsVi() throws Exception {
        mockMvc.perform(
            get("/api/users/99999")
                .header("Accept-Language", "vi")
        )
        .andExpect(status().isNotFound())
        .andExpect(jsonPath("$.message").isNotEmpty());
        // Nếu biết giá trị cụ thể: .andExpect(jsonPath("$.message").value("..."));
    }

    // Test 3: Fallback sang English khi không có Vietnamese key
    @Test
    void shouldFallbackToEnglishWhenVietnameseKeyMissing() {
        String msg = messageSource.getMessage(
            "some.key.only.in.english",
            null,
            new Locale("vi")
        );
        // Nên không throw exception nếu cấu hình fallback đúng
        assertThat(msg).isNotBlank();
    }
}
```

**Nếu test fail:** Trace theo quy trình debug ở section 13.

---

### Bonus: Cách tìm auto-configuration

Khi bạn không biết Spring Boot cấu hình tính năng gì tự động:

```
1. Mở dependency jar của spring-boot-autoconfigure trong Maven/Gradle
2. Tìm file: META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
3. Ctrl+F tìm "Message" hoặc "Locale" hoặc tên tính năng bạn cần
4. Mở class tìm được (ví dụ: MessageSourceAutoConfiguration)
5. Đọc các @ConditionalOn* annotation để biết khi nào nó được kích hoạt
```

Hoặc dùng Spring Boot Actuator:
```
GET /actuator/conditions
```
Trả về danh sách tất cả auto-configuration và lý do nó được hoặc không được apply.

---

### Common i18n Gotchas khi đọc existing code

**Gotcha 1: Locale không được truyền vào getMessage()**

```java
// SAI: hardcode Locale.ENGLISH
messageSource.getMessage("key", null, Locale.ENGLISH);

// ĐÚNG: lấy từ request context
messageSource.getMessage("key", null, LocaleContextHolder.getLocale());
// Hoặc inject qua parameter của handler method:
public String handle(Locale locale) { ... }
```

**Gotcha 2: LocaleChangeInterceptor không được đăng ký**

```java
// Config có định nghĩa LocaleChangeInterceptor bean nhưng không add vào registry
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Bean
    public LocaleChangeInterceptor localeChangeInterceptor() {
        LocaleChangeInterceptor lci = new LocaleChangeInterceptor();
        lci.setParamName("lang");
        return lci;
    }

    // BUG: quên override addInterceptors()!
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(localeChangeInterceptor()); // dòng này bị thiếu
    }
}
```

**Gotcha 3: MessageSource bean bị override**

Nếu có nhiều `@Bean` method trả về `MessageSource`, Spring chỉ dùng cái được định nghĩa sau cùng (hoặc cái được ưu tiên theo `@Order` / `@Primary`). Tìm tất cả nơi định nghĩa `MessageSource` bean trong project.

**Gotcha 4: File .properties encoding sai**

```yaml
# application.yml
spring:
  messages:
    encoding: UTF-8  # Phải set rõ ràng, đặc biệt trên Windows
```

Nếu file `messages_vi.properties` chứa tiếng Việt dạng unicode escape (`\u0041`) thay vì UTF-8 thuần túy, kiểm tra encoding của file trong IDE.

**Gotcha 5: Basename bị cấu hình sai**

```yaml
# Nếu file là: src/main/resources/i18n/messages_vi.properties
# Thì basename phải là:
spring:
  messages:
    basename: i18n/messages  # KHÔNG phải "messages" hay "i18n/messages_vi"
```

## 10. Production Concerns

### Scaling

- Số lượng locale tăng thì số file `.properties` tăng — đặt trong thư mục riêng (`src/main/resources/i18n/`) cho gọn.
- `ResourceBundleMessageSource` cache message sau lần load đầu — phù hợp production. Nếu cần hot-reload trong dev, dùng `ReloadableResourceBundleMessageSource` với `setCacheSeconds(60)`.
- Với module phức tạp hơn, document data flow ra file wiki hoặc README trước khi merge để người tiếp theo không mất công research lại.

### Failure

- Hiểu sai entry point dẫn đến fix sai chỗ — bug vẫn còn, mất thêm thời gian debug.
- Không viết smoke test sau research dẫn đến "tưởng mình hiểu" nhưng thực tế assumption sai.
- Bỏ qua `@ConditionalOn*` annotation dẫn đến không hiểu khi nào bean được tạo / không được tạo.

### Monitoring

- Sau research, tự đánh giá bằng câu hỏi: "Tôi có thể giải thích module này cho đồng nghiệp trong 5 phút không?"
- Nếu câu trả lời là "không" — chưa hiểu đủ, cần thêm bước hoặc hỏi người có kinh nghiệm.
- Lưu kết quả research vào Obsidian note ngay trong lúc research, không phải sau khi xong — chi tiết bị quên rất nhanh.

## 11. Common Mistakes

- Mistake: Mở IDE và bắt đầu đọc code ngay khi nhận task, không xác định mục tiêu trước.
  Fix: Dành 5 phút trả lời "Module này làm gì? Task của mình là gì?" trước khi mở bất kỳ file nào. Nếu không trả lời được, đọc JIRA ticket kỹ hơn hoặc hỏi đồng nghiệp.

- Mistake: Đọc toàn bộ package từ đầu đến cuối theo thứ tự alphabet — đọc xong không biết mối quan hệ giữa các file.
  Fix: Tìm entry point trước (config class, bean, annotation), sau đó trace theo data flow. Chỉ đọc các file nằm trong flow, bỏ qua file không liên quan đến task.

- Mistake: Tin rằng hiểu đúng chỉ qua việc đọc code mà không verify.
  Fix: Viết một smoke test đơn giản (có thể 5-10 dòng) để confirm behavior. Nếu test pass — hiểu đúng. Nếu fail — quay lại bước 2.

- Mistake: Không ghi chép trong lúc research — sau 2 giờ đóng màn hình, kiến thức bay đi.
  Fix: Mở Obsidian note (chính note này) và điền vào section 9 trong lúc research. Note được viết "trong nóng" sẽ chi tiết và chính xác hơn note viết lại sau.

## 12. Sample Project

**Dự án:** tamice backend — nghiên cứu module i18n theo quy trình 5 bước.

**Constraint cứng:**
- Không được hỏi bất kỳ đồng nghiệp nào trong quá trình research.
- Chỉ được dùng: code trong repo, git log, JIRA ticket, Spring official docs.
- Thời gian tối đa: 2 giờ cho toàn bộ quy trình.

**Output bắt buộc:**
1. Text diagram data flow (như ví dụ ở Bước 3) chính xác với project thực tế.
2. Danh sách đầy đủ các entry point (class tên, file tên, annotation).
3. Giải thích sự khác biệt giữa `LocaleResolver` đang được dùng và 2 loại khác.
4. Smoke test chạy được: kiểm tra API trả về message tiếng Việt khi `Accept-Language: vi`.
5. Điền đầy đủ Section 9 của note này với thông tin từ dự án thực tế.

**Điều kiện hoàn thành:** Tất cả 5 output trên phải xong trước khi được coi là "đã hiểu module i18n."

## 13. Interview

### Core Q&A

**Q: Khi được giao nghiên cứu một module lạ, bạn bắt đầu từ đâu?**
A: Bắt đầu bằng cách xác định mục tiêu: module này làm gì, tại sao dự án cần nó, task cụ thể của mình là gì. Sau đó tìm entry point — không phải đọc file theo thứ tự mà tìm `@Configuration` class, bean definition, hoặc annotation liên quan. Từ entry point, trace data flow từ request đến response và vẽ text diagram. Chỉ sau khi có diagram mới bắt đầu đọc implementation theo flow đó.

**Q: Làm sao để biết mình đã hiểu đúng module chứ không phải tự suy đoán?**
A: Viết smoke test xác nhận behavior. Nếu test pass, hiểu đúng. Cách khác là giải thích lại module bằng lời nói có cấu trúc (như buổi pair programming) — nếu không diễn đạt được, tức là chưa hiểu đủ.

**Q: i18n trong Spring Boot hoạt động thế nào ở mức cao?**
A: Request đến kèm `Accept-Language` header. `LocaleResolver` đọc header đó và trả về `Locale` object. Spring MVC lưu Locale vào `LocaleContextHolder` (ThreadLocal). Khi code gọi `messageSource.getMessage(key, params, locale)`, `MessageSource` tìm file `.properties` phù hợp (ví dụ `messages_vi.properties`), lấy chuỗi đã dịch, thay thế placeholder, và trả về kết quả.

**Q: `AcceptHeaderLocaleResolver` khác `SessionLocaleResolver` thế nào, dùng cái nào cho REST API?**
A: `AcceptHeaderLocaleResolver` đọc Locale từ HTTP header `Accept-Language` mỗi request — stateless, phù hợp REST API vì RESTful API nên stateless. `SessionLocaleResolver` lưu Locale trong HttpSession — stateful, phù hợp web app có login state và muốn user chỉ thay đổi ngôn ngữ một lần. Cho REST API, dùng `AcceptHeaderLocaleResolver` là chuẩn.

**Q: `@ConditionalOnMissingBean` trong MessageSourceAutoConfiguration có nghĩa là gì?**
A: Có nghĩa là Spring Boot chỉ tự động tạo bean `MessageSource` MẶC ĐỊNH nếu KHÔNG CÓ bean `MessageSource` nào khác được đăng ký trong context. Nếu dự án định nghĩa bean `MessageSource` tùy chỉnh trong `@Configuration` class, auto-configuration sẽ bị bỏ qua. Đây là cơ chế "override" của Spring Boot — framework từ chối cấu hình nếu bạn đã tự cấu hình.

**Q: Làm thế nào để tìm tất cả các điều kiện auto-configuration đang được áp dụng?**
A: Dùng Spring Boot Actuator: `GET /actuator/conditions` trả về toàn bộ các `@Conditional` evaluation — điều kiện nào match, điều kiện nào không match và tại sao. Trong production, dùng `/actuator/env` để xem tất cả các property đang hiệu lực.

### Scenario

**Scenario 1:** Bug report: "API trả về message tiếng Anh dù client gửi `Accept-Language: vi`." Quy trình debug của bạn là gì?

Trả lời: Trace theo data flow i18n từ đầu đến cuối: (1) Kiểm tra request có gửi đúng header `Accept-Language: vi` không — dùng browser devtools hoặc log request headers. (2) Kiểm tra `LocaleResolver` bean có được cấu hình không — nếu không, Spring dùng `AcceptHeaderLocaleResolver` mặc định đọc từ Accept-Language, phù hợp. (3) Kiểm tra `LocaleContextHolder.getLocale()` ở điểm gọi `getMessage()` — có đúng trả `vi` không (đặt breakpoint hoặc log). (4) Kiểm tra file `messages_vi.properties` có tồn tại với đúng path (khớp với `spring.messages.basename`) và có key đang cần không. (5) Kiểm tra encoding của file. Trace từng bước thay vì đoán mò.

**Scenario 2:** Bạn cần thêm ngôn ngữ Nhật Bản vào hệ thống i18n đã có. Sau khi đọc code theo quy trình 5 bước, bạn biết mình cần làm gì?

Trả lời: Sau research, biết rằng chỉ cần thêm file `messages_ja.properties` với các key giống `messages.properties` nhưng giá trị bằng tiếng Nhật. `MessageSource` tự động fallback về `messages.properties` cho các key chưa được dịch. Không cần thay đổi bất kỳ Java code nào. Nếu `LocaleResolver` có `setSupportedLocales()`, cần thêm `Locale.JAPANESE` vào danh sách. Đề xuất: smoke test với `Accept-Language: ja`.

**Scenario 3:** Bạn thấy trong code có cả `MessageSource` bean tự định nghĩa trong `@Configuration` và cấu hình `spring.messages.basename` trong `application.yml`. Cái nào được dùng?

Trả lời: Bean tự định nghĩa trong `@Configuration` được ưu tiên. `@ConditionalOnMissingBean` trong `MessageSourceAutoConfiguration` có nghĩa là nếu bạn đã có bean `MessageSource` tùy chỉnh, auto-configuration sẽ không tạo bean default nữa. `spring.messages.basename` trong `application.yml` chỉ ảnh hưởng đến bean auto-configured — nếu bạn tự tạo bean, bạn tự cấu hình basename trong bean đó (ví dụ: `setBasenames("messages")`). Có thể `spring.messages.*` property của bạn đang bị ignore vì auto-config bị disabled.

## 14. References

- Spring Boot Docs - Internationalization: https://docs.spring.io/spring-boot/reference/web/servlet.html#web.servlet.spring-mvc.message-codes
- Spring Framework Docs - LocaleResolver: https://docs.spring.io/spring-framework/reference/web/webmvc/mvc-servlet/localeresolver.html
- Spring Framework Docs - MessageSource: https://docs.spring.io/spring-framework/reference/core/beans/context-introduction.html#context-functionality-messagesource
- Spring Boot Docs - Auto-configuration: https://docs.spring.io/spring-boot/reference/using/auto-configuration.html
- Spring Boot Docs - @ConditionalOn* annotations: https://docs.spring.io/spring-boot/reference/features/developing-auto-configuration.html#features.developing-auto-configuration.condition-annotations
- IntelliJ IDEA Docs - Navigation: https://www.jetbrains.com/help/idea/navigating-through-the-source-code.html

## 15. Real-world Code

- Spring Boot MessageSourceAutoConfiguration source: https://github.com/spring-projects/spring-boot/blob/main/spring-boot-project/spring-boot-autoconfigure/src/main/java/org/springframework/boot/autoconfigure/context/MessageSourceAutoConfiguration.java
- Spring Framework AcceptHeaderLocaleResolver source: https://github.com/spring-projects/spring-framework/blob/main/spring-webmvc/src/main/java/org/springframework/web/servlet/i18n/AcceptHeaderLocaleResolver.java
- Spring Framework LocaleChangeInterceptor source: https://github.com/spring-projects/spring-framework/blob/main/spring-webmvc/src/main/java/org/springframework/web/servlet/i18n/LocaleChangeInterceptor.java
- Spring PetClinic (ví dụ sử dụng MessageSource): https://github.com/spring-projects/spring-petclinic
- Baeldung i18n example repo: https://github.com/eugenp/tutorials/tree/master/spring-boot-modules/spring-boot-mvc-3

## 16. Community

- Stack Overflow - "Spring Boot i18n not working": https://stackoverflow.com/questions/36531131/i18n-in-spring-boot-thymeleaf
- Stack Overflow - "AcceptHeaderLocaleResolver vs SessionLocaleResolver": https://stackoverflow.com/questions/10286293/spring-mvc-locale-change-without-locale-change-interceptor
- Baeldung - Spring MVC Internationalization: https://www.baeldung.com/spring-boot-internationalization
- Stack Overflow - "How to find Spring Boot auto-configuration": https://stackoverflow.com/questions/32747089/how-does-spring-boots-autoconfig-work
- Reddit r/SpringBoot - "How do you approach reading unfamiliar Spring code": https://www.reddit.com/r/SpringBoot/
