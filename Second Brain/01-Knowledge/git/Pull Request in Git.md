---
created: 2026-04-30
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/version-control"
related:
  - "[[Code Review and Merge Process]]"
  - "[[Git vs GitHub]]"
  - "[[GitHub Actions]]"
---

## 1. What
Pull Request (PR) - hay còn gọi là Merge Request trong GitLab - là một cơ chế cho phép nhà phát triển thông báo cho các thành viên khác trong nhóm về những thay đổi mà họ đã đẩy lên một nhánh trong repository trên GitHub. Nó là một không gian thảo luận chuyên biệt để xem xét, thảo luận và điều chỉnh code trước khi gộp vào nhánh chính.

## 2. Why
Nếu không có PR, việc gộp mã sẽ diễn ra âm thầm và thiếu kiểm soát. PR ra đời để đảm bảo chất lượng code thông qua việc kiểm tra chéo (Code Review), phát hiện lỗi sớm, chia sẻ kiến thức giữa các thành viên và lưu lại lịch sử thảo luận tại sao một thay đổi lại được thực hiện.

## 3. Mental Model
Hãy tưởng tượng bạn là một kiến trúc sư đang thiết kế một căn phòng mới cho một ngôi nhà.
- **Commit**: Là các bản vẽ chi tiết từng cái cửa, cái cửa sổ.
- **Pull Request**: Là việc bạn đặt tất cả bản vẽ đó lên bàn của chủ nhà và nói: "Tôi đã thiết kế xong căn phòng này, mời mọi người xem qua và cho ý kiến trước khi chúng ta thực sự xây dựng nó".
- **Review**: Là lúc mọi người cùng chỉ vào bản vẽ và góp ý.

## 4. Where it fits
Nằm giữa giai đoạn lập trình và tích hợp:
Feature Branch -> Push to Remote -> **Pull Request** -> Review & Feedback -> Approval -> Merge.

## 5. When to use
- Luôn sử dụng khi làm việc nhóm để đề xuất thay đổi.
- Kể cả khi làm việc một mình, PR giúp bạn tự review lại code của mình một cách khách quan trước khi merge.
- Khi đóng góp vào các dự án mã nguồn mở (Open Source).

## 6. When NOT to use
- Các thay đổi cực kỳ nhỏ như sửa lỗi chính tả trong tài liệu (với một số team cho phép commit trực tiếp).
- Trong các tình huống cứu vãn hệ thống khẩn cấp mà quy trình PR thông thường quá chậm (Emergency fixes).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Kiểm soát chất lượng code chặt chẽ | Làm chậm tốc độ gộp mã |
| Chia sẻ kiến thức và context dự án | Có thể gây áp lực cho người review (Reviewer fatigue) |
| Tự động hóa test ngay trên không gian PR | Đòi hỏi kỹ năng giao tiếp tốt để tránh xung đột cá nhân |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Direct Merge | Nhanh nhưng rủi ro cực cao, không có bước kiểm duyệt. |
| Pair Programming | Review diễn ra trực tiếp khi viết code, không cần PR nhưng tốn 2 người cho 1 task. |

## 9. How
Quy trình thực hiện một PR tốt:
1. **Tiêu đề**: Rõ ràng, phản ánh đúng tính năng (VD: `feat: add login with Google`).
2. **Mô tả**: Giải thích "Tại sao" thay đổi này cần thiết và "Cách thức" thực hiện.
3. **Ảnh chụp/Video**: Nếu là thay đổi về giao diện (UI).
4. **Checklist**: Tự kiểm tra các tiêu chuẩn của dự án.

```markdown
## Mô tả
Thêm tính năng đăng nhập bằng Google để tăng tỉ lệ chuyển đổi người dùng.

## Thay đổi chính
- Tích hợp Google OAuth SDK.
- Thêm API endpoint `/api/auth/google`.
- Cập nhật giao diện trang Login.

## Lưu ý cho Reviewer
Cần kiểm tra kỹ phần bảo mật Token trong middleware.
```

## 10. Production concerns
### Scaling
Với các dự án lớn, cần sử dụng file `PULL_REQUEST_TEMPLATE.md` để chuẩn hóa nội dung PR cho mọi thành viên.

### Failure
Nếu PR bị từ chối do lỗi logic, đừng tạo PR mới. Hãy tiếp tục commit sửa lỗi lên cùng một nhánh đó, PR sẽ tự động cập nhật.

### Monitoring
Sử dụng các công cụ như CodeClimate hoặc SonarQube để tự động chấm điểm PR ngay khi vừa khởi tạo.

## 11. Common mistakes
- Mistake: Tạo PR quá lớn (hàng chục file, hàng nghìn dòng code).
  Fix: Chia nhỏ task thành các PR nhỏ hơn, dễ review hơn.

- Mistake: Không tự review code của mình trước khi gửi PR.
  Fix: Luôn đọc lại diff trên GitHub để phát hiện lỗi ngớ ngẩn trước khi làm phiền đồng nghiệp.

## 12. Sample project
Tạo một PR mẫu, gắn tag đồng nghiệp, thực hiện phản hồi lại một comment góp ý và cuối cùng là merge sau khi được approved.

## 13. Interview
### Core Q&A
1. Q: "Draft Pull Request" là gì và khi nào nên dùng?
   A: Draft PR là trạng thái PR chưa sẵn sàng để merge. Nên dùng khi bạn muốn chia sẻ code đang làm dở để xin ý kiến sớm về kiến trúc hoặc để chạy các test tự động mà không làm phiền reviewer phải vào đánh giá chính thức.

2. Q: Bạn làm gì nếu PR của bạn mãi không được ai review?
   A: Tôi sẽ chủ động ping đồng nghiệp qua Slack/Teams, hoặc nêu ra trong buổi họp Daily Standup. Nếu cần thiết, tôi sẽ nhờ Leader phân bổ người review phù hợp.

### Scenario
Tình huống: Reviewer yêu cầu bạn thay đổi một logic mà bạn cho là cách của bạn tốt hơn. Bạn xử lý thế nào?
Trả lời: Tôi sẽ giải thích rõ lý do tại sao tôi chọn cách đó (về hiệu năng, độ sạch, hoặc logic nghiệp vụ). Nếu vẫn không đồng nhất, tôi sẽ đề xuất một buổi thảo luận ngắn 5-10 phút để hai bên cùng hiểu quan điểm của nhau thay vì tranh cãi qua lại trên PR.

## 14. References
- GitHub PR Guide: https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/about-pull-requests
- Writing a Great PR: https://www.atlassian.com/blog/git/written-unwritten-guide-pull-requests

## 15. Real-world Code
Hầu hết các dự án trong `Second Brain` của bạn nên tuân thủ quy trình PR này để duy trì tính chuyên nghiệp.

## 16. Community
- Reddit: r/softwareengineering (thảo luận về văn hóa review).
- Stack Overflow: Tag [pull-request].
