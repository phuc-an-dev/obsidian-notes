---
created: 2026-04-30
tags:
  - "#type/pattern"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/version-control"
related:
  - "[[Pull Request in Git]]"
  - "[[Code Review and Merge Process]]"
  - "[[Branching Strategy in Git]]"
---

## 1. What
Team Collaboration Workflow in Git là quy trình phối hợp giữa các thành viên trong nhóm để cùng xây dựng một sản phẩm phần mềm. Quy trình này bao gồm các bước từ việc nhận nhiệm vụ, tạo nhánh, viết code, cho đến việc kiểm duyệt (review) và tích hợp mã nguồn vào dự án chung.

## 2. Why
Trong môi trường làm việc nhóm, nếu mỗi người tự ý đẩy code lên nhánh chính sẽ gây ra xung đột mã nguồn liên tục, phá hỏng tính ổn định của sản phẩm và khó kiểm soát chất lượng. Một workflow chuẩn giúp đảm bảo mọi dòng code đều được kiểm tra, giảm thiểu lỗi và giúp các thành viên hiểu được thay đổi của nhau.

## 3. Mental Model
Hãy tưởng tượng quy trình này như một "Tòa soạn báo":
- **Phóng viên (Developer)**: Viết bài trên bản thảo riêng (Feature Branch).
- **Biên tập viên (Reviewer)**: Đọc bản thảo, đưa ra nhận xét và yêu cầu sửa đổi (Code Review).
- **Tổng biên tập (Maintainer)**: Phê duyệt bản thảo cuối cùng để đưa vào số báo chính thức (Merge into Main/Develop).
Quy trình này đảm bảo không có bài viết kém chất lượng nào được xuất bản.

## 4. Where it fits
Nằm ở trung tâm của quá trình phát triển (SDLC):
Task Assignment -> Git Workflow -> CI/CD Pipeline -> Deployment.

## 5. When to use
- Luôn áp dụng khi dự án có từ 2 lập trình viên trở lên.
- Khi làm việc trong các dự án chuyên nghiệp đòi hỏi tính ổn định cao.
- Khi muốn xây dựng văn hóa chia sẻ kiến thức thông qua việc đọc code của nhau.

## 6. When NOT to use
- Dự án cá nhân (solo project) không cần quy trình Pull Request rườm rà.
- Các thay đổi khẩn cấp (Emergency Hotfix) trong một số ít trường hợp cực kỳ đặc biệt có thể bỏ qua một vài bước (nhưng vẫn cần commit đầy đủ).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Chất lượng code được đảm bảo qua Review | Tốn thêm thời gian cho việc chờ đợi review |
| Giảm thiểu rủi ro làm hỏng nhánh chính | Có thể gây ra "bottleneck" nếu Reviewer quá bận |
| Lịch sử dự án rõ ràng, dễ truy vết | Đòi hỏi tính kỷ luật cao từ mọi thành viên |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Pair Programming | Hai người cùng viết code trên một máy. Review diễn ra tức thì, không cần PR nhưng tốn tài nguyên nhân sự. |
| Trunk-based Development | Đẩy code trực tiếp vào main (với Automated Tests cực tốt). Nhanh hơn nhưng rủi ro cao hơn nếu test không phủ hết. |

## 9. How
### Quy trình 7 bước phối hợp nhóm chuẩn:
1. **Sync**: Cập nhật code mới nhất từ server trước khi làm: `git pull origin develop`.
2. **Branch**: Tạo nhánh tính năng mới: `git checkout -b feature/task-name`.
3. **Code**: Thực hiện công việc và commit cục bộ thường xuyên với message rõ ràng.
4. **Push**: Đẩy nhánh lên remote: `git push origin feature/task-name`.
5. **PR (Pull Request)**: Tạo PR trên GitHub/GitLab, mô tả rõ những gì đã làm.
6. **Review**: Chờ đồng nghiệp review, thảo luận và sửa đổi nếu có yêu cầu (Update commits).
7. **Merge**: Sau khi được Approved, tiến hành gộp vào nhánh chính và xóa nhánh feature.

## 10. Production concerns
### Scaling
Với team lớn, cần chia nhỏ các Pull Request (Atomic PRs) để Reviewer dễ dàng kiểm tra. Tránh các PR khổng lồ chứa hàng nghìn dòng code.

### Failure
Nếu một PR sau khi merge gây ra lỗi trên môi trường Test, cần thực hiện `git revert` ngay lập tức để giải phóng nhánh chính cho các task khác.

### Monitoring
Sử dụng GitHub Actions để tự động check Lint, chạy Unit Test ngay khi PR được tạo. PR chỉ được merge nếu tất cả các "Checks" đều xanh.

## 11. Common mistakes
- Mistake: Tạo PR quá lớn, bao gồm nhiều tính năng không liên quan.
  Fix: Tuân thủ nguyên tắc "Một PR - Một nhiệm vụ".

- Mistake: Tự ý merge code của mình mà không chờ review hoặc khi test đang fail.
  Fix: Thiết lập "Branch Protection Rules" để ngăn chặn việc merge trái phép.

## 12. Sample project
Thực hành quy trình "Collaborative Coding": Người A tạo PR -> Người B comment yêu cầu sửa code -> Người A update commit -> Người B approve -> Merge.

## 13. Interview
### Core Q&A
1. Q: Bạn làm gì nếu PR của bạn bị từ chối (Request Changes)?
   A: Tôi sẽ đọc kỹ các comment của Reviewer, thảo luận nếu có điểm chưa rõ. Sau đó, tôi thực hiện sửa đổi ngay trên nhánh đó và push lên. PR sẽ tự động cập nhật các commit mới để Reviewer kiểm tra lại.

2. Q: Làm thế nào để giải quyết xung đột (Conflict) khi tạo PR?
   A: Tôi sẽ cập nhật nhánh chính về máy (`git pull origin develop`), sau đó merge nhánh chính vào nhánh feature của mình (`git merge develop`) hoặc thực hiện rebase. Sau khi giải quyết xong conflict cục bộ và đảm bảo code vẫn chạy đúng, tôi sẽ push lên lại.

### Scenario
Tình huống: Bạn thấy một PR của đồng nghiệp có lỗi logic nghiêm trọng nhưng họ đang cần merge gấp để kịp deadline. Bạn xử lý thế nào?
Trả lời: Tôi sẽ liên hệ trực tiếp (chat/gọi điện) để giải thích lỗi và hỗ trợ họ fix nhanh nhất có thể. Tuyệt đối không Approve cho một PR có lỗi nghiêm trọng vì nó sẽ gây ra hậu quả lớn hơn cho cả team sau này.

## 14. References
- GitHub Flow: https://docs.github.com/en/get-started/using-github/github-flow
- Atlassian Git Workflows: https://www.atlassian.com/git/tutorials/comparing-workflows

## 15. Real-world Code
Tham khảo file `CONTRIBUTING.md` trong các repo lớn để thấy quy trình phối hợp của họ.

## 16. Community
- Discussion on "Good Code Review Practices" trên Medium/Dev.to.
- Reddit: r/cscareerquestions (thảo luận về văn hóa làm việc nhóm).
