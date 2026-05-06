---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
  - "#topic/serialization"
related:
  - "[[PropertyNamingStrategies in Jackson.md]]"
---

## 1. What
`PropertyNamingStrategy` là một lớp trong thư viện Jackson (thường dùng trong Spring Boot) cho phép tùy biến cách đặt tên các trường (fields) khi chuyển đổi giữa Java Object và JSON. Nó định nghĩa các quy tắc chung để map các tên biến kiểu `camelCase` của Java sang các định dạng khác như `snake_case` hoặc `kebab-case`.

## 2. Why
Trong Java, quy chuẩn đặt tên biến là `camelCase` (ví dụ: `userName`). Tuy nhiên, nhiều hệ thống bên ngoài hoặc API tiêu chuẩn lại yêu cầu định dạng `snake_case` (ví dụ: `user_name`). Thay vì phải đánh dấu `@JsonProperty` thủ công cho từng trường, `PropertyNamingStrategy` cung cấp một giải pháp cấu hình tập trung cho toàn bộ ứng dụng.

## 3. Mental Model
Hãy tưởng tượng `PropertyNamingStrategy` giống như một **"Máy phiên dịch tên"** tại quầy thủ tục:
- Phía trong (Java) dùng ngôn ngữ địa phương: `firstName`.
- Phía ngoài (JSON/API) yêu cầu ngôn ngữ quốc tế: `first_name`.
- Khi bất kỳ tài liệu nào đi qua quầy, máy sẽ tự động dịch tên theo đúng quy tắc đã cài đặt mà bạn không cần phải viết lại từng chữ trên tài liệu đó.

## 4. Where it fits
Vị trí trong quy trình xử lý dữ liệu:
`Java Object (camelCase) -> Jackson ObjectMapper -> PropertyNamingStrategy -> JSON Output (snake_case)`

## 5. When to use
- Khi muốn đồng bộ hóa toàn bộ API của dự án theo một quy chuẩn đặt tên nhất định (thường là `snake_case`).
- Khi tích hợp với các hệ thống cũ hoặc API của bên thứ ba có quy tắc đặt tên khác với Java.

## 6. When NOT to use
- Kể từ Jackson 2.12, lớp này đã bị **Deprecated** (lỗi thời). Bạn nên chuyển sang sử dụng `PropertyNamingStrategies` (có chữ 's' ở cuối).
- Khi dự án chỉ có một vài trường đặc thù cần đổi tên (trường hợp này dùng `@JsonProperty` sẽ tường minh hơn).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Cấu hình tập trung, giảm bớt boilerplate code. | Gây khó khăn khi tìm kiếm (grep) tên trường giữa code và log JSON. |
| Dễ dàng thay đổi quy chuẩn cho toàn bộ dự án. | Có thể gây nhầm lẫn nếu không được tài liệu hóa rõ ràng cho team Frontend. |
| Giúp code Java giữ đúng chuẩn `camelCase`. | Đã bị đánh dấu Deprecated trong các bản mới nhất. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `@JsonProperty("name")` | Áp dụng cho từng trường cụ thể, độ ưu tiên cao hơn strategy. |
| `PropertyNamingStrategies` | Phiên bản kế thừa hiện đại, khuyên dùng cho các dự án mới. |

## 9. How
Cách cấu hình trong Spring Boot (Sử dụng lớp cũ):

```java
@Configuration
public class JacksonConfig {
    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        // Cấu hình toàn bộ sang snake_case
        mapper.setPropertyNamingStrategy(PropertyNamingStrategy.SNAKE_CASE);
        return mapper;
    }
}
```

Hoặc qua `application.properties`:
`spring.jackson.property-naming-strategy=SNAKE_CASE`

## 10. Production concerns
### Compatibility
Khi thay đổi strategy cho một dự án đang chạy, bạn sẽ làm hỏng (break) toàn bộ các client đang gọi API đó vì tên trường JSON bị thay đổi hoàn toàn. Luôn thực hiện việc này ngay từ đầu dự án hoặc có kế hoạch migration cẩn thận.

### Mixed Naming
Nếu một số API cần `snake_case` nhưng số khác cần giữ nguyên `camelCase`, việc dùng strategy toàn cục sẽ gặp khó khăn. Bạn có thể cần định nghĩa nhiều `ObjectMapper` khác nhau.

## 11. Common mistakes
- Mistake: Tiếp tục dùng `PropertyNamingStrategy` trong Spring Boot 3 / Jackson 2.12+.
  Fix: Chuyển sang `PropertyNamingStrategies`.

- Mistake: Quên rằng `@JsonProperty` sẽ ghi đè (override) cấu hình của strategy.

## 12. Sample project
Tạo một DTO `UserDto` với trường `isActiveMember`. Cấu hình `SNAKE_CASE` và kiểm chứng kết quả JSON trả về là `"is_active_member": true`.

## 13. Interview
### Core Q&A
1. Q: `PropertyNamingStrategy` dùng để làm gì?
   A: Dùng để cấu hình quy tắc đặt tên trường JSON một cách tập trung cho toàn bộ các đối tượng Java khi được Jackson serialize/deserialize.

2. Q: Strategy nào thường được dùng nhất trong các ứng dụng Web?
   A: `SNAKE_CASE` (dùng dấu gạch dưới) là phổ biến nhất để tương thích với các chuẩn API RESTful.

### Scenario
"Tại sao bạn cấu hình SNAKE_CASE rồi nhưng một số trường vẫn hiển thị kiểu camelCase trong JSON?"
-> Trả lời: Có 2 khả năng: 1 là các trường đó đang được đánh dấu bằng `@JsonProperty` thủ công (annotation này có độ ưu tiên cao hơn). 2 là các trường đó không có Getter/Setter hợp lệ khiến Jackson không nhận diện được để áp dụng strategy.

## 14. References
- Jackson Javadoc: [PropertyNamingStrategy](https://fasterxml.github.io/jackson-databind/javadoc/2.11/com/fasterxml/jackson/databind/PropertyNamingStrategy.html)
- Baeldung: [Jackson Property Naming Strategy](https://www.baeldung.com/jackson-property-naming-strategy)

## 15. Real-world Code
Nghiên cứu file cấu hình Jackson trong các dự án Starter của JHipster để thấy cách họ quản lý chuẩn đặt tên.

## 16. Community
- GitHub: [FasterXML/jackson-databind](https://github.com/FasterXML/jackson-databind)
- Stack Overflow: Tag [jackson] [propertynamingstrategy].
