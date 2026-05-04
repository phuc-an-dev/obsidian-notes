---
created: 2026-05-04
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
  - "#topic/error-handling"
related:
  - "[[IOException]]"
  - "[[Instant]]"
---

## 1. What
`ParseException` là một "checked exception" trong Java thuộc gói `java.text`. Nó báo hiệu rằng một lỗi đã xảy ra một cách bất ngờ trong quá trình chuyển đổi (parsing) một chuỗi văn bản (String) sang một kiểu dữ liệu có cấu trúc (như `Date` hoặc `Number`).

## 2. Why
Dữ liệu đầu vào từ người dùng hoặc từ các hệ thống bên thứ ba thường không đáng tin cậy. Khi bạn yêu cầu Java "hãy biến chuỗi '2023-13-45' thành một ngày", Java sẽ không thể thực hiện được vì định dạng không hợp lệ. `ParseException` buộc lập trình viên phải xử lý tình huống dữ liệu sai lệch này ngay từ lúc viết code để đảm bảo ứng dụng không bị dừng đột ngột.

## 3. Mental Model
Hãy tưởng tượng `ParseException` như một **"Lỗi của biên dịch viên"**. Bạn đưa cho một người dịch một câu tiếng Pháp và yêu cầu họ dịch sang tiếng Việt. Nếu trong câu đó có những từ không phải tiếng Pháp hoặc ngữ pháp hoàn toàn sai, người dịch sẽ dừng lại và bảo: "Tôi không hiểu cấu trúc này, tôi không thể dịch tiếp được".

## 4. Where it fits
Nó nằm trong hệ thống phân cấp ngoại lệ của Java, thường xuất hiện ở tầng Data Conversion:
`Throwable -> Exception -> ParseException`

## 5. When to use
- Khi sử dụng lớp `SimpleDateFormat` để chuyển đổi chuỗi thành đối tượng `Date`.
- Khi sử dụng `NumberFormat` hoặc `DecimalFormat` để parse chuỗi thành số.
- Khi xây dựng các bộ parser tùy chỉnh cho các định dạng văn bản có quy tắc.

## 6. When NOT to use
- Khi sử dụng API Date-Time mới của Java 8 (`java.time`). Các lớp này ném ra `DateTimeParseException` (là một runtime exception).
- Khi kiểm tra tính hợp lệ của dữ liệu đầu vào đơn giản (nên dùng validation logic thay vì dựa vào exception).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Bắt buộc lập trình viên phải xử lý lỗi định dạng dữ liệu. | Là checked exception nên gây rườm rà (boilerplate code) với try-catch. |
| Cung cấp `errorOffset` để biết chính xác vị trí lỗi trong chuỗi. | Thiết kế cũ, không phù hợp với phong cách lập trình hàm (functional programming). |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `DateTimeParseException` | (Java 8+) Là Unchecked Exception, chuyên dụng cho thời gian, dễ dùng hơn. |
| `NumberFormatException` | (Của lớp `Integer`, `Double`) Cũng là Unchecked Exception, phổ biến hơn cho việc parse số cơ bản. |
| `Regular Expressions` | Dùng để validate định dạng trước khi parse để tránh ném exception. |

## 9. How
```java
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Date;

public class ParseExample {
    public static void main(String[] args) {
        SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd");
        String input = "2023-05-32"; // Ngày không hợp lệ

        try {
            Date date = sdf.parse(input);
            System.out.println(date);
        } catch (ParseException e) {
            System.err.println("Lỗi định dạng tại vị trí: " + e.getErrorOffset());
            e.printStackTrace();
        }
    }
}
```

## 10. Production concerns
### Scaling
Việc ném ngoại lệ liên tục trong các vòng lặp xử lý dữ liệu lớn (batch processing) sẽ làm giảm hiệu năng hệ thống. Nên validate định dạng chuỗi bằng Regex trước nếu có thể.

### Failure
Cần phân biệt giữa lỗi định dạng (input sai) và lỗi logic (định dạng đúng nhưng giá trị không thực tế).

### Monitoring
Log lại các chuỗi gây lỗi parse để phân tích xem người dùng thường nhập sai ở đâu hoặc API bên thứ ba có thay đổi định dạng hay không.

## 11. Common mistakes
- **Mistake**: Sử dụng `SimpleDateFormat` (vốn ném `ParseException`) trong môi trường đa luồng mà không có đồng bộ hóa.
  **Fix**: Dùng `ThreadLocal<SimpleDateFormat>` hoặc chuyển sang `java.time.format.DateTimeFormatter` (thread-safe).

- **Mistake**: Bỏ qua `errorOffset` khi log lỗi.
  **Fix**: Luôn in ra vị trí lỗi để dễ dàng debug dữ liệu đầu vào.

## 12. Sample project
Viết một chương trình đọc danh sách ngày sinh từ một file CSV. Nếu một dòng bị lỗi định dạng ngày tháng, hãy bỏ qua dòng đó, log lại vị trí lỗi và tiếp tục xử lý các dòng tiếp theo thay vì dừng toàn bộ chương trình.

## 13. Interview
### Core Q&A
1. **Q**: `ParseException` là checked hay unchecked exception? Tại sao?
   **A**: Là checked exception. Vì việc parse dữ liệu từ nguồn bên ngoài là một thao tác tiềm ẩn nhiều rủi ro và Java muốn đảm bảo lập trình viên luôn có phương án xử lý khi dữ liệu không khớp định dạng.
2. **Q**: Làm thế nào để biết vị trí chính xác của lỗi trong chuỗi?
   **A**: Sử dụng phương thức `getErrorOffset()` của đối tượng `ParseException`.
3. **Q**: Tại sao Java 8 lại giới thiệu `DateTimeParseException` thay vì dùng tiếp `ParseException`?
   **A**: Để chuyển sang mô hình Runtime Exception giúp code gọn gàng hơn và tương thích tốt hơn với Lambda/Streams.

### Scenario
**Tình huống**: Bạn nhận được một chuỗi JSON chứa ngày tháng dạng "dd/MM/yyyy", nhưng thỉnh thoảng client lại gửi "dd-MM-yyyy". Bạn xử lý thế nào với `ParseException`?
**Trả lời**: Tôi sẽ thử parse với định dạng thứ nhất trong khối `try`. Nếu bắt được `ParseException`, tôi sẽ tiếp tục thử parse với định dạng thứ hai trong khối `catch`. Nếu cả hai đều thất bại, lúc đó mới ném ra ngoại lệ cuối cùng.

## 14. References
- Official Docs: [java.text.ParseException API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/text/ParseException.html)

## 15. Real-world Code
- Thường thấy trong các bộ parser cũ của Hibernate hoặc các thư viện xử lý Excel (Apache POI).

## 16. Community
- Stack Overflow: Tag [parseexception]
- Blog: "Handling Exceptions in Java" (Baeldung).
