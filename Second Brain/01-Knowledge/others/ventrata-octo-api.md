---
created: 2026-04-22
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/others"
  - "#topic/http"
related: "[[RESTful API]]"
---

## 1. What
Ventrata OCTO API là bản triển khai của tiêu chuẩn OCTO (Open Connectivity for Tourism), một giao diện lập trình ứng dụng mở dành riêng cho ngành công nghiệp tour, hoạt động và điểm tham quan. Nó cho phép các bên bán lại (Resellers/OTAs) kết nối và đồng bộ dữ liệu thời gian thực với hệ thống cung cấp của Ventrata.


## 2. Why
Trước khi có OCTO, mỗi nhà cung cấp dịch vụ du lịch (Supplier) sử dụng một API riêng biệt (Proprietary API). Việc này gây khó khăn cho các OTA khi muốn tích hợp với hàng trăm supplier khác nhau vì phải viết code riêng cho từng bên. OCTO ra đời để chuẩn hóa cách giao tiếp, giúp giảm chi phí phát triển và tăng tốc độ kết nối trong ngành du lịch.


## 3. Mental Model
Hãy tưởng tượng OCTO API như một bộ "phích cắm điện đa năng" toàn cầu. Cho dù bạn là thiết bị điện từ nước nào (OTA nào), chỉ cần bạn dùng chuẩn phích cắm OCTO, bạn có thể cắm vào bất kỳ ổ điện nào (Supplier nào) hỗ trợ chuẩn này mà không cần quan tâm mạng lưới điện bên trong họ vận hành ra sao.


## 4. Where it fits
Vị trí trong luồng dữ liệu:
Reseller (OTA) -> Ventrata OCTO API -> Ventrata Booking Engine -> Supplier Inventory.


## 5. When to use
- Khi bạn đang xây dựng một nền tảng đặt vé (Booking Platform) hoặc trang web du lịch cần tích hợp các sản phẩm từ các nhà cung cấp sử dụng phần mềm Ventrata.
- Khi muốn đồng bộ hóa danh sách sản phẩm, kiểm tra tình trạng chỗ trống (availability) và thực hiện đặt vé (booking) theo thời gian thực.


## 6. When NOT to use
- Khi bạn chỉ cần thông tin tĩnh (hình ảnh, mô tả) mà không cần đặt vé thời gian thực (nên dùng các phương thức export file nếu số lượng quá lớn).
- Nếu nhà cung cấp không hỗ trợ chuẩn OCTO (phải dùng API riêng của họ).


## 7. Trade-offs
| Pros | Cons |
|------|------|
| Chuẩn hóa cao, dễ dàng mở rộng sang các đối tác khác cũng dùng OCTO. | Bị giới hạn bởi các tính năng mà tiêu chuẩn OCTO định nghĩa. |
| Hỗ trợ nhiều Capability tùy chọn (Pricing, Pickups, Waivers). | Yêu cầu quản lý Header phức tạp (Octo-Capabilities). |
| Documentation rõ ràng, có môi trường test (EdinExplore). | Quy trình chứng thực (Certification) trước khi live khá nghiêm ngặt. |


## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Ventrata Legacy API | API cũ của Ventrata, không theo chuẩn OCTO, ít được khuyến khích hơn. |
| Rezdy API / Bokun API | Các đối thủ cạnh tranh, có chuẩn API riêng hoặc cũng đang dần hỗ trợ OCTO. |


## 9. How
Để bắt đầu, cần gửi Header `Octo-Capabilities` để khai báo các tính năng mà Reseller hỗ trợ.

```bash
# Lấy danh sách sản phẩm
curl -X GET "https://api.ventrata.com/octo/products" \
     -H "Authorization: Bearer YOUR_API_KEY" \
     -H "Octo-Capabilities: octo/pricing,octo/content" \
     -H "Accept: application/json"
```

Cấu trúc Booking Request tối thiểu:
```json
{
  "uuid": "random-uuid-v4",
  "productId": "product-id",
  "optionId": "option-id",
  "availabilityId": "availability-id",
  "unitItems": [
    {
      "unitId": "adult"
    }
  ]
}
```


## 10. Production concerns
### Scaling
Nên sử dụng cơ chế Cache cho danh sách sản phẩm (`/products`) vì dữ liệu này ít thay đổi thường xuyên. Tuy nhiên, tuyệt đối không cache dữ liệu Availability quá lâu.

### Failure
Hệ thống phải xử lý được các mã lỗi OCTO tiêu chuẩn (ví dụ: `INVALID_AVAILABILITY_ID`, `UNAVAILABLE`). Khi API Ventrata down, hệ thống Reseller nên có cơ chế fallback hoặc thông báo bảo trì cho user.

### Monitoring
Theo dõi Header `X-Rate-Limit` để tránh bị khóa tài khoản do gọi API quá dày đặc.


## 11. Common mistakes
- Mistake: Quên gửi Header `Octo-Capabilities` khiến API không trả về dữ liệu giá hoặc nội dung mở rộng.
  Fix: Luôn khai báo đầy đủ các capabilities mà code của bạn có thể xử lý (ví dụ: `octo/pricing`).

- Mistake: Nhầm lẫn giữa `productId` và `optionId`. Trong OCTO, một Product có thể có nhiều Options (ví dụ: Tour đi bộ có Option Sáng và Option Chiều).
  Fix: Luôn kiểm tra cấu trúc phân cấp trong response của `/products`.


## 12. Sample project
Xây dựng một Worker đơn giản bằng Node.js để đồng bộ sản phẩm từ Ventrata về database nội bộ mỗi 24h, sử dụng capability `octo/content` để lấy đầy đủ hình ảnh và mô tả.


## 13. Interview
### Core Q&A
1. Q: OCTO API khác gì với các API du lịch thông thường?
   A: OCTO là một tiêu chuẩn chung được nhiều bên chấp nhận, không phải API độc quyền. Nó tập trung vào tính tương thích (interoperability) giữa các hệ thống khác nhau trong ngành.

2. Q: Tại sao cần `availabilityId` khi đặt vé?
   A: Vì trong ngành du lịch, tình trạng chỗ trống thay đổi theo từng giây. `availabilityId` đại diện cho một khung giờ cụ thể và trạng thái của nó tại thời điểm kiểm tra.

### Scenario
"Khách hàng báo giá trên website của bạn thấp hơn giá tại bước checkout của Ventrata. Bạn sẽ kiểm tra gì?"
-> Kiểm tra xem đã gửi Header `octo/pricing` chưa, và đảm bảo đã handle đúng `currency` và các loại thuế (taxes) trả về từ OCTO.


## 14. References
- Official Docs: [https://docs.ventrata.com/](https://docs.ventrata.com/)
- GitHub Repo: [https://github.com/octotravel](https://github.com/octotravel)
- Spec / RFC: [https://docs.octo.travel/](https://docs.octo.travel/)
- Changelog: Kiểm tra tại trang docs chính thức của Ventrata.


## 15. Real-world Code
Nên tham khảo các bộ SDK OCTO trên GitHub của cộng đồng du lịch để thấy cách họ handle việc retry và mapping dữ liệu.


## 16. Community
- Reddit: r/traveltech
- Stack Overflow: Tag #ventrata #octo-api
- Blog: Ventrata Engineering Blog
- Talk: Các hội thảo về Arival (Tours & Activities event).
