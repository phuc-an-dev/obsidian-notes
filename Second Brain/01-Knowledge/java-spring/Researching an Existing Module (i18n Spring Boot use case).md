---
created: 2026-04-15
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
related: "[[Tamice - Logging Refactor]]"
---
## 1. What

Best practice để nghiên cứu một module **trong dự án có sẵn** — không phải học từ zero, mà là _hiểu một codebase người khác đã viết_ rồi có thể extend/fix nó. Use case cụ thể: module i18n (internationalization) trong Spring Boot backend.

## 2. Why (problem it solves)

Đọc code người khác không có hướng dẫn thường dẫn đến:

- Đọc lan man, không biết entry point ở đâu
- Hiểu từng dòng nhưng không hiểu _tại sao_ module tồn tại
- Sửa code mà không biết mình đang break gì
- Mất nhiều giờ mà không capture được kiến thức để dùng lại

## 3. Mental Model

> Nghiên cứu module lạ giống như nhận bàn giao nhà cũ: **đừng đụng tường trước khi biết đâu là tường chịu lực**. Đi từ ngoài vào trong — mặt tiền (API/config), sơ đồ điện nước (data flow), rồi mới mở từng phòng (implementation).

## 4. Where it fits (architecture)

```
[Request] → Controller → Service → [Module cần nghiên cứu] → Response/DB
                                         ↑
                          Config / Bean / External dependency
```

Trước khi đọc code, cần biết module nằm ở tầng nào, ai gọi nó, và nó phụ thuộc vào gì.

## 5. When to use

- Được giao task liên quan đến một tính năng mình chưa đụng
- Cần fix bug trong code người khác viết
- Onboarding vào project mới hoặc module mới
- Cần extend module hiện có mà không được phép break behavior cũ

## 6. When NOT to use

- Đừng áp dụng quy trình này cho code **mình tự viết** — lãng phí thời gian
- Không cần thiết nếu module chỉ có 1 file < 100 dòng — đọc thẳng

## 7. Trade-offs

|Pros|Cons|
|---|---|
|Hiểu đúng mục đích trước khi đọc code|Tốn thêm 30–60 phút ban đầu|
|Có thể giải thích lại cho người khác|Cần hỏi người cũ hoặc tự tìm doc|
|Giảm risk break existing behavior|Không áp dụng được cho module zero-doc|
|Tạo ra note tái sử dụng được||

## 8. Alternatives (with comparison)

|Approach|Khi nào chọn|
|---|---|
|Đọc code top-down từ đầu file|Module nhỏ, < 3 file, không có dependency phức tạp|
|Hỏi trực tiếp tác giả|Tác giả còn trong team và sẵn sàng|
|Đọc test trước|Project có test coverage tốt (> 70%)|
|**Quy trình 5 bước dưới đây**|Module lớn, nhiều file, ít hoặc không có doc|

## 9. How (5-bước nghiên cứu module i18n thực tế)

### Bước 1 — Xác định mục tiêu trước khi đọc code

```
Trả lời 3 câu hỏi trước khi mở IDE:
  1. Module này làm gì? (i18n → dịch message theo locale)
  2. Tại sao dự án cần nó? (multi-language support, hay chỉ để format error msg?)
  3. Task của mình là gì? (fix bug? thêm ngôn ngữ mới? refactor?)
```

Nếu không trả lời được → hỏi Quang hoặc đọc JIRA ticket kỹ hơn trước.

---

### Bước 2 — Tìm entry point (đừng đọc từ đầu)

Với i18n trong Spring Boot, entry point thường là:

java

```java
// 1. Tìm class config
@Configuration
public class MessageSourceConfig {
    @Bean
    public MessageSource messageSource() {
        ReloadableResourceBundleMessageSource source = new ReloadableResourceBundleMessageSource();
        source.setBasename("classpath:messages");
        source.setDefaultEncoding("UTF-8");
        return source;
    }

    @Bean
    public LocaleResolver localeResolver() {
        // SessionLocaleResolver hoặc AcceptHeaderLocaleResolver
    }
}

// 2. Tìm nơi gọi MessageSource
// Ctrl+Shift+F (IntelliJ): search "messageSource.getMessage" hoặc "@MessageSource"
```

**Checklist tìm entry point:**

- Search annotation: `@Configuration`, `@Bean`, `@Component` liên quan đến tên module
- Search file config: `application.yml` → tìm key `spring.messages`
- Search nơi được inject: `@Autowired MessageSource` hoặc constructor injection

---

### Bước 3 — Vẽ data flow (bằng tay, không cần tool)

Sau khi tìm được entry point, trace flow **theo chiều request đến response**:

```
Request với header "Accept-Language: vi"
  → LocaleChangeInterceptor (nếu có)
  → LocaleResolver → Locale.forLanguageTag("vi")
  → Controller gọi messageSource.getMessage("error.notFound", null, locale)
  → MessageSource đọc messages_vi.properties
  → "Không tìm thấy tài nguyên"
  → Trả về trong ResponseBody
```

Ghi ra giấy hoặc Obsidian dưới dạng text diagram trước khi đọc implementation.

---

### Bước 4 — Đọc code theo flow đã vẽ, không đọc lan man

java

```java
// ✅ Đúng: đọc theo flow
// Step A: Đọc LocaleResolver config
// Step B: Đọc interceptor (nếu có)
// Step C: Đọc nơi gọi getMessage()
// Step D: Kiểm tra file .properties

// ❌ Sai: mở package i18n rồi đọc từng file theo alphabet
```

Với mỗi class đọc, ghi nhanh 1 dòng: _"Class này làm gì trong flow"_.

---

### Bước 5 — Viết một smoke test nhỏ để verify hiểu đúng

java

```java
// Không cần test production-ready, chỉ cần confirm hiểu đúng
@SpringBootTest
class I18nSmokeTest {

    @Autowired
    private MessageSource messageSource;

    @Test
    void shouldReturnVietnameseMessage() {
        String msg = messageSource.getMessage(
            "error.notFound",
            null,
            new Locale("vi")
        );
        assertThat(msg).isNotBlank();
        System.out.println("VI message: " + msg);
    }
}
```

Nếu test pass → flow đúng. Nếu fail → quay lại bước 2.

## 10. Production concerns

### Scaling

- Với i18n: số lượng locale càng nhiều, `.properties` file càng nhiều → cần naming convention rõ ràng
- Module phức tạp hơn: document data flow ra wiki team trước khi merge

### Failure

- Hiểu sai entry point → fix nhầm chỗ → bug không được giải quyết
- Không verify bằng test → tưởng hiểu nhưng thực ra vẫn sai assumption

### Monitoring

- Sau khi nghiên cứu xong: đo lại bằng câu hỏi _"Tôi có thể giải thích module này cho Quang Tran trong 5 phút không?"_
- Nếu không → chưa hiểu đủ, cần thêm bước

## 11. Common mistakes / anti-patterns

- ❌ **Mistake**: Đọc code ngay khi nhận task, không xác định mục tiêu trước ✅ **Fix**: Dành 5 phút trả lời "Module này làm gì? Task của mình là gì?" trước khi mở IDE
- ❌ **Mistake**: Đọc toàn bộ package từ đầu đến cuối theo alphabet ✅ **Fix**: Tìm entry point → trace theo flow → đọc có chủ đích
- ❌ **Mistake**: Tin rằng mình đã hiểu chỉ qua đọc code ✅ **Fix**: Viết smoke test hoặc giải thích lại bằng text diagram để verify
- ❌ **Mistake**: Không ghi chép lại, research xong là mất ✅ **Fix**: Tạo note Obsidian trong lúc research, không phải sau khi xong

## 12. Sample project (with constraint)

**Tên project**: tamice backend — module i18n **Constraint**: Không được hỏi Quang Tran bất kỳ câu nào trong quá trình research. Chỉ được dùng code, git log, và JIRA ticket. **Output kỳ vọng**: Một data flow diagram dạng text + smoke test chạy được + note này điền đầy đủ section 9.

## 13. Interview

### Core Q&A

1. Q: Khi được giao nghiên cứu một module lạ, bạn bắt đầu từ đâu? A: Tìm entry point (config class hoặc annotation liên quan), sau đó trace data flow từ request đến response. Không đọc từng file theo thứ tự alphabet.
2. Q: Làm sao để biết mình đã hiểu đúng module chứ không phải tự suy đoán? A: Viết một smoke test nhỏ verify behavior, hoặc giải thích lại flow bằng text diagram — nếu không diễn đạt được thì chưa hiểu đủ.

### Follow-up

1. Q: Trong Spring Boot, i18n entry point thường nằm ở đâu? A: Class `@Configuration` định nghĩa bean `MessageSource` và `LocaleResolver`. Trong `application.yml` tìm key `spring.messages.basename`.
2. Q: Sự khác biệt giữa `AcceptHeaderLocaleResolver` và `SessionLocaleResolver`? A: `AcceptHeaderLocaleResolver` đọc locale từ HTTP header `Accept-Language` — stateless, phù hợp REST API. `SessionLocaleResolver` lưu locale trong session — phù hợp web app có login state.

### Scenario

> "Bạn được giao fix bug: API trả về message tiếng Anh dù client gửi `Accept-Language: vi`. Bạn sẽ debug theo hướng nào?"

A: Check theo flow: (1) `LocaleResolver` bean có được config không, có phải dùng đúng loại resolver không. (2) `LocaleChangeInterceptor` có được đăng ký vào `InterceptorRegistry` không. (3) File `messages_vi.properties` có tồn tại và có key đó không. (4) Nơi gọi `getMessage()` có truyền `locale` hay hardcode `Locale.ENGLISH`. Trace từng bước thay vì đoán mò.