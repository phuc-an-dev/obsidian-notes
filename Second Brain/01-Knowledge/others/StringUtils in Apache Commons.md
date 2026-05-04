---
created: 2026-05-04
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/java"
  - "#topic/clean-code"
related:
  - "[[RequiredArgsConstructor in Lombok]]"
---

## 1. What
`StringUtils` là một lớp tiện ích (utility class) thuộc thư viện **Apache Commons Lang**, cung cấp hàng trăm phương thức tĩnh (static methods) để thao tác với chuỗi (String). Điểm đặc trưng nhất của `StringUtils` là khả năng xử lý "Null-safe" - tức là hầu hết các phương thức sẽ không ném ra `NullPointerException` nếu đầu vào là `null`.

## 2. Why
Trong Java thuần, các thao tác như kiểm tra chuỗi rỗng (`str.isEmpty()`) hoặc cắt khoảng trắng (`str.trim()`) sẽ ném ra lỗi nếu `str` là `null`. Điều này buộc lập trình viên phải viết rất nhiều câu lệnh `if (str != null)` rườm rà. `StringUtils` ra đời để đơn giản hóa việc này, giúp code sạch hơn, an toàn hơn và xử lý được nhiều tình huống phức tạp (như kiểm tra chuỗi chỉ toàn khoảng trắng).

## 3. Mental Model
Hãy tưởng tượng `StringUtils` như một chiếc **"Dao đa năng Thụy Sĩ (Swiss Army Knife)"** được bọc một lớp **"Vỏ chống nước (Null-safe wrapper)"**. Bạn có thể dùng nó để cắt, gọt, vặn vít (trim, split, join) một cách thoải mái. Ngay cả khi bạn "nhúng" nó vào nước (truyền vào giá trị `null`), nó vẫn hoạt động bình thường mà không bị hỏng hóc hay gây ra sự cố.

## 4. Where it fits
Nó nằm ở tầng Utility/Helper, được sử dụng xuyên suốt ở tất cả các tầng (Controller, Service, Repository) để chuẩn hóa dữ liệu đầu vào và đầu ra.

## 5. When to use
- Khi cần kiểm tra chuỗi rỗng hoặc chuỗi trắng (`isBlank`, `isEmpty`).
- Khi cần nối chuỗi (`join`) hoặc cắt chuỗi (`split`) một cách an toàn.
- Khi cần thực hiện các thao tác định dạng như viết hoa chữ cái đầu (`capitalize`), đảo ngược chuỗi (`reverse`), hoặc thêm ký tự đệm (`pad`).
- Khi làm việc với dữ liệu không đáng tin cậy từ người dùng hoặc API bên thứ ba.

## 6. When NOT to use
- Khi bạn đang sử dụng Java 11+ và chỉ cần các thao tác cực kỳ cơ bản (như `String.isBlank()`, `String.repeat()`) mà không muốn thêm dependency bên ngoài.
- Khi hiệu năng là cực kỳ tối quan trọng và bạn muốn tránh overhead của việc gọi qua một thư viện trung gian (tuy nhiên overhead này là rất nhỏ).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Null-safe tuyệt đối, loại bỏ hàng loạt check null. | Thêm một dependency (mặc dù nhẹ) vào project. |
| API phong phú, giải quyết được nhiều case khó. | Có thể gây nhầm lẫn với `StringUtils` của Spring hoặc các thư viện khác. |
| Code ngắn gọn, dễ đọc và bảo trì. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `java.lang.String` | (JDK 11+) Đã bổ sung `isBlank()`, `repeat()`, `strip()`... nhưng vẫn không null-safe. |
| `Guava Strings` | Thư viện của Google, gọn nhẹ nhưng ít tính năng hơn Commons Lang. |
| `Spring StringUtils` | Tích hợp sẵn trong Spring, nhưng mục đích chính là dùng nội bộ cho framework, ít tính năng xử lý chuỗi chuyên sâu. |

## 9. How
```java
import org.apache.commons.lang3.StringUtils;

public class StringUtilsExample {
    public static void main(String[] args) {
        String input = "  ";

        // 1. Kiểm tra Null-safe
        System.out.println(StringUtils.isEmpty(input)); // false (vì có khoảng trắng)
        System.out.println(StringUtils.isBlank(input)); // true (vì chỉ có khoảng trắng hoặc rỗng)
        System.out.println(StringUtils.isBlank(null));  // true (Không ném NPE)

        // 2. Thao tác biến đổi
        System.out.println(StringUtils.capitalize("hello world")); // "Hello world"
        System.out.println(StringUtils.abbreviate("Đây là một chuỗi rất dài", 10)); // "Đây là..."

        // 3. Nối/Cắt chuỗi
        String[] parts = {"Java", "Spring", "Lombok"};
        System.out.println(StringUtils.join(parts, " - ")); // "Java - Spring - Lombok"
        
        // 4. Mặc định giá trị
        System.out.println(StringUtils.defaultString(null, "N/A")); // "N/A"
    }
}
```

## 10. Production concerns
### Scaling
`StringUtils` là stateless (không lưu trạng thái) và các phương thức là static, nên nó hoàn toàn thread-safe và cực kỳ nhẹ khi chạy trong các ứng dụng quy mô lớn.

### Failure
Rủi ro lớn nhất là nạp sai thư viện (ví dụ nạp `commons-lang` bản cũ 2.x thay vì `commons-lang3`). Bản cũ sử dụng package `org.apache.commons.lang`, không tương thích hoàn toàn.

### Monitoring
Không cần monitoring đặc biệt, nhưng cần chú ý trong quá trình build để tránh "Dependency Hell" (xung đột phiên bản).

## 11. Common mistakes
- **Mistake**: Nhầm lẫn giữa `isEmpty()` và `isBlank()`.
  **Fix**: `isEmpty()` chỉ kiểm tra độ dài bằng 0. `isBlank()` kiểm tra cả trường hợp chuỗi chỉ chứa khoảng trắng (space, tab, newline). Luôn ưu tiên `isBlank()` khi xử lý input từ form.

- **Mistake**: Import nhầm `StringUtils` của Spring Framework (`org.springframework.util.StringUtils`).
  **Fix**: Luôn kiểm tra kỹ package `org.apache.commons.lang3`.

## 12. Sample project
Xây dựng một lớp `UserSanitizer` để chuẩn hóa thông tin người dùng:
- Xóa khoảng trắng thừa ở hai đầu.
- Viết hoa chữ cái đầu của Tên.
- Nếu người dùng không nhập Bio, mặc định trả về "Chưa có giới thiệu".
**Ràng buộc**: Không được sử dụng câu lệnh `if` để check null.

## 13. Interview
### Core Q&A
1. **Q**: Sự khác biệt lớn nhất giữa `StringUtils.isBlank()` và `String.isEmpty()` (trong JDK) là gì?
   **A**: `isBlank()` của StringUtils trả về `true` nếu chuỗi là `null`, rỗng, hoặc chỉ chứa khoảng trắng. `isEmpty()` của JDK sẽ ném `NullPointerException` nếu chuỗi là `null` và trả về `false` nếu chuỗi có khoảng trắng.
2. **Q**: Tại sao `StringUtils` lại được coi là Null-safe?
   **A**: Vì bên trong các phương thức của nó luôn có check `if (str == null) return ...` trước khi thực hiện logic, giúp bảo vệ ứng dụng khỏi NPE.
3. **Q**: Phương thức `abbreviate()` dùng để làm gì?
   **A**: Dùng để rút gọn một chuỗi dài và thêm dấu "..." vào cuối sao cho tổng độ dài không vượt quá một mức cho phép (thường dùng để hiển thị preview).

### Scenario
**Tình huống**: Bạn đang nhận một danh sách các từ khóa từ một file CSV, có những dòng bị trống hoặc chỉ có dấu cách. Bạn muốn lọc bỏ chúng và nối các từ khóa còn lại bằng dấu phẩy. Bạn dùng `StringUtils` như thế nào?
**Trả lời**: Tôi sẽ lặp qua danh sách, dùng `StringUtils.isBlank()` để loại bỏ các dòng không hợp lệ, sau đó dùng `StringUtils.join()` để nối các phần tử còn lại một cách nhanh chóng.

## 14. References
- Official Docs: [Apache Commons Lang StringUtils API](https://commons.apache.org/proper/commons-lang/javadocs/api-release/org/apache/commons/lang3/StringUtils.html)
- GitHub Repo: [https://github.com/apache/commons-lang](https://github.com/apache/commons-lang)

## 15. Real-world Code
- Xuất hiện trong hầu hết các dự án Java Enterprise, đặc biệt là trong các lớp xử lý DTO và Validation.

## 16. Community
- Reddit: r/java
- Stack Overflow: Tag [commons-lang]
- Blog: "Mastering Apache Commons StringUtils" (Baeldung).
