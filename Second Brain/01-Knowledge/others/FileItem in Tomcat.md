---
created: 2026-05-04
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/java"
  - "#topic/file-handling"
related:
  - "[[file-upload-workflow]]"
  - "[[InputStream]]"
  - "[[apache-tika]]"
---

## 1. What
`FileItem` là một interface trong gói `org.apache.tomcat.util.http.fileupload` (thực chất là một bản fork nội bộ của Apache Commons FileUpload trong Tomcat). Nó đại diện cho một mục (item) nhận được trong một `multipart/form-data` POST request, có thể là một tệp tin (file binary) hoặc một trường dữ liệu văn bản (form field).

## 2. Why
Giao thức HTTP truyền thống chỉ gửi dữ liệu dạng key-value đơn giản. Khi cần gửi các tệp tin lớn hoặc nhiều tệp tin cùng lúc, trình duyệt sử dụng định dạng `multipart`. `FileItem` cung cấp một cách tiếp cận trừu tượng để lập trình viên có thể truy cập vào dữ liệu này mà không cần quan tâm đến việc dữ liệu đang được lưu tạm trong bộ nhớ (RAM) hay trên đĩa cứng (Disk).

## 3. Mental Model
Hãy tưởng tượng `FileItem` như một **"Kiện hàng bưu kiện"** nằm trong một xe tải lớn (HTTP Request). Xe tải này chở nhiều kiện hàng khác nhau. Một kiện hàng có thể là một bức thư nhỏ (form field như `username`) hoặc một thùng gỗ lớn (file hình ảnh). Mỗi kiện hàng đều có nhãn dán (metadata như `fieldName`, `contentType`) và bạn có thể mở nó ra để lấy nội dung bên trong.

## 4. Where it fits
Nó nằm ở tầng thấp của quá trình xử lý Request trong các ứng dụng web dựa trên Servlet/Tomcat:
`HTTP Request -> Tomcat Multipart Parser -> List<FileItem> -> Spring MultipartFile (Wrapper)`

## 5. When to use
- Khi bạn viết các ứng dụng Servlet thuần (không dùng framework) và cần xử lý upload file.
- Khi cần can thiệp sâu vào quá trình parse file của Tomcat để tối ưu hóa hiệu năng.
- Khi làm việc với các hệ thống legacy vẫn sử dụng thư viện Apache Commons FileUpload.

## 6. When NOT to use
- Khi sử dụng Spring Boot/Spring MVC (nên dùng `MultipartFile` vì nó thân thiện và dễ dùng hơn).
- Khi sử dụng Servlet 3.0+ (nên dùng `javax.servlet.http.Part` - tiêu chuẩn của Java EE).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Kiểm soát cực tốt việc lưu trữ (Memory vs Disk). | API cũ, rườm rà (phải dùng `FileItemFactory`). |
| Hỗ trợ stream dữ liệu để xử lý các file cực lớn. | Gắn chặt với implementation của Tomcat/Commons. |
| Cho phép lấy metadata (size, content-type) dễ dàng. | Dễ gây rò rỉ bộ nhớ/ổ cứng nếu không dọn dẹp file tạm. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `MultipartFile` | Wrapper của Spring, cực kỳ phổ biến và dễ dùng cho DI. |
| `javax.servlet.http.Part` | Tiêu chuẩn Java EE, không cần thư viện ngoài. |
| `Streaming API` | (Cũng của Commons FileUpload) Hiệu năng cao nhất, không tạo file tạm. |

## 9. How
```java
import org.apache.tomcat.util.http.fileupload.FileItem;
import org.apache.tomcat.util.http.fileupload.disk.DiskFileItemFactory;
import org.apache.tomcat.util.http.fileupload.servlet.ServletFileUpload;
import java.io.InputStream;
import java.util.List;

// Ví dụ trong một Servlet
public void doPost(HttpServletRequest request, HttpServletResponse response) {
    if (ServletFileUpload.isMultipartContent(request)) {
        DiskFileItemFactory factory = new DiskFileItemFactory();
        ServletFileUpload upload = new ServletFileUpload(factory);
        
        try {
            List<FileItem> items = upload.parseRequest(new ServletRequestContext(request));
            for (FileItem item : items) {
                if (!item.isFormField()) { // Nếu là file
                    String fileName = item.getName();
                    long size = item.getSize();
                    // Lấy InputStream để xử lý hoặc lưu trữ
                    try (InputStream is = item.getInputStream()) {
                        // Xử lý với InputStream (ví dùng Tika để detect)
                    }
                } else { // Nếu là field thường
                    String fieldName = item.getFieldName();
                    String value = item.getString();
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

## 10. Production concerns
### Scaling
`DiskFileItemFactory` cho phép cấu hình `sizeThreshold`. Nếu file nhỏ hơn ngưỡng này, nó nằm trong RAM. Nếu lớn hơn, nó tự động ghi ra Disk. Cần cấu hình hợp lý để tránh treo RAM hoặc làm chậm IO đĩa.

### Failure
Nếu quá trình upload bị ngắt quãng, các file tạm (.tmp) có thể vẫn còn nằm trên ổ cứng. Cần cơ chế dọn dẹp (Lombok/Commons thường có `FileCleaningTracker`).

### Monitoring
Theo dõi thư mục chứa file tạm (thường là `/tmp` hoặc `java.io.tmpdir`) để đảm bảo không bị đầy ổ cứng.

## 11. Common mistakes
- **Mistake**: Không kiểm tra `item.isFormField()` dẫn đến cố gắng đọc stream của một chuỗi text như một file.
  **Fix**: Luôn dùng `if (item.isFormField())` để phân loại.

- **Mistake**: Quên gọi `item.delete()` sau khi xử lý xong (trong một số trường hợp thủ công).
  **Fix**: Đảm bảo file tạm được xóa sau khi đã move vào kho lưu trữ chính thức.

## 12. Sample project
Tạo một Servlet nhận file video lớn, sử dụng `FileItem` để stream dữ liệu trực tiếp lên Amazon S3 mà không lưu toàn bộ video vào RAM của Server.
**Ràng buộc**: RAM sử dụng không được vượt quá 10MB dù file video nặng 1GB.

## 13. Interview
### Core Q&A
1. **Q**: `FileItem.getInputStream()` và `FileItem.get()` khác nhau như thế nào?
   **A**: `getInputStream()` cung cấp luồng dữ liệu (phù hợp cho file lớn), còn `get()` trả về toàn bộ mảng byte (chỉ nên dùng cho file nhỏ).
2. **Q**: Làm thế nào để giới hạn kích thước file upload bằng `FileItem`?
   **A**: Cấu hình `ServletFileUpload.setFileSizeMax(long)` cho từng file hoặc `setSizeMax(long)` cho toàn bộ request.
3. **Q**: Tại sao Tomcat lại có gói `org.apache.tomcat.util.http.fileupload` thay vì dùng bản gốc của Apache Commons?
   **A**: Để tránh xung đột phiên bản (JAR hell) và để Tomcat tự chủ động trong việc quản lý Multipart request mà không phụ thuộc vào thư viện bên ngoài của người dùng.

### Scenario
**Tình huống**: Bạn nhận thấy server thường xuyên bị đầy ổ cứng ở thư mục `/tmp` sau khi triển khai tính năng upload ảnh. Bạn sẽ kiểm tra gì ở `FileItem`?
**Trả lời**: Tôi sẽ kiểm tra xem `DiskFileItemFactory` đã được cấu hình đúng chưa, và quan trọng nhất là liệu các đối tượng `FileItem` có được dọn dẹp (gọi `delete()`) sau khi upload thành công hoặc thất bại hay không.

## 14. References
- Official Docs: [Apache Tomcat Documentation](https://tomcat.apache.org/tomcat-9.0-doc/api/org/apache/tomcat/util/http/fileupload/FileItem.html)
- GitHub Repo: [Tomcat Source Code](https://github.com/apache/tomcat)

## 15. Real-world Code
- Lớp `CommonsMultipartFile` của Spring Framework sử dụng chính `FileItem` bên dưới để thực thi các phương thức của mình.

## 16. Community
- Stack Overflow: Tag [tomcat], [commons-fileupload]
- Blog: "Handling File Uploads in Java" (Baeldung).
