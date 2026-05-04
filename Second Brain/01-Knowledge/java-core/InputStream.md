---
created: 2026-05-04
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
  - "#topic/http"
related:
  - "[[IOException]]"
  - "[[ByteArrayInputStream]]"
---

## 1. What
`java.io.InputStream` là một lớp trừu tượng (abstract class) đại diện cho một luồng nhập dữ liệu dưới dạng byte (input stream of bytes). Nó là lớp cha tối cao của tất cả các lớp đọc dữ liệu nhị phân trong Java IO.

## 2. Why
Trước khi có `InputStream`, việc đọc dữ liệu từ file, mạng, hay bộ nhớ được thực hiện bằng các phương thức riêng lẻ và không nhất quán. `InputStream` ra đời để cung cấp một giao diện (interface) chung: "Tôi không quan tâm dữ liệu đến từ đâu, tôi chỉ quan tâm đến việc đọc từng byte một từ nó". Điều này cho phép các thư viện có thể viết code xử lý dữ liệu mà không cần biết nguồn gốc dữ liệu là gì.

## 3. Mental Model
Hãy tưởng tượng `InputStream` như một cái **"Vòi hút (Drinking Straw)"**. Đầu kia của vòi có thể cắm vào một ly nước (File), một bình sữa (Network), hay một bể bơi (Memory). Công việc của bạn chỉ là **"Hút (read)"** từng ngụm một qua cái vòi đó cho đến khi hết nước.

## 4. Where it fits
Nó là gốc của cây phân cấp các luồng nhập byte:
`InputStream (Abstract) -> [FileInputStream, ByteArrayInputStream, FilterInputStream, PipedInputStream]`

## 5. When to use
- Khi cần đọc dữ liệu nhị phân (hình ảnh, âm thanh, tệp thực thi).
- Khi viết các hàm xử lý dữ liệu chung chung (như bộ giải mã, bộ kiểm tra MIME) mà nguồn dữ liệu có thể thay đổi.
- Khi làm việc với các hệ thống cũ (Legacy) hoặc các giao thức mạng cấp thấp.

## 6. When NOT to use
- Khi đọc dữ liệu dạng văn bản (text). Trong trường hợp này, hãy dùng `java.io.Reader` (như `FileReader`, `BufferedReader`) để xử lý đúng bảng mã ký tự (encoding).
- Khi cần truy cập ngẫu nhiên (Random Access) vào các vị trí khác nhau trong file (dùng `RandomAccessFile`).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Tính đa hình cao, dễ dàng thay thế nguồn dữ liệu. | Đọc byte-by-byte rất chậm nếu không có buffer. |
| API đơn giản, dễ hiểu. | Không hỗ trợ xử lý Unicode/Charset một cách tự nhiên. |
| Hỗ trợ đánh dấu (marking) và quay lại (resetting) ở một số lớp con. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `Reader` | Chuyên dụng cho text, xử lý char thay vì byte. |
| `Channel` (NIO) | Hiệu suất cao hơn, hỗ trợ Non-blocking nhưng phức tạp hơn. |
| `Scanner` | Tiện lợi để đọc và parse các kiểu dữ liệu cơ bản từ stream. |

## 9. How
```java
import java.io.InputStream;
import java.io.FileInputStream;
import java.io.IOException;

public class InputStreamExample {
    public static void main(String[] args) {
        // Sử dụng FileInputStream (lớp con của InputStream)
        try (InputStream is = new FileInputStream("image.png")) {
            int byteData;
            // Đọc từng byte cho đến khi hết (-1)
            while ((byteData = is.read()) != -1) {
                // Xử lý byteData
            }
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}
```

## 10. Production concerns
### Scaling
Việc đọc từng byte một bằng `read()` là một thảm họa về hiệu năng do mỗi lần gọi là một system call. Luôn luôn sử dụng `BufferedInputStream` bọc ngoài hoặc đọc theo mảng `read(byte[] b)`.

### Failure
Luôn đảm bảo đóng stream trong khối `finally` hoặc sử dụng `try-with-resources` để tránh rò rỉ file descriptor.

### Monitoring
Theo dõi số lượng byte đã đọc và tốc độ đọc (throughput) để phát hiện các bottleneck trong I/O.

## 11. Common mistakes
- **Mistake**: Quên đóng `InputStream` dẫn đến "Too many open files".
  **Fix**: Sử dụng `try-with-resources`.

- **Mistake**: Dùng `InputStream` để đọc file `.txt` có tiếng Việt (UTF-8) dẫn đến lỗi font.
  **Fix**: Chuyển sang dùng `InputStreamReader` bọc ngoài.

## 12. Sample project
Tạo một "Byte Filter" nhận vào một `InputStream` bất kỳ và đếm xem có bao nhiêu byte có giá trị là 0 (null byte) trong luồng đó.
**Ràng buộc**: Phải dùng buffer để đảm bảo tốc độ xử lý nhanh.

## 13. Interview
### Core Q&A
1. **Q**: Tại sao `read()` trả về `int` thay vì `byte`?
   **A**: Vì `byte` trong Java có dải từ -128 đến 127. `read()` cần trả về giá trị từ 0 đến 255 cho dữ liệu thực tế, và dùng giá trị **-1** để báo hiệu kết thúc luồng.
2. **Q**: `available()` có đảm bảo trả về chính xác tổng số byte trong stream không?
   **A**: Không. Nó chỉ ước lượng số byte có thể đọc được ngay lập tức mà không bị "block".
3. **Q**: Sự khác biệt giữa `InputStream` và `Reader` là gì?
   **A**: `InputStream` làm việc với đơn vị byte (8-bit), phù hợp cho binary. `Reader` làm việc với đơn vị char (16-bit), phù hợp cho text và encoding.

### Scenario
**Tình huống**: Bạn được yêu cầu viết một hàm tính mã băm (MD5) cho một tệp tin. Tham số đầu vào nên là gì để hàm này linh hoạt nhất?
**Trả lời**: Tham số nên là `InputStream`. Như vậy hàm có thể tính MD5 cho file trên đĩa, cho mảng byte trong RAM, hoặc thậm chí cho dữ liệu đang tải về từ URL mà không cần viết lại logic.

## 14. References
- Official Docs: [https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/io/InputStream.html](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/io/InputStream.html)
- GitHub Repo: OpenJDK source.

## 15. Real-world Code
- `System.in` là một `InputStream` kinh điển để nhận dữ liệu từ bàn phím.
- ServletRequest.getInputStream() dùng để đọc body của request.

## 16. Community
- Reddit: r/java
- Stack Overflow: Tag [inputstream]
- Blog: Java IO Tutorial (Jenkov).
