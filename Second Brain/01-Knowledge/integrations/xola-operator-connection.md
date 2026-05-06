---
created: 2026-04-22
tags:
  - "#type/tutorial"
  - "#status/draft"
  - "#lang/others"
  - "#topic/http"
related: "[[ventrata-octo-api]]"
---

## 1. What
Xola Operator Connection là một cấu hình tích hợp trên nền tảng Ventrata, cho phép kết nối trực tiếp với hệ thống của các nhà cung cấp (Operators) đang sử dụng phần mềm Xola. Tích hợp này sử dụng chuẩn OCTO API để đồng bộ hóa dữ liệu.


## 2. Why
Trước khi có kết nối này, Ventrata Resellers phải cập nhật chỗ trống (availability) và giá (pricing) thủ công cho các sản phẩm của Xola Operator, dẫn đến nguy cơ overbooking cao. Việc kết nối tự động hóa toàn bộ quy trình từ kiểm tra chỗ đến đặt vé và hủy vé.


## 3. Mental Model
Hãy coi Ventrata như một "đại lý bán lẻ" và Xola như một "kho hàng". Xola Operator Connection chính là "đường dây nóng" kết nối trực tiếp kho hàng với quầy bán lẻ. Mỗi khi có khách mua tại quầy, hệ thống sẽ tự động gọi điện vào kho để kiểm tra và lấy hàng ngay lập tức.


## 4. Where it fits
Ventrata Dashboard (Mapping) -> OCTO API (Endpoint: `octo.xola.com`) -> Xola System.


## 5. When to use
- Khi bạn là một Reseller sử dụng Ventrata và muốn bán các tour/hoạt động từ một đối tác sử dụng Xola.
- Khi cần độ chính xác 100% về tình trạng chỗ trống thời gian thực.
- Khi muốn tự động hóa việc xuất vé (Voucher/Tickets) từ hệ thống Xola cho khách hàng.


## 6. When NOT to use
- Khi đối tác (Operator) không đồng ý chia sẻ API Key hoặc không cài đặt ứng dụng phân phối trên Xola App Store.
- Đối với các sản phẩm có tính chất "On Request" (cần xác nhận thủ công) mà Xola không hỗ trợ đồng bộ tự động.


## 7. Trade-offs
| Pros | Cons |
|------|------|
| Đồng bộ Real-time tuyệt đối về Availability và Pricing. | Phụ thuộc hoàn toàn vào uptime của API Xola. |
| Giảm thiểu rủi ro Overbooking và lỗi nhập liệu manual. | Quy trình thiết lập ban đầu cần sự phối hợp từ cả hai phía (Reseller & Operator). |
| Hỗ trợ cả giá Retail và giá Wholesale/Net. | Cần phải Mapping thủ công từng Option và Unit. |


## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Manual Inventory | Nhập tay số lượng chỗ. Dễ sai sót, tốn công sức. |
| CSV/Excel Export | Cập nhật định kỳ. Không phải thời gian thực. |


## 9. How
Quy trình thiết lập gồm 3 bước chính:

**Bước 1: Cấu hình Operator**
- Lấy **OCTO API Key** từ Xola Operator (họ cần cài đặt ứng dụng của bạn trên Xola App Store).
- Tại Ventrata: **Products -> Operators**, chọn Operator tương ứng.
- Phần **Backend Partner**, chọn OCTO và nhập API Key. Endpoint mặc định: `https://octo.xola.com/latest`.

**Bước 2: Mapping sản phẩm**
- Tại Ventrata: **Products -> Products**, chọn sản phẩm cần kết nối.
- Bật **Backend Connected**.
- Sang tab **Mappings**, chọn đúng Product, Option và Unit tương ứng từ danh sách dropdown của Xola.

**Bước 3: Kích hoạt đồng bộ**
- Chọn các tùy chọn: `Pull Backend Availability` và `Pull Backend Pricing`.
- Save và kiểm tra trạng thái kết nối.


## 10. Production concerns
### Scaling
Khi số lượng sản phẩm lớn, việc gọi API check availability liên tục có thể chạm giới hạn rate limit. Nên tận dụng cơ chế webhook nếu có để cập nhật thay đổi.

### Failure
Nếu API Xola không phản hồi, Ventrata sẽ không thể xác nhận booking. Cần có quy trình xử lý lỗi để thông báo cho khách hàng hoặc chuyển sang chế độ booking thủ công tạm thời.

### Monitoring
Kiểm tra log mapping thường xuyên để đảm bảo các `UnitId` (Người lớn, Trẻ em) giữa hai hệ thống vẫn khớp nhau.


## 11. Common mistakes
- Mistake: Mapping sai Unit (ví dụ: Unit "Adult" của Ventrata map nhầm vào "Child" của Xola).
  Fix: Luôn kiểm tra kỹ dropdown list trong tab Mappings và thực hiện một booking test ngay sau khi map.

- Mistake: Quên bật `Pull Backend Availability` khiến hệ thống vẫn dùng số lượng tồn kho ảo của Ventrata.
  Fix: Đảm bảo các checkbox đồng bộ đã được tích chọn sau khi hoàn tất mapping.


## 12. Sample project
Thực hiện kết nối thử nghiệm với một Xola Operator giả lập, map 1 sản phẩm có 2 options (Sáng/Chiều) và thực hiện luồng: Check Availability -> Create Booking -> Cancel Booking để kiểm tra tính toàn vẹn dữ liệu.


## 13. Interview
### Core Q&A
1. Q: Làm sao để Ventrata biết cần gọi đến đâu để lấy dữ liệu Xola?
   A: Thông qua OCTO API Key và Endpoint được cấu hình trong phần Backend Partner của Operator.

2. Q: Nếu Xola thay đổi giá, Ventrata có cập nhật theo không?
   A: Có, nếu tùy chọn `Pull Backend Pricing` được bật, Ventrata sẽ lấy giá trực tiếp từ Xola mỗi khi có yêu cầu.

### Scenario
"Một Operator báo rằng họ đã thêm một loại vé mới (ví dụ: Senior) trên Xola nhưng bạn không thấy trong Ventrata. Bạn làm gì?"
-> Vào tab Mappings của sản phẩm, nhấn refresh để fetch lại danh sách Unit từ Xola, sau đó thực hiện mapping cho loại vé mới đó.


## 14. References
- Official Tutorial: [https://support.ventrata.com/en/articles/9532178-xola-operator-connection](https://support.ventrata.com/en/articles/9532178-xola-operator-connection)
- Xola App Store Docs: [https://xola.com/app-store](https://xola.com/app-store)


## 15. Real-world Code
Thường không can thiệp code trực tiếp mà qua giao diện Dashboard. Tuy nhiên, có thể debug thông qua việc quan sát các network request đến endpoint `octo.xola.com`.


## 16. Community
- Ventrata Support Center.
- Xola Developer Community.
