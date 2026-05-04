---
created: 2026-05-04
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/file-handling"
related:
  - "[[FileItem in Tomcat]]"
  - "[[InputStream]]"
  - "[[apache-tika]]"
---

## 1. What
`MultipartFile` là một interface trong Spring Framework đại diện cho một tệp tin được upload lên trong một multipart request. Nó cung cấp các phương thức tiện lợi để truy cập nội dung, tên tệp, và metadata của file mà không cần quan tâm đến công nghệ bên dưới (Tomcat, Jetty, hay Commons FileUpload).

## 2. Why
Việc xử lý upload file ở tầng Servlet thuần rất phức tạp (phải parse request, quản lý file tạm). `MultipartFile` ra đời để đơn giản hóa quá trình này, cho phép lập trình viên nhận file trực tiếp như một tham số trong phương thức của Controller.

## 3. Mental Model
Hãy tưởng tượng `MultipartFile` như một **"Hộp quà bọc sẵn"** được đặt trên bàn của bạn. Bạn không cần biết hộp quà đó được vận chuyển như thế nào, qua xe tải hay máy bay. Bạn chỉ cần mở hộp (getInputStream), xem nhãn (getOriginalFilename), hoặc chuyển nó sang một cái hộp khác (transferTo).

## 4. Where it fits
Nó là lớp trừu tượng cao nhất trong Spring MVC cho việc upload file:
`HTTP Request -> MultipartResolver -> MultipartFile -> Controller Business Logic`

## 5. When to use
- Trong mọi ứng dụng Spring Boot / Spring MVC khi cần nhận file từ client.
- Khi cần thực hiện các thao tác cơ bản như lưu file xuống đĩa, kiểm tra kích thước, hoặc kiểm tra loại tệp (Content-Type).

## 6. When NOT to use
- Khi bạn không dùng Spring Framework.
- Khi cần xử lý luồng dữ liệu (streaming) cực lớn mà không muốn Spring tự động parse và tạo file tạm (trường hợp này dùng streaming API của Tomcat).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| API cực kỳ đơn giản, tích hợp sẵn với Spring DI. | Spring mặc định parse toàn bộ request trước khi vào Controller (có thể gây trễ). |
| Dễ dàng viết Unit Test (dùng `MockMultipartFile`). | Phụ thuộc vào cấu hình `MultipartResolver`. |
| Hỗ trợ lưu file nhanh chóng qua `transferTo()`. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `javax.servlet.http.Part` | Tiêu chuẩn Servlet, Spring `MultipartFile` có thể wrap lớp này. |
| `FileItem` | (Tomcat/Commons) Tầng thấp hơn, linh hoạt hơn nhưng khó dùng hơn. |

## 9. How
```java
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;
import java.io.IOException;
import java.nio.file.*;

@RestController
public class FileUploadController {

    @PostMapping("/upload")
    public String handleFileUpload(@RequestParam("file") MultipartFile file) {
        if (file.isEmpty()) return "File rỗng";

        try {
            // 1. Lấy metadata
            String fileName = file.getOriginalFilename();
            String contentType = file.getContentType();
            
            // 2. Lưu file xuống đĩa (cách nhanh nhất)
            Path path = Paths.get("uploads/" + fileName);
            file.transferTo(path);

            // 3. Hoặc lấy InputStream để xử lý logic (như check virus, detect type)
            // InputStream is = file.getInputStream();

            return "Upload thành công: " + fileName;
        } catch (IOException e) {
            return "Lỗi: " + e.getMessage();
        }
    }
}
```

## 10. Production concerns
### Scaling
Mặc định Spring Boot giới hạn dung lượng file (thường là 1MB). Cần cấu hình `spring.servlet.multipart.max-file-size` trong `application.properties` để phù hợp với nhu cầu.

### Failure
Nếu tệp tin vượt quá dung lượng cho phép, Spring sẽ ném ra `MaxUploadSizeExceededException` trước khi vào đến Controller. Cần dùng `@ControllerAdvice` để bắt lỗi này.

### Monitoring
Theo dõi thời gian upload và kích thước file để tối ưu hóa băng thông và lưu trữ.

## 11. Common mistakes
- **Mistake**: Sử dụng `file.getBytes()` cho các file cực lớn (vài GB) dẫn đến nổ RAM (OutOfMemory).
  **Fix**: Luôn dùng `file.getInputStream()` hoặc `file.transferTo()`.

- **Mistake**: Tin tưởng hoàn toàn vào `file.getContentType()` từ client gửi lên.
  **Fix**: Luôn dùng Apache Tika để kiểm tra lại nội dung thực tế của file.

## 12. Sample project
Tạo một hệ thống upload ảnh đại diện, sử dụng `MultipartFile`, sau đó dùng `ImageIO` để resize ảnh và lưu vào thư mục `static`.

## 13. Interview
### Core Q&A
1. **Q**: `MultipartFile.transferTo(File)` có ưu điểm gì so với việc tự đọc InputStream và ghi file?
   **A**: Nó cực kỳ tối ưu. Nếu file tạm đã được tạo trên đĩa bởi container (Tomcat), `transferTo` có thể chỉ đơn giản là thực hiện lệnh `rename` file, thay vì phải copy dữ liệu.
2. **Q**: Làm thế nào để nhận nhiều file cùng lúc trong Controller?
   **A**: Sử dụng `List<MultipartFile>` hoặc `MultipartFile[]` trong `@RequestParam`.
3. **Q**: Sự khác biệt giữa `getName()` và `getOriginalFilename()` là gì?
   **A**: `getName()` trả về tên của tham số trong form (ví dụ: "file"), còn `getOriginalFilename()` trả về tên tệp tin thực tế trên máy người dùng (ví dụ: "my_photo.jpg").

### Scenario
**Tình huống**: Khách hàng báo lỗi không thể upload file 5MB mặc dù code của bạn không có giới hạn gì. Bạn kiểm tra ở đâu?
**Trả lời**: Tôi sẽ kiểm tra file cấu hình (properties/yml). Trong Spring Boot, mặc định giới hạn file thường là 1MB. Tôi cần tăng `spring.servlet.multipart.max-file-size` và `spring.servlet.multipart.max-request-size`.

## 14. References
- Official Docs: [Spring MultipartFile API](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/web/multipart/MultipartFile.html)

## 15. Real-world Code
- Xuất hiện trong hầu hết các service xử lý tài liệu, hình ảnh trong các dự án Spring.

## 16. Community
- Reddit: r/springboot
- Stack Overflow: Tag [spring-mvc], [multipartfile]
- Blog: Baeldung (Guide to MultipartFile).
