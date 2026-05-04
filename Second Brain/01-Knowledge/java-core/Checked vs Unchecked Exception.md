---
created: 2026-05-04
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
  - "#topic/error-handling"
related:
  - "[[IOException]]"
  - "[[ParseException]]"
---

## 1. What
Trong Java, Exception được chia thành hai loại chính dựa trên thời điểm chúng được kiểm tra:
- **Checked Exception**: Là các ngoại lệ mà trình biên dịch (Compiler) bắt buộc bạn phải xử lý (catch) hoặc khai báo (throws) ngay khi viết code. Chúng thường là các lớp con của `Exception` nhưng không thuộc `RuntimeException`.
- **Unchecked Exception**: Là các ngoại lệ không bị bắt buộc kiểm tra tại thời điểm biên dịch. Chúng bao gồm các lớp con của `RuntimeException` và `Error`. Lỗi này thường xảy ra do logic lập trình sai.

## 2. Why
Java phân chia hai loại này để phân biệt giữa **"Sự cố có thể dự đoán và phục hồi"** và **"Lỗi logic của lập trình viên"**:
- **Checked**: Buộc lập trình viên phải chuẩn bị cho các tình huống ngoài ý muốn từ môi trường bên ngoài (như mất mạng, file không tồn tại).
- **Unchecked**: Giúp code gọn gàng hơn bằng cách không bắt buộc catch các lỗi mà đáng lẽ lập trình viên phải tránh ngay từ đầu (như chia cho 0, truy cập mảng quá chỉ số).

## 3. Mental Model
Hãy tưởng tượng bạn đang **"Lái một chiếc máy bay"**:
- **Checked Exception** giống như một **"Bảng danh sách kiểm tra an toàn (Checklist)"**: Trước khi cất cánh, phi công *bắt buộc* phải kiểm tra xăng, động cơ, thời tiết. Nếu không kiểm tra, máy bay không được phép cất cánh. Đây là những thứ bạn biết chắc chắn có thể xảy ra sự cố và phải có phương án dự phòng.
- **Unchecked Exception** giống như một **"Cơn đau tim đột ngột"** của phi công: Đây là sự cố không ai mong đợi và không có bảng checklist nào bắt bạn phải kiểm tra mỗi giây. Nó thường là hệ quả của một vấn đề sức khỏe tiềm ẩn (code lỗi) mà bạn lẽ ra phải chữa trị trước đó.

## 4. Where it fits
Nằm trong hệ thống phân cấp `Throwable`:
`Throwable -> Error (Unchecked) | Exception -> RuntimeException (Unchecked) | Other Exceptions (Checked)`

## 5. When to use
- **Dùng Checked Exception**: Khi bạn viết một thư viện/phương thức tương tác với các nguồn lực bên ngoài (File, Network, DB) mà người gọi phương thức có khả năng thực hiện hành động phục hồi (như thử lại hoặc báo lỗi cho user).
- **Dùng Unchecked Exception**: Khi bạn phát hiện ra một tình huống mà đáng lẽ không bao giờ nên xảy ra nếu code được viết đúng (ví dụ: tham số truyền vào là null khi không được phép).

## 6. When NOT to use
- **Không dùng Checked Exception**: Cho các lỗi mà người gọi không thể làm gì để khắc phục (ví dụ: lỗi cấu hình hệ thống nghiêm trọng). Việc bắt họ catch chỉ làm code thêm rác.
- **Không dùng Unchecked Exception**: Cho các lỗi I/O thông thường mà ứng dụng cần phải xử lý để đảm bảo tính ổn định.

## 7. Trade-offs
| Loại | Pros | Cons |
|------|------|------|
| **Checked** | Đảm bảo an toàn, buộc phải xử lý lỗi. | Làm code rườm rà (Checked Exception Boilerplate), khó dùng với Stream/Lambda. |
| **Unchecked** | Code sạch, linh hoạt, phù hợp với phong cách lập trình hiện đại. | Dễ bỏ sót các lỗi quan trọng dẫn đến crash ứng dụng khi chạy (Runtime). |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `Optional` | Tránh dùng exception cho các trường hợp "không tìm thấy dữ liệu" thông thường. |
| `Result/Either Pattern` | (Phổ biến trong Functional Programming) Trả về một đối tượng chứa hoặc kết quả hoặc lỗi thay vì ném exception. |

## 9. How
```java
// 1. Checked Exception: Bắt buộc xử lý
public void readFile(String path) throws IOException { // Khai báo
    if (path == null) throw new IOException("Path invalid");
}

// 2. Unchecked Exception: Không bắt buộc xử lý
public int divide(int a, int b) {
    if (b == 0) throw new ArithmeticException("Cannot divide by zero"); // Unchecked
    return a / b;
}

// 3. Cách dùng thực tế
try {
    readFile("data.txt");
} catch (IOException e) {
    // Xử lý bắt buộc
}
```

## 10. Production concerns
### Scaling
Checked Exception có thể gây ra hiện tượng "Exception Pollution" - khi một phương thức ở tầng thấp ném exception và hàng loạt phương thức tầng trên phải khai báo `throws`, tạo nên sự phụ thuộc chặt chẽ.

### Failure
Trong các kiến trúc Microservices, thường ưu tiên chuyển đổi Checked Exception thành Unchecked Exception ở tầng biên (boundary) để tránh việc lan truyền các exception đặc thù của thư viện ra toàn hệ thống.

### Monitoring
Theo dõi tỉ lệ Unchecked Exception (như `NullPointerException`) để đánh giá chất lượng code và tìm kiếm các lỗi logic tiềm ẩn.

## 11. Common mistakes
- **Mistake**: Nuốt lỗi (Empty catch block) đối với Checked Exception chỉ để làm compiler "hết báo đỏ".
  **Fix**: Luôn log hoặc ném lại (rethrow) dưới dạng Unchecked Exception nếu không thể xử lý.

- **Mistake**: Lạm dụng Checked Exception cho mọi tình huống.
  **Fix**: Chỉ dùng khi thực sự cần người gọi phải có trách nhiệm xử lý.

## 12. Sample project
Xây dựng một lớp `EmailValidator`. Nếu email sai định dạng, ném `IllegalArgumentException` (Unchecked). Nếu không thể kết nối tới Mail Server để verify, ném `MailServerConnectionException` (Checked).

## 13. Interview
### Core Q&A
1. **Q**: Sự khác biệt lớn nhất giữa Checked và Unchecked là gì?
   **A**: Checked được kiểm tra bởi Compiler (phải catch/throws), Unchecked thì không.
2. **Q**: Tại sao `RuntimeException` và các con của nó lại là Unchecked?
   **A**: Vì chúng đại diện cho các lỗi lập trình (bugs) mà đáng lẽ có thể tránh được bằng cách kiểm tra logic, không nên bắt code phải try-catch khắp nơi.
3. **Q**: Có nên tạo Checked Exception tùy chỉnh không?
   **A**: Có, nếu bạn muốn người sử dụng API của bạn bắt buộc phải xử lý một tình huống nghiệp vụ quan trọng.

### Scenario
**Tình huống**: Bạn viết một hàm lấy thông tin User từ Database. Nếu không thấy User, bạn nên ném Checked hay Unchecked Exception?
**Trả lời**: Tùy ngữ cảnh. Nếu việc không thấy User là một "lỗi" nghiêm trọng cần xử lý đặc biệt, dùng Checked. Tuy nhiên, xu hướng hiện nay là dùng `Optional<User>` để tránh ném exception cho những trường hợp dữ liệu không tồn tại thông thường.

## 14. References
- Official Docs: [Java Language Specification - Exceptions](https://docs.oracle.com/javase/specs/jls/se17/html/jls-11.html)
- Clean Code (Robert C. Martin): Chương về Error Handling.

## 15. Real-world Code
- `IOException`, `SQLException` là các Checked Exception kinh điển.
- `NullPointerException`, `IndexOutOfBoundsException` là các Unchecked Exception kinh điển.

## 16. Community
- Reddit: r/java (Tranh luận về việc Checked Exception có còn cần thiết hay không).
- Stack Overflow: Tag [checked-exceptions] vs [unchecked-exceptions].
- Blog: "The Case Against Checked Exceptions" - Anders Hejlsberg.
