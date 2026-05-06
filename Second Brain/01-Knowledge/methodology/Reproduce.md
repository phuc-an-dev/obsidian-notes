---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/others"
  - "#topic/error-handling"
related:
  - "[[tdd.md]]"
  - "[[github-actions-ci.md]]"
---

## 1. What
Reproduce (Tái hiện lỗi) là quá trình lặp lại các bước cụ thể để một lỗi (bug) hoặc một hành vi không mong muốn của phần mềm xảy ra một cách nhất quán. Đây là bước quan trọng nhất và bắt buộc phải thực hiện trước khi bắt tay vào sửa bất kỳ lỗi nào.

## 2. Why
Nếu không thể tái hiện lỗi, lập trình viên sẽ rơi vào tình trạng "Sửa mò" (Guesswork). Reproduce giúp:
- **Xác nhận sự tồn tại của lỗi**: Đảm bảo lỗi đó là có thật, không phải do cấu hình sai nhất thời.
- **Tìm ra nguyên nhân gốc rễ (Root Cause)**: Hiểu rõ điều kiện nào kích hoạt lỗi.
- **Kiểm chứng giải pháp**: Sau khi sửa, nếu không còn tái hiện được lỗi theo các bước cũ, chứng tỏ giải pháp đã hiệu quả.
- **Tiết kiệm thời gian**: Tránh việc sửa nhầm chỗ hoặc gây ra lỗi mới (regression).

## 3. Mental Model
Hãy tưởng tượng Reproduce giống như một **"Buổi thực nghiệm hiện trường"** của cảnh sát:
- Khi có một vụ án (Bug) xảy ra, cảnh sát phải dựng lại hiện trường với đúng các nhân vật, công cụ và trình tự thời gian.
- Nếu dựng lại mà vụ án không xảy ra, nghĩa là họ chưa hiểu đúng bản chất vấn đề.
- Chỉ khi nào họ có thể làm cho vụ án xảy ra lại y hệt, họ mới biết chính xác hung thủ là ai.

## 4. Where it fits
Vị trí trong quy trình sửa lỗi:
`Bug Report -> Reproduce (Confirm Bug) -> Root Cause Analysis -> Fix -> Verify -> Regression Test`

## 5. When to use
- Ngay sau khi nhận được thông báo lỗi từ người dùng hoặc hệ thống monitoring.
- Khi muốn viết một Unit Test hoặc Integration Test mới để bao phủ một trường hợp lỗi vừa phát hiện (TDD approach).

## 6. When NOT to use
- Đối với các lỗi "Heisenbug" (lỗi biến mất khi cố gắng quan sát hoặc debug) - trường hợp này cần dùng log phân tích thay vì tái hiện trực tiếp.
- Khi chi phí để tái hiện lỗi quá cao hoặc nguy hiểm (ví dụ: lỗi làm sập toàn bộ hạ tầng mạng của quốc gia).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Đảm bảo lỗi được sửa triệt để 100%. | Tốn nhiều thời gian cho việc thiết lập môi trường giống thật. |
| Tạo ra bộ test case giá trị cho tương lai. | Khó thực hiện với các lỗi liên quan đến race condition hoặc tải cao. |
| Tăng uy tín của lập trình viên đối với QA/User. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Log Analysis | Phân tích dấu vết cũ thay vì chạy lại, phù hợp cho lỗi hiếm gặp. |
| Remote Debugging | Debug trực tiếp trên máy người dùng, nhanh nhưng rủi ro bảo mật. |

## 9. How
Quy trình Reproduce chuẩn:
1. **Thu thập thông tin**: Version ứng dụng, OS, Trình duyệt, Dữ liệu đầu vào, Logs.
2. **Thiết lập môi trường**: Tạo một môi trường bị cô lập (Sandbox) giống hệt môi trường bị lỗi.
3. **Thực hiện các bước (Steps to reproduce)**:
   - Bước 1: Login với user X.
   - Bước 2: Nhấn vào nút Y.
   - Bước 3: Nhập giá trị Z.
4. **Quan sát kết quả**: Ghi lại mã lỗi hoặc thông báo nhận được.
5. **Đơn giản hóa (Isolation)**: Loại bỏ các bước không ảnh hưởng để tìm ra chuỗi hành động ngắn nhất gây lỗi.

## 10. Production concerns
### Data Privacy
Khi lấy dữ liệu từ Production về máy local để reproduce, phải đảm bảo đã che mờ (masking) các thông tin nhạy cảm của người dùng thực.

### Version Matching
Bắt buộc phải reproduce trên đúng commit hash hoặc tag mà môi trường Production đang chạy.

## 11. Common mistakes
- Mistake: Sửa lỗi ngay khi vừa đọc report mà chưa thử chạy lại.
  Fix: Luôn tuân thủ quy tắc "No repro, no fix".

- Mistake: Reproduce trên môi trường development có dữ liệu khác hoàn toàn với production.
  Fix: Cố gắng đồng bộ schema và các điều kiện logic của dữ liệu.

## 12. Sample project
Viết một script Bash hoặc một bộ test case JUnit thực hiện đúng 3 bước gửi request đến API để làm cho database bị tràn bộ nhớ (OOM), sau đó kiểm chứng sau khi fix code thì script này không còn gây lỗi nữa.

## 13. Interview
### Core Q&A
1. Q: Bạn làm gì nếu không thể reproduce được lỗi người dùng báo?
   A: Tôi sẽ kiểm tra log chi tiết hơn, yêu cầu người dùng quay clip hoặc screenshot, và kiểm tra xem có sự khác biệt nào về môi trường (mạng, thiết bị) mà tôi chưa tính tới không.

2. Q: Tại sao cần tìm ra "Minimal steps to reproduce"?
   A: Để loại bỏ nhiễu, giúp xác định chính xác dòng code hoặc module gây lỗi, giúp việc sửa lỗi tập trung và nhanh chóng hơn.

### Scenario
"QA báo một lỗi nghiêm trọng nhưng bạn thử mãi không được. Bạn trả lời QA thế nào?"
-> Trả lời: Tôi sẽ không đóng ticket ngay mà gửi lại phản hồi: "Tôi đã thử theo các bước mô tả nhưng chưa tái hiện được. Bạn có thể cho tôi biết thêm về [Dữ liệu đầu vào cụ thể/Log trình duyệt] không? Chúng ta có thể thảo luận trực tiếp để cùng tái hiện không?"

## 14. References
- Bug Reporting Best Practices: [Mozilla Bug Writing Guidelines](https://developer.mozilla.org/en-US/docs/Mozilla/Developer_guide/How_to_Submit_a_Bug_Report)
- Debugging Book: [The Art of Debugging](https://nostarch.com/debugging.htm)

## 15. Real-world Code
Nghiên cứu các "Issue Template" trên các repo lớn như React hoặc Spring để thấy cách họ bắt buộc người dùng cung cấp thông tin reproduce.

## 16. Community
- Reddit: r/programming.
- Stack Overflow: Tag [debugging].
