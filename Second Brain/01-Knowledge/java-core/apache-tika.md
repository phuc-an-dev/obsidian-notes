---
created: 2026-05-04
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/java"
  - "#topic/file-handling"
related:
  - "[[file-upload-workflow]]"
---

## 1. What
**Apache Tika** là một thư viện mã nguồn mở (toolkit) thuộc Apache Software Foundation, được thiết kế để tự động nhận dạng (detection) và trích xuất nội dung, metadata từ hàng nghìn loại tệp tin khác nhau (như PDF, Office, hình ảnh, video). Phương thức `tika.detect()` là tính năng cốt lõi dùng để xác định MIME type của một tệp tin dựa trên nội dung thực tế thay vì chỉ dựa vào phần mở rộng (extension).

## 2. Why
Trong các ứng dụng web, việc chỉ dựa vào file extension (như `.jpg`, `.pdf`) để kiểm tra loại tệp là cực kỳ không an toàn vì người dùng có thể dễ dàng đổi tên một file thực thi nguy hiểm (`.exe`) thành `.jpg`. `tika.detect()` giải quyết vấn đề này bằng cách phân tích "vân tay" (magic bytes) và cấu trúc bên trong tệp để đưa ra kết quả chính xác về loại nội dung (Content-Type).

## 3. Mental Model
Hãy tưởng tượng Tika như một **"Giám định viên hải quan"**. Khi một kiện hàng đến, thay vì chỉ tin vào nhãn dán bên ngoài (file extension), ông ấy sẽ dùng các nghiệp vụ như soi đèn, kiểm tra cấu trúc kiện hàng, thậm chí mở hé để xem "chất liệu" bên trong (magic bytes) là gì. Chỉ sau khi kiểm tra kỹ lưỡng, ông ấy mới đóng dấu xác nhận đây thực sự là "vàng" hay là "hàng cấm" đội lốt.

## 4. Where it fits
Tika thường nằm ở lớp Validation trong luồng xử lý File Upload:
`User Upload -> Controller -> [Tika Detection] -> Validation Logic (Whitelist) -> Storage (S3/Disk)`

## 5. When to use
- Kiểm tra tính hợp lệ của tệp tin khi upload để ngăn chặn các cuộc tấn công upload file độc hại.
- Tự động phân loại tài liệu trong các hệ thống lưu trữ lớn.
- Trích xuất text từ các file phức tạp (như PDF, Docx) để phục vụ cho việc đánh chỉ mục tìm kiếm (Elasticsearch, Solr).

## 6. When NOT to use
- Khi hiệu năng là ưu tiên tuyệt đối và bạn tin tưởng hoàn toàn nguồn dữ liệu (Tika có chi phí tính toán và bộ nhớ nhất định).
- Đối với các loại tệp tin cực kỳ đơn giản hoặc định dạng tùy chỉnh (custom format) mà Tika chưa hỗ trợ.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Độ chính xác cực cao nhờ phân tích magic bytes. | Tăng thêm overhead về CPU và RAM khi xử lý tệp lớn. |
| Hỗ trợ thư viện định dạng khổng lồ (hơn 1400 loại). | Thêm dependency nặng vào project (nếu dùng bản full). |
| Thread-safe cho instance `Tika`. | Có thể bị đánh lừa bởi một số file "polyglot" tinh vi. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `Files.probeContentType` | Có sẵn trong JDK, nhưng độ chính xác thấp, phụ thuộc vào OS. |
| `MimetypesFileTypeMap` | Chủ yếu dựa vào extension, không an toàn cho bảo mật. |
| `jMimeMagic` | Một thư viện Java khác nhưng ít được cập nhật hơn Tika. |

## 9. How
```java
import org.apache.tika.Tika;
import java.io.File;
import java.io.IOException;
import java.io.InputStream;

public class TikaExample {
    public static void main(String[] args) throws IOException {
        Tika tika = new Tika();

        // 1. Detect từ File object
        File file = new File("example.pdf");
        String mimeType = tika.detect(file);
        System.out.println("Mime Type: " + mimeType);

        // 2. Detect từ InputStream (Khuyến nghị cho Web Upload)
        try (InputStream is = ... ) {
            String type = tika.detect(is);
        }

        // 3. Detect từ byte array
        byte[] data = ...;
        String type = tika.detect(data);
    }
}
```

## 10. Production concerns
### Scaling
Instance của lớp `Tika` là thread-safe, do đó bạn có thể khởi tạo một lần (Singleton/Bean) và dùng chung cho toàn bộ ứng dụng để tiết kiệm tài nguyên.

### Failure
Nếu tệp tin bị hỏng (corrupted) hoặc không xác định được, Tika thường trả về `application/octet-stream`. Cần có logic xử lý fallback cho trường hợp này.

### Monitoring
Cần log lại thời gian xử lý `detect()` đối với các tệp tin có kích thước lớn để tránh gây bottleneck cho hệ thống.

## 11. Common mistakes
- **Mistake**: Tạo mới một instance `new Tika()` cho mỗi lần request upload.
  **Fix**: Sử dụng Singleton pattern hoặc quản lý Tika instance như một Spring Bean để tái sử dụng.

- **Mistake**: Quên đóng `InputStream` sau khi truyền vào `tika.detect(is)`.
  **Fix**: Tika không tự động đóng stream cho bạn, hãy sử dụng try-with-resources.

## 12. Sample project
Xây dựng một API Upload ảnh bằng Spring Boot, sử dụng Tika để đảm bảo rằng ngay cả khi người dùng upload một file `.exe` đã đổi tên thành `.png`, hệ thống vẫn nhận diện đúng và từ chối lưu trữ.
**Ràng buộc**: Không được sử dụng bất kỳ thư viện nào khác ngoài Apache Tika Core.

## 13. Interview
### Core Q&A
1. **Q**: `tika.detect()` hoạt động dựa trên cơ chế nào chủ yếu?
   **A**: Nó kết hợp nhiều kỹ thuật: kiểm tra "magic bytes" (những byte đầu tiên của file), phân tích metadata bên trong và đôi khi là cấu trúc tổng thể của tệp.
2. **Q**: Tại sao không nên dùng `Files.probeContentType` thay cho Tika?
   **A**: Vì `Files.probeContentType` thường phụ thuộc vào cài đặt của hệ điều hành và registry, dẫn đến kết quả không nhất quán giữa Windows và Linux.
3. **Q**: Tika có tốn nhiều bộ nhớ khi xử lý file lớn không?
   **A**: Với `detect()`, Tika chỉ đọc một lượng nhỏ byte đầu tiên nên khá tiết kiệm. Tuy nhiên, nếu dùng để trích xuất text (parse full content), nó sẽ tốn nhiều RAM hơn.

### Scenario
**Tình huống**: Bạn đang làm hệ thống ngân hàng, cho phép khách hàng upload bản sao ID dạng PDF/Image. Một hacker cố tình upload một shell script độc hại nhưng đặt tên là `id_card.jpg`. Bạn sẽ dùng Tika như thế nào để chặn?
**Trả lời**: Tôi sẽ lấy `InputStream` từ tệp tin, truyền vào `tika.detect(inputStream)`. Nếu kết quả trả về không nằm trong whitelist (`image/jpeg`, `image/png`, `application/pdf`), tôi sẽ lập tức hủy request và log lại cảnh báo bảo mật.

## 14. References
- Official Docs: [https://tika.apache.org/](https://tika.apache.org/)
- GitHub Repo: [https://github.com/apache/tika](https://github.com/apache/tika)
- Spec / RFC: RFC 2045, RFC 2046 (MIME types)
- Changelog: [https://tika.apache.org/2.9.1/changelog.html](https://tika.apache.org/2.9.1/changelog.html)

## 15. Real-world Code
- Tích hợp Tika trong các dự án CMS lớn như Alfresco hay Apache Jackrabbit để đánh chỉ mục nội dung.
- Các gateway bảo mật tệp tin thường nhúng Tika để lọc nội dung.

## 16. Community
- Reddit: r/java, r/softwareengineering
- Stack Overflow: Tag [apache-tika]
- Blog: Baeldung (Apache Tika Guide)
- Talk: "Content Analysis with Apache Tika" - ApacheCon.
