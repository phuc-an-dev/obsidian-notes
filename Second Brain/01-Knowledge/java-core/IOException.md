---
created: 2026-05-04
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
  - "#topic/error-handling"
related:
  - "[[InputStream]]"
  - "[[ByteArrayInputStream]]"
---

## 1. What
`IOException` (Input/Output Exception) là một "checked exception" trong Java, báo hiệu rằng một thao tác I/O (đọc/ghi dữ liệu) đã bị thất bại hoặc bị gián đoạn. Đây là lớp cha của hầu hết các ngoại lệ liên quan đến việc tương tác với tài nguyên bên ngoài như file, network, hoặc database.

## 2. Why
Khác với các tính toán nội bộ trong RAM, các thao tác I/O phụ thuộc vào các yếu tố nằm ngoài tầm kiểm soát của chương trình (ổ cứng bị rút, mất mạng, file bị xóa bởi tiến trình khác). `IOException` tồn tại để bắt buộc lập trình viên phải dự phòng và xử lý các tình huống "không mong muốn nhưng có thể xảy ra" này để đảm bảo ứng dụng không bị crash đột ngột.

## 3. Mental Model
Hãy tưởng tượng bạn đang cố gắng gửi một bức thư qua **"Hệ thống bưu điện (I/O)"**. `IOException` giống như một **"Thông báo thất lạc"** từ bưu điện. Bạn không thể chắc chắn 100% lá thư sẽ đến nơi (vì bão, tai nạn, nhầm địa chỉ). Hệ thống bưu điện yêu cầu bạn phải để lại địa chỉ phản hồi (try-catch) để họ biết phải báo cho ai nếu có sự cố xảy ra.

## 4. Where it fits
Nó nằm ở đỉnh của hệ thống phân cấp ngoại lệ I/O:
`Throwable -> Exception -> IOException -> [FileNotFoundException, SocketException, EOFException, ...]`

## 5. When to use
- Khi định nghĩa các phương thức có thực hiện đọc/ghi dữ liệu và muốn bắt buộc người gọi phải xử lý lỗi.
- Khi muốn bắt (catch) tất cả các lỗi I/O chung chung mà không cần quan tâm chi tiết đó là lỗi file hay lỗi mạng.

## 6. When NOT to use
- Không dùng cho các lỗi logic lập trình (ví dụ: truyền vào tham số null - dùng `NullPointerException`).
- Không dùng cho các lỗi định dạng dữ liệu không liên quan đến việc truyền tải (ví dụ: parse JSON lỗi - dùng `JsonParseException`).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Đảm bảo tính an toàn (robustness) vì lỗi I/O buộc phải được xử lý. | Làm code trở nên rườm rà với các khối try-catch/throws. |
| Cung cấp thông tin chi tiết về nguyên nhân lỗi từ hệ điều hành. | Đôi khi gây khó khăn khi làm việc với Functional Programming (Streams API). |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `RuntimeException` | Không bắt buộc xử lý, giúp code sạch hơn nhưng dễ bỏ sót lỗi nguy hiểm. |
| `UncheckedIOException` | Một wrapper của Java 8 giúp biến `IOException` thành unchecked exception (thường dùng trong Stream). |

## 9. How
```java
import java.io.FileReader;
import java.io.IOException;

public class ExceptionExample {
    public static void main(String[] args) {
        try (FileReader reader = new FileReader("data.txt")) {
            int data = reader.read();
            // Xử lý dữ liệu
        } catch (IOException e) {
            // Xử lý lỗi tập trung tại đây
            System.err.println("Lỗi I/O xảy ra: " + e.getMessage());
            e.printStackTrace();
        }
    }
}
```

## 10. Production concerns
### Scaling
Trong các hệ thống high-concurrency, việc ném ngoại lệ (throwing exceptions) liên tục có chi phí khá lớn về CPU do phải build stack trace. Cần tối ưu bằng cách kiểm tra điều kiện trước (nếu có thể).

### Failure
Cần phân biệt giữa lỗi có thể thử lại (Retryable) như `SocketTimeoutException` và lỗi vĩnh viễn như `FileNotFoundException`.

### Monitoring
Luôn log lại stack trace của `IOException` vào hệ thống giám sát (như ELK, Sentry) để biết được tình trạng sức khỏe của ổ cứng hoặc đường truyền mạng.

## 11. Common mistakes
- **Mistake**: Nuốt lỗi (Swallowing exception) - `catch (IOException e) { }` mà không làm gì cả.
  **Fix**: Ít nhất phải log lỗi hoặc ném lại dưới dạng một ngoại lệ khác có ý nghĩa hơn.

- **Mistake**: Không đóng tài nguyên khi xảy ra lỗi (trong các bản Java cũ).
  **Fix**: Luôn sử dụng `try-with-resources`.

## 12. Sample project
Viết một chương trình tải file từ một URL. Nếu xảy ra `IOException` do mất mạng, chương trình sẽ tự động thử lại (retry) tối đa 3 lần trước khi bỏ cuộc.
**Ràng buộc**: Phải phân biệt được lỗi mạng với các lỗi I/O khác.

## 13. Interview
### Core Q&A
1. **Q**: Tại sao `IOException` lại là checked exception?
   **A**: Vì lỗi I/O là những lỗi ngoại cảnh thường xuyên xảy ra và chương trình có khả năng phục hồi (như thử lại hoặc báo lỗi cho người dùng).
2. **Q**: Sự khác biệt giữa `IOException` và `FileNotFoundException` là gì?
   **A**: `FileNotFoundException` là lớp con của `IOException`, nó cụ thể hơn và chỉ xảy ra khi tệp tin không tồn tại trên đĩa.
3. **Q**: Làm thế nào để xử lý `IOException` trong Java 8 Stream?
   **A**: Có thể bọc nó trong một `RuntimeException` hoặc sử dụng một functional interface tùy chỉnh có khai báo `throws`.

### Scenario
**Tình huống**: Bạn đang ghi một file log rất quan trọng, nhưng đĩa cứng bị đầy (`Disk full`). Hệ thống ném ra `IOException`. Bạn xử lý thế nào để không mất dữ liệu?
**Trả lời**: Tôi sẽ catch `IOException`, thông báo cho hệ thống giám sát, và nếu có thể, tôi sẽ tạm thời chuyển hướng ghi dữ liệu vào một bộ nhớ đệm (buffer) hoặc một phân vùng đĩa khác làm fallback.

## 14. References
- Official Docs: [https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/io/IOException.html](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/io/IOException.html)
- GitHub Repo: OpenJDK Source code.

## 15. Real-world Code
- Xuất hiện trong mọi thao tác với Database (JDBC), File System (NIO), và Networking (Netty, OkHttp).

## 16. Community
- Stack Overflow: Tag [io-exception]
- Blog: Guide to Java Exceptions (Baeldung).
