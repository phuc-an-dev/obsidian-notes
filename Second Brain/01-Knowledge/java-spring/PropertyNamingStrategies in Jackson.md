---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
  - "#topic/serialization"
related:
  - "[[PropertyNamingStrategy in Jackson.md]]"
---

## 1. What
`PropertyNamingStrategies` là một lớp tĩnh (static class) trong thư viện Jackson (phiên bản 2.12 trở lên), chứa các hằng số và lớp lồng nhau đại diện cho các chiến lược đặt tên thuộc tính JSON. Đây là phiên bản hiện đại, thay thế cho lớp `PropertyNamingStrategy` cũ đã bị lỗi thời.

## 2. Why
Jackson thực hiện tái cấu trúc (refactoring) để cải thiện tính đóng gói và dễ mở rộng. `PropertyNamingStrategies` ra đời để:
- **Tránh nhầm lẫn**: Phân biệt rõ ràng giữa "lớp cơ sở để kế thừa" (`PropertyNamingStrategy`) và "danh sách các chiến lược có sẵn" (`PropertyNamingStrategies`).
- **Hiện đại hóa**: Cung cấp cách tiếp cận kiểu "Fluent API" và nhất quán hơn với các module khác của Jackson.

## 3. Mental Model
Hãy tưởng tượng `PropertyNamingStrategies` giống như một **"Menu các kiểu font chữ"** trong Microsoft Word:
- Thay vì bạn phải tự thiết kế cách vẽ từng chữ (viết code strategy), bạn chỉ cần chọn từ Menu: "In đậm" (Snake Case), "Gạch ngang" (Kebab Case), hoặc "Viết hoa chữ cái đầu" (Pascal Case).
- Menu này tập hợp tất cả các lựa chọn chuẩn nhất vào một nơi để bạn dễ dàng tìm thấy và áp dụng.

## 4. Where it fits
Vị trí trong hệ thống:
`ObjectMapper -> setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE) -> Serialization Process`

## 5. When to use
- Trong mọi dự án Spring Boot mới sử dụng Jackson 2.12+ (Spring Boot 2.4 trở lên).
- Khi cần chuyển đổi định dạng tên field cho toàn bộ ứng dụng một cách chuyên nghiệp và không bị cảnh báo Deprecated.

## 6. When NOT to use
- Khi đang làm việc trong các dự án cực cũ sử dụng Jackson phiên bản 2.11 trở xuống.
- Khi chỉ muốn thay đổi tên cho một vài trường đơn lẻ (dùng `@JsonProperty`).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Là chuẩn mới nhất, được hỗ trợ lâu dài bởi Jackson. | Tên gọi hơi dài hơn một chút so với bản cũ. |
| Hỗ trợ đầy đủ các chuẩn: Snake, Kebab, LowerCase, UpperCamelCase, LowerDotCase. | Không có sự thay đổi về mặt logic xử lý so với bản cũ (chỉ thay đổi về cách tổ chức lớp). |
| Giúp tránh các cảnh báo gạch ngang (Deprecated) trong IDE. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `PropertyNamingStrategy` | Bản cũ, không nên dùng nữa. |
| Custom Strategy | Kế thừa từ `PropertyNamingStrategy` để tự viết logic đặt tên riêng (ví dụ: thêm tiền tố "api_" vào mọi trường). |

## 9. How
Các chiến lược phổ biến có trong `PropertyNamingStrategies`:

```java
// 1. SNAKE_CASE: user_name
ObjectMapper mapper = new ObjectMapper()
    .setPropertyNamingStrategy(PropertyNamingStrategies.SNAKE_CASE);

// 2. KEBAB_CASE: user-name
mapper.setPropertyNamingStrategy(PropertyNamingStrategies.KEBAB_CASE);

// 3. LOWER_CASE: username
mapper.setPropertyNamingStrategy(PropertyNamingStrategies.LOWER_CASE);

// 4. UPPER_CAMEL_CASE (PascalCase): UserName
mapper.setPropertyNamingStrategy(PropertyNamingStrategies.UPPER_CAMEL_CASE);

// 5. LOWER_DOT_CASE: user.name
mapper.setPropertyNamingStrategy(PropertyNamingStrategies.LOWER_DOT_CASE);
```

Cấu hình trong `application.yml` (Spring Boot tự nhận diện):
```yaml
spring:
  jackson:
    property-naming-strategy: SNAKE_CASE
```

## 10. Production concerns
### Migration
Khi nâng cấp Jackson trong dự án cũ, hãy kiểm tra các file cấu hình Java. Nếu thấy IDE báo lỗi hoặc cảnh báo ở dòng `setPropertyNamingStrategy`, hãy đổi sang dùng `PropertyNamingStrategies`.

### Third-party SDKs
Một số SDK yêu cầu định dạng JSON cực kỳ đặc thù. Hãy cân nhắc tạo một `ObjectMapper` riêng cho các SDK đó để không ảnh hưởng đến API chung của hệ thống.

## 11. Common mistakes
- Mistake: Nhầm lẫn giữa `PropertyNamingStrategy` (Lớp cơ sở) và `PropertyNamingStrategies` (Lớp chứa hằng số).
  Fix: Luôn dùng bản có chữ 's' để chọn các strategy có sẵn.

- Mistake: Khai báo strategy trong code nhưng trong `application.properties` lại khai báo khác, dẫn đến xung đột.

## 12. Sample project
Sử dụng `LOWER_DOT_CASE` cho một dự án Spring Boot để tạo ra các API trả về JSON dùng cho các hệ thống quản lý cấu hình (Configuration Management) vốn rất ưa chuộng định dạng `key.name`.

## 13. Interview
### Core Q&A
1. Q: Tại sao Jackson lại chuyển từ `PropertyNamingStrategy` sang `PropertyNamingStrategies`?
   A: Để phân tách rõ ràng giữa kiểu dữ liệu (Class) và các thể hiện (Instances/Constants), đồng thời dọn dẹp các thiết kế cũ để hỗ trợ mở rộng tốt hơn trong tương lai.

2. Q: `LowerDotCase` là gì và khi nào dùng?
   A: Nó chuyển `userName` thành `user.name`. Thường dùng trong các hệ thống tích hợp log hoặc config server.

### Scenario
"Làm thế nào để áp dụng Snake Case chỉ cho một đối tượng DTO cụ thể mà không ảnh hưởng toàn cục?"
-> Trả lời: Tôi sẽ sử dụng Annotation `@JsonNaming(PropertyNamingStrategies.SnakeCaseStrategy.class)` trực tiếp trên Class của DTO đó.

## 14. References
- Jackson Release Notes: [Jackson 2.12](https://github.com/FasterXML/jackson/wiki/Jackson-Release-2.12#databind)
- Official Doc: [PropertyNamingStrategies](https://fasterxml.github.io/jackson-databind/javadoc/2.12/com/fasterxml/jackson/databind/PropertyNamingStrategies.html)

## 15. Real-world Code
Kiểm tra class `JacksonAutoConfiguration` trong mã nguồn Spring Boot để thấy cách nó nạp các strategy này từ file cấu hình.

## 16. Community
- Reddit: r/java.
- Stack Overflow: Tag [jackson] [jackson-databind].
