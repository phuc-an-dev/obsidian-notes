---
created: 2026-05-04
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
  - "#topic/async"
related:
  - "[[Instant]]"
  - "[[LocalDateTime]]"
---

## 1. What
`ZoneOffset` là một lớp trong gói `java.time` đại diện cho khoảng chênh lệch thời gian cố định so với múi giờ chuẩn UTC (Greenwich). Ví dụ: `+07:00` (giờ Việt Nam) hoặc `-05:00` (giờ New York).

## 2. Why
Thời gian toàn cầu được lấy mốc là UTC. Tuy nhiên, mỗi vùng lãnh thổ lại có quy định về giờ giấc riêng. `ZoneOffset` giúp máy tính hiểu được mối liên hệ giữa một giờ địa phương và giờ quốc tế. Khác với `ZoneId` (như "Asia/Ho_Chi_Minh"), `ZoneOffset` là một con số **cố định**, không thay đổi theo quy tắc giờ mùa hè (DST - Daylight Saving Time).

## 3. Mental Model
Hãy tưởng tượng `ZoneOffset` như một cái **"Nút chỉnh đồng hồ nhanh/chậm"**. Khi bạn đi du lịch từ London (UTC) sang Việt Nam, bạn phải vặn đồng hồ của mình **nhanh hơn 7 tiếng (+07:00)**. Cái nút vặn này chính là `ZoneOffset`. Nó chỉ đơn thuần là phép cộng/trừ thời gian để khớp với một vị trí cụ thể.

## 4. Where it fits
Nó là thành phần cầu nối để chuyển đổi giữa thời gian địa phương và thời gian tuyệt đối:
`LocalDateTime + ZoneOffset = OffsetDateTime -> Instant`

## 5. When to use
- Khi cần chuyển đổi một `LocalDateTime` sang `Instant`.
- Khi làm việc với các hệ thống/database yêu cầu khoảng lệch UTC cố định thay vì tên múi giờ.
- Khi xử lý dữ liệu từ các thiết bị/logs ghi nhận thời gian kèm offset (ví dụ: `2023-05-04T10:00:00+07:00`).

## 6. When NOT to use
- Khi ứng dụng cần xử lý quy tắc giờ mùa hè (DST). Ví dụ: Ở London, mùa hè là `+01:00`, mùa đông là `+00:00`. Trong trường hợp này, hãy dùng `ZoneId` thay vì cố định một `ZoneOffset`.
- Khi hiển thị múi giờ cho người dùng cuối (hãy dùng tên vùng như "Hanoi" thay vì "+07:00").

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Đơn giản, tính toán cực nhanh (chỉ là cộng/trừ số). | Không tự động cập nhật khi chính phủ thay đổi quy tắc múi giờ. |
| Phù hợp cho việc lưu trữ dữ liệu lịch sử (vì offset đã xảy ra là cố định). | Không phản ánh được ngữ cảnh địa lý thực tế (nhiều vùng có cùng offset). |
| Chuẩn hóa theo ISO-8601. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `ZoneId` | Thông minh hơn, chứa toàn bộ lịch sử thay đổi múi giờ và quy tắc DST của một vùng. |
| `ZoneRules` | Lớp bên dưới của `ZoneId` dùng để lấy offset tại một thời điểm cụ thể. |

## 9. How
```java
import java.time.ZoneOffset;
import java.time.LocalDateTime;
import java.time.Instant;

public class ZoneOffsetExample {
    public static void main(String[] args) {
        // 1. Khởi tạo từ chuỗi hoặc số giờ
        ZoneOffset vnOffset = ZoneOffset.of("+07:00");
        ZoneOffset nyOffset = ZoneOffset.ofHours(-5);
        ZoneOffset utc = ZoneOffset.UTC;

        // 2. Dùng để chuyển LocalDateTime sang Instant
        LocalDateTime local = LocalDateTime.of(2023, 5, 4, 10, 0);
        Instant instant = local.toInstant(vnOffset);
        
        System.out.println("Local: " + local);
        System.out.println("Instant (UTC): " + instant); // Sẽ là 03:00 UTC
    }
}
```

## 10. Production concerns
### Scaling
`ZoneOffset` là bất biến và thread-safe, nên bạn có thể khai báo các hằng số dùng chung trong toàn bộ hệ thống (ví dụ `Constants.DEFAULT_OFFSET = ZoneOffset.of("+07:00")`).

### Failure
Tránh việc hard-code `ZoneOffset` trong code nếu ứng dụng của bạn phục vụ người dùng ở nhiều quốc gia. Hãy lấy offset từ request của client hoặc cấu hình của User.

### Monitoring
Log lại thông tin offset khi lưu dữ liệu thời gian để dễ dàng điều tra các lỗi lệch giờ (Time drift).

## 11. Common mistakes
- **Mistake**: Nghĩ rằng `+07:00` luôn tương đương với "Asia/Ho_Chi_Minh".
  **Fix**: Hiện tại thì đúng, nhưng trong lịch sử hoặc tương lai, một `ZoneId` có thể thay đổi offset của nó, còn `ZoneOffset` thì không.

- **Mistake**: Dùng `ZoneOffset` cho các sự kiện xảy ra ở tương lai xa tại các vùng có DST.
  **Fix**: Dùng `ZoneId` để đảm bảo khi đến ngày đó, hệ thống sẽ tự lấy đúng offset theo quy tắc mùa.

## 12. Sample project
Viết một hàm convert dữ liệu từ một file log cũ (không có timezone, chỉ có offset đi kèm). Hàm nhận vào `LocalDateTime` và `String offset`, trả về `Instant` để lưu vào database trung tâm.

## 13. Interview
### Core Q&A
1. **Q**: Sự khác biệt giữa `ZoneOffset` và `ZoneId` là gì?
   **A**: `ZoneOffset` là một khoảng lệch thời gian cố định (ví dụ +7). `ZoneId` là định danh của một vùng địa lý (ví dụ Asia/Saigon), nó chứa một tập hợp các `ZoneOffset` thay đổi theo thời gian (do DST hoặc luật pháp).
2. **Q**: `ZoneOffset.UTC` tương đương với giá trị nào?
   **A**: Tương đương với `ZoneOffset.ofHours(0)`.
3. **Q**: Có thể tạo `ZoneOffset` từ số giây không?
   **A**: Có, sử dụng `ZoneOffset.ofTotalSeconds(int)`.

### Scenario
**Tình huống**: Bạn đang viết code cho một hệ thống IoT, thiết bị gửi dữ liệu kèm theo số phút lệch so với UTC. Bạn làm thế nào để tạo `Instant` từ dữ liệu này?
**Trả lời**: Tôi dùng `ZoneOffset.ofTotalSeconds(minutes * 60)` để tạo đối tượng `ZoneOffset`, sau đó dùng nó kết hợp với ngày giờ nhận được để tạo `Instant`.

## 14. References
- Official Docs: [ZoneOffset API](https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/time/ZoneOffset.html)

## 15. Real-world Code
- Dùng trong định dạng JSON ISO-8601: `2023-05-04T10:00:00Z` (Z chính là UTC offset).

## 16. Community
- Stack Overflow: Tag [zoneoffset], [java-time]
- Blog: "Dealing with Time Zones in Java" (Baeldung).
