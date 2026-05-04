---
created: 2026-05-04
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
  - "#topic/async"
related:
  - "[[Instant]]"
  - "[[ZoneOffset]]"
---

## 1. What
`LocalDateTime` là một lớp bất biến (immutable) trong gói `java.time` đại diện cho một cặp ngày-giờ (ví dụ: 2023-12-25T10:30:00). Nó **không chứa thông tin về múi giờ (timezone)** hay khoảng lệch so với giờ UTC (offset).

## 2. Why
Trước Java 8, việc xử lý ngày giờ rất phức tạp và dễ lỗi (như `java.util.Date` bị mutable). `LocalDateTime` ra đời để cung cấp một cách biểu diễn ngày giờ "thuần túy" - giống như cách con người nhìn vào lịch và đồng hồ treo tường. Nó tách biệt logic ngày giờ khỏi sự phức tạp của múi giờ khi không cần thiết.

## 3. Mental Model
Hãy tưởng tượng `LocalDateTime` như một cái **"Đồng hồ treo tường và Tờ lịch"** trong phòng bạn. Nó cho bạn biết bây giờ là 8 giờ sáng ngày 1/1. Tuy nhiên, nếu bạn gọi cho một người bạn ở London và nói "Bây giờ là 8 giờ sáng", họ sẽ hiểu nhầm vì đó chỉ là giờ *tại phòng bạn*. Nó mô tả một thời điểm "địa phương" mà không có ngữ cảnh toàn cầu.

## 4. Where it fits
Nó nằm ở tầng ứng dụng, dùng để hiển thị hoặc lưu trữ các mốc thời gian không phụ thuộc múi giờ:
`LocalDateTime -> + ZoneId/ZoneOffset -> ZonedDateTime / OffsetDateTime -> Instant`

## 5. When to use
- Khi lưu trữ các mốc thời gian mang tính chất "kế hoạch" trong tương lai (ví dụ: "Tiệc sinh nhật lúc 19:00 ngày 20/10").
- Khi làm việc với các hệ thống không quan tâm đến múi giờ (ví dụ: thời gian mở cửa cửa hàng cố định).
- Khi hiển thị dữ liệu cho người dùng mà bạn đã biết chắc chắn họ đang ở cùng múi giờ với hệ thống.

## 6. When NOT to use
- Khi lưu trữ các mốc thời gian "thực tế đã xảy ra" (Event timestamps) như thời gian tạo đơn hàng, thời gian log lỗi (trường hợp này dùng `Instant`).
- Khi cần tính toán khoảng cách thời gian giữa hai người ở hai múi giờ khác nhau.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Dễ đọc, gần gũi với tư duy con người. | Không thể đại diện cho một thời điểm tuyệt đối trên dòng thời gian toàn cầu. |
| Bất biến (Immutable) và Thread-safe. | Có thể gây lỗi nghiêm trọng nếu dùng để tính toán logic xuyên quốc gia. |
| API phong phú, dễ cộng/trừ ngày giờ. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `Instant` | Thời điểm tuyệt đối (UTC), tốt nhất cho lưu trữ DB và log. |
| `ZonedDateTime` | Ngày-giờ đầy đủ thông tin múi giờ (ví dụ: Asia/Ho_Chi_Minh). |
| `OffsetDateTime` | Ngày-giờ kết hợp với khoảng lệch UTC cố định. |

## 9. How
```java
import java.time.LocalDateTime;
import java.time.Month;

public class DateTimeExample {
    public static void main(String[] args) {
        // 1. Lấy thời gian hiện tại
        LocalDateTime now = LocalDateTime.now();
        System.out.println("Bây giờ: " + now);

        // 2. Khởi tạo một thời điểm cụ thể
        LocalDateTime christmas = LocalDateTime.of(2023, Month.DECEMBER, 25, 20, 0);

        // 3. Cộng trừ thời gian (trả về đối tượng mới)
        LocalDateTime nextWeek = now.plusWeeks(1);
        
        // 4. Lấy từng phần
        int hour = now.getHour();
        Month month = now.getMonth();
    }
}
```

## 10. Production concerns
### Scaling
Trong các hệ thống phân tán toàn cầu, tránh dùng `LocalDateTime` để truyền dữ liệu giữa các microservices. Hãy dùng định dạng ISO-8601 kèm múi giờ hoặc dùng Long (milliseconds).

### Failure
Lỗi phổ biến nhất là lưu `LocalDateTime` xuống Database mà Server và DB nằm ở hai múi giờ khác nhau, dẫn đến dữ liệu bị lệch khi đọc lên.

### Monitoring
Kiểm tra cấu hình múi giờ của JVM (`-Duser.timezone`) khi ứng dụng sử dụng `LocalDateTime.now()` để đảm bảo kết quả như mong đợi.

## 11. Common mistakes
- **Mistake**: Dùng `LocalDateTime` để lưu "Created At" trong database.
  **Fix**: Luôn dùng `Instant` hoặc `OffsetDateTime` cho các mốc thời gian lịch sử.

- **Mistake**: So sánh hai `LocalDateTime` mà không quan tâm chúng thuộc múi giờ nào.
  **Fix**: Chỉ so sánh khi chắc chắn chúng cùng một ngữ cảnh địa phương.

## 12. Sample project
Thiết lập một hệ thống nhắc lịch hẹn bác sĩ. Người dùng nhập ngày giờ khám. Hệ thống lưu dưới dạng `LocalDateTime` và chỉ gửi thông báo dựa trên giờ địa phương của phòng khám đó.

## 13. Interview
### Core Q&A
1. **Q**: `LocalDateTime` có chứa thông tin múi giờ không?
   **A**: Không. Nó chỉ chứa Ngày và Giờ.
2. **Q**: Tại sao `LocalDateTime` lại bất biến (immutable)?
   **A**: Để đảm bảo an toàn trong môi trường đa luồng (Thread-safe) và tránh các lỗi thay đổi dữ liệu ngoài ý muốn (side effects).
3. **Q**: Làm thế nào để chuyển `LocalDateTime` sang `Instant`?
   **A**: Cần cung cấp thêm một `ZoneOffset`. Ví dụ: `localDateTime.toInstant(ZoneOffset.UTC)`.

### Scenario
**Tình huống**: Bạn đang viết app báo thức. User đặt báo thức lúc 6:00 sáng. Bạn nên lưu kiểu dữ liệu nào?
**Trả lời**: Tôi dùng `LocalDateTime`. Vì dù User có mang điện thoại đi du lịch sang nước khác, họ vẫn muốn báo thức kêu lúc 6:00 sáng theo giờ địa phương nơi họ đang đứng, chứ không phải 6:00 sáng theo múi giờ cũ.

## 14. References
- Official Docs: [LocalDateTime API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/time/LocalDateTime.html)

## 15. Real-world Code
- Dùng trong các Entity của JPA khi mapping với cột `DATETIME` hoặc `TIMESTAMP` (nếu không cần timezone).

## 16. Community
- Reddit: r/java
- Stack Overflow: Tag [localdatetime]
- Blog: "Java 8 Date-Time API Guide" (Baeldung).
