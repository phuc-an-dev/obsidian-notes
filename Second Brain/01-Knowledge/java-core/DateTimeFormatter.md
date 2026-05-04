---
created: 2026-05-04
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
  - "#topic/date-time"
related:
  - "[[LocalDateTime]]"
  - "[[Instant]]"
  - "[[ParseException]]"
---

## 1. What
`DateTimeFormatter` là một lớp trong gói `java.time.format` dùng để in (formatting) và phân tích (parsing) các đối tượng ngày giờ trong Java 8+. Nó thay thế hoàn toàn cho lớp `SimpleDateFormat` cũ vốn có nhiều nhược điểm về bảo mật và hiệu năng.

## 2. Why
Trước Java 8, `SimpleDateFormat` không an toàn trong môi trường đa luồng (not thread-safe), buộc lập trình viên phải tạo mới instance liên tục hoặc dùng `ThreadLocal`. `DateTimeFormatter` ra đời để giải quyết triệt để vấn đề này bằng cách thiết kế dưới dạng bất biến (immutable) và cung cấp các hằng số chuẩn hóa (như ISO-8601) sẵn có.

## 3. Mental Model
Hãy tưởng tượng `DateTimeFormatter` như một **"Khuôn mẫu (Stencil)"**. 
- Khi bạn có một "Khối đất sét" (đối tượng `LocalDateTime`), bạn ép cái khuôn lên đó để nó biến thành một "Hình dạng văn bản" (String) đẹp đẽ.
- Ngược lại, khi bạn có một "Dòng chữ" (String), bạn dùng cái khuôn này để "Đúc" nó thành một đối tượng ngày giờ có cấu trúc.

## 4. Where it fits
Nó đóng vai trò là bộ lọc chuyển đổi ở lớp giao tiếp dữ liệu:
`Date Object <-> DateTimeFormatter <-> String (User UI / API JSON)`

## 5. When to use
- Khi cần hiển thị ngày giờ cho người dùng theo định dạng mong muốn (ví dụ: dd/MM/yyyy).
- Khi cần parse dữ liệu ngày giờ từ các chuỗi văn bản nhận được từ API, file CSV hoặc Database.
- Khi cần làm việc với các chuẩn thời gian quốc tế (ISO-8601).

## 6. When NOT to use
- Khi bạn chỉ thực hiện các phép tính toán ngày giờ nội bộ (cộng, trừ, so sánh) mà không cần biểu diễn dưới dạng chuỗi.
- Khi làm việc với các hệ thống Legacy cực cũ (trước Java 8) mà không thể dùng thư viện hỗ trợ (backport).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| **Thread-safe**: Có thể dùng chung một instance (Singleton) cho toàn bộ app. | Cú pháp các ký tự định dạng (pattern) có một số thay đổi nhỏ so với bản cũ. |
| **Immutable**: Không thể bị thay đổi trạng thái sau khi khởi tạo. | Ném ra `DateTimeParseException` (Runtime) thay vì `ParseException` (Checked). |
| Cung cấp sẵn các định dạng ISO chuẩn hóa cực kỳ tiện lợi. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `SimpleDateFormat` | Cũ, không thread-safe, nên tránh dùng trong các dự án mới. |
| `FastDateFormat` | Thuộc Apache Commons Lang, thread-safe nhưng hiện tại đã ít dùng hơn formatter native của Java. |

## 9. How
```java
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;

public class FormatterExample {
    // Nên khai báo static final để tái sử dụng vì nó thread-safe
    private static final DateTimeFormatter CUSTOM_FORMATTER = 
        DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm:ss");

    public static void main(String[] args) {
        LocalDateTime now = LocalDateTime.now();

        // 1. Formatting (Object -> String)
        String formattedDate = now.format(CUSTOM_FORMATTER);
        System.out.println("Formatted: " + formattedDate);

        // 2. Parsing (String -> Object)
        String input = "25/12/2023 20:00:00";
        LocalDateTime parsedDate = LocalDateTime.parse(input, CUSTOM_FORMATTER);
        System.out.println("Parsed: " + parsedDate);
        
        // 3. Sử dụng hằng số ISO có sẵn
        String isoDate = now.format(DateTimeFormatter.ISO_DATE_TIME);
    }
}
```

## 10. Production concerns
### Scaling
Vì `DateTimeFormatter` là thread-safe, bạn nên khởi tạo nó một lần duy nhất dưới dạng `static final` hằng số để tối ưu hiệu năng, tránh việc compile pattern lặp đi lặp lại.

### Failure
Nếu chuỗi đầu vào không khớp chính xác với khuôn mẫu (pattern), hệ thống sẽ ném ra `DateTimeParseException`. Cần có logic catch exception này để xử lý lỗi input từ phía client.

### Monitoring
Theo dõi các lỗi parse thời gian trong log để phát hiện các thay đổi không báo trước từ các API tích hợp bên thứ ba.

## 11. Common mistakes
- **Mistake**: Sử dụng `yyyy` thay cho `uuuu` trong một số trường hợp liên quan đến kỷ nguyên (Era). 
  **Fix**: Trong Java 8+, `uuuu` thường an toàn hơn cho các năm âm hoặc các hệ lịch đặc biệt.
  
- **Mistake**: Tạo mới `DateTimeFormatter` bên trong một vòng lặp hàng triệu lần.
  **Fix**: Đưa formatter ra ngoài làm hằng số dùng chung.

## 12. Sample project
Xây dựng một lớp `LogParser` chuyên đọc các file log từ nhiều hệ thống khác nhau (mỗi hệ thống có một định dạng ngày giờ riêng). Sử dụng một `Map<SystemType, DateTimeFormatter>` để ánh xạ và chuyển đổi tất cả về chuẩn `Instant` duy nhất.

## 13. Interview
### Core Q&A
1. **Q**: Tại sao `DateTimeFormatter` lại tốt hơn `SimpleDateFormat`?
   **A**: Vì nó thread-safe (có thể dùng chung giữa các luồng), bất biến (immutable), và có API rõ ràng, mạnh mẽ hơn.
2. **Q**: `DateTimeFormatter` ném ra ngoại lệ gì khi parse lỗi?
   **A**: Nó ném ra `DateTimeParseException` (một loại Runtime Exception), khác với `ParseException` (Checked Exception) của bản cũ.
3. **Q**: Làm thế nào để tạo một formatter có tính đến ngôn ngữ (Locale)?
   **A**: Sử dụng `DateTimeFormatter.ofPattern(pattern).withLocale(Locale.US)`.

### Scenario
**Tình huống**: Bạn đang nhận một chuỗi thời gian "20231225". Bạn parse nó như thế nào?
**Trả lời**: Tôi sẽ dùng `DateTimeFormatter.ofPattern("yyyyMMdd")` và gọi `LocalDate.parse(input, formatter)`.

## 14. References
- Official Docs: [DateTimeFormatter API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/time/format/DateTimeFormatter.html)

## 15. Real-world Code
- Thường được dùng trong cấu hình `Jackson` để format ngày giờ khi trả về JSON trong Spring Boot.

## 16. Community
- Reddit: r/java
- Stack Overflow: Tag [datetimeformatter]
- Blog: Baeldung (Guide to DateTimeFormatter).
