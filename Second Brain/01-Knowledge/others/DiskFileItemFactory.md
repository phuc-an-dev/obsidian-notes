---
created: 2026-05-04
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/java"
  - "#topic/file-handling"
related:
  - "[[FileItem in Tomcat]]"
---

## 1. What
`DiskFileItemFactory` là lớp thực thi (implementation) mặc định của `FileItemFactory` trong Tomcat. Nhiệm vụ chính của nó là quy định cách thức và nơi lưu trữ các `FileItem` (dữ liệu từ multipart request) trước khi chúng được xử lý chính thức bởi ứng dụng.

## 2. Why
Khi một tệp tin được upload, server không thể lúc nào cũng giữ toàn bộ dữ liệu trong RAM (vì sẽ gây tốn bộ nhớ) nhưng cũng không nên lúc nào cũng ghi ngay xuống đĩa (vì sẽ chậm). `DiskFileItemFactory` giải quyết vấn đề này bằng cách cung cấp một ngưỡng kích thước (threshold). Nếu file nhỏ, nó giữ trong RAM; nếu lớn, nó tự động tạo file tạm trên đĩa.

## 3. Mental Model
Hãy tưởng tượng `DiskFileItemFactory` như một **"Quản kho thông minh"**. Khi có hàng (data) về, nếu là hàng nhẹ (file nhỏ), ông ấy cầm trên tay (RAM) cho nhanh. Nếu là hàng nặng (file lớn), ông ấy mang vào kho tạm (Disk) để cất cho đỡ mỏi tay. Ông ấy cũng quyết định "kho tạm" nằm ở đâu và khi nào thì hàng được coi là "nặng".

## 4. Where it fits
Nó là cấu hình đầu vào cho quá trình parse request:
`Configuration (Threshold, TempDir) -> DiskFileItemFactory -> ServletFileUpload -> List<FileItem>`

## 5. When to use
- Khi sử dụng Tomcat/Commons FileUpload để xử lý upload file thủ công.
- Khi cần tinh chỉnh ngưỡng bộ nhớ (RAM usage) cho các luồng upload file.
- Khi cần chỉ định thư mục lưu trữ file tạm khác với thư mục mặc định của hệ thống.

## 6. When NOT to use
- Khi sử dụng Spring Boot với cấu hình mặc định (Spring đã tự quản lý việc này qua `MultipartProperties`).
- Khi sử dụng các giải pháp upload trực tiếp (Streaming API) không cần tạo file tạm.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Quản lý tài nguyên RAM hiệu quả, tránh OOM. | Gây tốn IO đĩa nếu threshold quá thấp. |
| Linh hoạt trong việc chọn thư mục lưu tạm. | Có thể để lại file rác nếu không dọn dẹp đúng cách. |
| Đã được kiểm chứng qua thời gian trong Tomcat. | API mang tính kế thừa (legacy), hơi rườm rà. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `MemoryFileItemFactory` | Luôn giữ mọi thứ trong RAM (nguy hiểm cho file lớn). |
| `StandardMultipartHttpServletRequest` | (Servlet 3.0) Dùng cơ chế nội bộ của container thay vì factory này. |

## 9. How
```java
import org.apache.tomcat.util.http.fileupload.disk.DiskFileItemFactory;
import java.io.File;

public class FactoryExample {
    public void setupFactory() {
        // 1. Khởi tạo factory
        DiskFileItemFactory factory = new DiskFileItemFactory();

        // 2. Thiết lập ngưỡng (ví dụ: 100KB)
        // File nhỏ hơn 100KB nằm trong RAM, lớn hơn ghi ra Disk
        factory.setSizeThreshold(1024 * 100);

        // 3. Thiết lập thư mục tạm
        File tempDir = new File("/data/tmp_uploads");
        if (!tempDir.exists()) tempDir.mkdirs();
        factory.setRepository(tempDir);
        
        // Sau đó truyền factory vào ServletFileUpload
        // ServletFileUpload upload = new ServletFileUpload(factory);
    }
}
```

## 10. Production concerns
### Scaling
Nếu hệ thống có hàng ngàn request upload đồng thời, giá trị `sizeThreshold` nhân với số lượng request sẽ là lượng RAM tối đa bị chiếm dụng. Cần tính toán kỹ số này.

### Failure
Nếu ổ cứng ở thư mục `repository` bị đầy, factory sẽ không thể tạo file tạm và ném ra lỗi khi parse request.

### Monitoring
Cần giám sát dung lượng đĩa của thư mục `repository` và số lượng file tạm đang tồn tại.

## 11. Common mistakes
- **Mistake**: Đặt `sizeThreshold` quá lớn (ví dụ vài trăm MB) dẫn đến treo server khi nhiều người upload cùng lúc.
  **Fix**: Đặt ngưỡng vừa phải (thường là 10KB - 1MB).

- **Mistake**: Không kiểm tra quyền ghi (write permission) của thư mục `repository`.
  **Fix**: Đảm bảo user chạy app có quyền ghi vào thư mục tạm.

## 12. Sample project
Thiết lập một hệ thống upload file cho một server có cấu hình RAM yếu (512MB). Cấu hình `DiskFileItemFactory` sao cho RAM dành cho upload không bao giờ vượt quá 50MB.

## 13. Interview
### Core Q&A
1. **Q**: `sizeThreshold` có ý nghĩa gì?
   **A**: Là ngưỡng kích thước (tính bằng byte). File có kích thước nhỏ hơn ngưỡng này sẽ được lưu hoàn toàn trong bộ nhớ, ngược lại sẽ được ghi vào file tạm trên đĩa.
2. **Q**: `setRepository(File)` dùng để làm gì?
   **A**: Để chỉ định thư mục mà factory sẽ dùng để tạo các file tạm khi dữ liệu vượt quá `sizeThreshold`.
3. **Q**: Có cần phải dọn dẹp file tạm sau khi dùng xong không?
   **A**: Có, thông qua `FileItem.delete()`. Tuy nhiên, factory này cũng có thể tích hợp với một tracker để tự động xóa khi đối tượng bị GC.

### Scenario
**Tình huống**: Bạn đổi thư mục tạm sang một ổ đĩa mạng (Network Drive) và thấy tốc độ upload giảm thảm hại. Tại sao?
**Trả lời**: Vì `DiskFileItemFactory` phải thực hiện ghi dữ liệu liên tục vào ổ đĩa mạng (thông qua network IO) mỗi khi file vượt quá ngưỡng RAM, dẫn đến độ trễ lớn hơn rất nhiều so với ổ đĩa cục bộ (Local Disk).

## 14. References
- Official Docs: [Apache Tomcat DiskFileItemFactory](https://tomcat.apache.org/tomcat-9.0-doc/api/org/apache/tomcat/util/http/fileupload/disk/DiskFileItemFactory.html)

## 15. Real-world Code
- Spring Framework sử dụng lớp này bên trong `CommonsMultipartResolver`.

## 16. Community
- Stack Overflow: Tag [commons-fileupload], [tomcat]
- Blog: "Configuring Multipart Uploads in Tomcat".
