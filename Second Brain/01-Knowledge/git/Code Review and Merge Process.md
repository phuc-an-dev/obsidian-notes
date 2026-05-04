---
created: 2026-04-30
tags:
  - "#type/pattern"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/version-control"
related:
  - "[[Pull Request in Git]]"
  - "[[Team Collaboration Workflow in Git]]"
  - "[[Squash Commit]]"
---

## 1. What
Code Review và Merge Process là quy trình kiểm duyệt mã nguồn và tích hợp thay đổi vào nhánh chính của dự án. Đây là bước kiểm soát chất lượng cuối cùng đảm bảo code không chỉ chạy đúng mà còn dễ bảo trì, tuân thủ đúng coding standards và kiến trúc chung của hệ thống.

## 2. Why
Lập trình viên dù giỏi đến đâu cũng có thể mắc sai lầm hoặc thiếu sót một góc nhìn nào đó. Quy trình review giúp phát hiện lỗi logic, lỗ hổng bảo mật hoặc code rườm rà. Merge process đảm bảo việc tích hợp diễn ra an toàn, không làm hỏng code của người khác và giữ lịch sử dự án ổn định.

## 3. Mental Model
Hãy tưởng tượng dự án như một "Dây chuyền sản xuất ô tô":
- **Code Review**: Là trạm kiểm tra chất lượng (QC). Mỗi linh kiện (code) phải được kỹ sư kiểm tra kỹ lưỡng trước khi lắp ráp.
- **Merge**: Là hành động lắp linh kiện đó vào khung xe chính thức.
Nếu QC làm việc hời hợt, chiếc xe ra lò sẽ không an toàn (bugs trên production).

## 4. Where it fits
Nằm ở giai đoạn cuối của vòng đời phát triển tính năng:
Development -> Unit Test -> Pull Request -> **Code Review** -> **Approval** -> **Merge** -> CI/CD.

## 5. When to use
- Áp dụng cho mọi thay đổi mã nguồn trong các dự án chuyên nghiệp.
- Khi cần hướng dẫn và đào tạo các thành viên mới thông qua việc góp ý code.
- Khi muốn đồng bộ hóa phong cách viết code (coding style) trong toàn team.

## 6. When NOT to use
- Dự án cá nhân (mình làm mình biết).
- Các dự án prototype/hackathon đòi hỏi tốc độ tối đa và không đặt nặng tính bảo trì dài hạn.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Giảm thiểu bug và nợ kỹ thuật (Technical debt) | Tốn tài nguyên nhân sự (người review) |
| Chia sẻ kiến thức và trách nhiệm chung | Có thể gây chậm trễ tiến độ nếu quy trình quá cứng nhắc |
| Nâng cao kỹ năng của cả người viết và người review | Dễ dẫn đến tranh cãi cá nhân nếu không có văn hóa tốt |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Automated Code Analysis | Dùng tool quét tự động (SonarQube). Nhanh nhưng không hiểu được logic nghiệp vụ phức tạp. |
| Over-the-shoulder Review | Ngồi cạnh nhau cùng xem code. Nhanh nhưng không lưu lại lịch sử thảo luận. |

## 9. How
### Quy trình 5 bước chuẩn chuyên nghiệp:
1. **Self-Review**: Tác giả tự đọc lại code, chạy test và đảm bảo code sạch sẽ trước khi mời người khác.
2. **Reviewing**: Reviewer kiểm tra:
   - Logic nghiệp vụ (có chạy đúng yêu cầu không?).
   - Kiến trúc và thiết kế (có đúng pattern không?).
   - Tính dễ đọc và bảo trì (tên biến, hàm, độ phức tạp).
   - Bảo mật và hiệu năng.
3. **Commenting**: Đưa ra nhận xét mang tính xây dựng. Không dùng câu lệnh, hãy dùng câu hỏi hoặc gợi ý (VD: "Bạn nghĩ sao nếu dùng `map` ở đây thay vì `for`?").
4. **Addressing**: Tác giả phản hồi các comment, sửa code và cập nhật PR.
5. **Merging**: Sau khi đạt đủ số lượng Approval (thường là 1-2), tác giả hoặc Lead sẽ tiến hành Merge (thường chọn Squash Merge để sạch lịch sử).

## 10. Production concerns
### Scaling
Sử dụng `CODEOWNERS` trong GitHub để tự động gắn thẻ người review tương ứng cho từng module cụ thể (ví dụ: Team Backend review folder `/api`, Team Frontend review folder `/src`).

### Failure
Nếu sau khi merge mà phát hiện lỗi, phải có quy trình Rollback hoặc Hotfix ngay lập tức. Tuyệt đối không sửa trực tiếp trên nhánh chính mà không qua review lại.

### Monitoring
Theo dõi các chỉ số như "Cycle Time" (thời gian từ lúc tạo PR đến khi merge) để tối ưu quy trình làm việc.

## 11. Common mistakes
- Mistake: Reviewer chỉ soi lỗi cú pháp (typos) mà bỏ qua logic nghiệp vụ.
  Fix: Tập trung vào "Big Picture" trước, sau đó mới đến các chi tiết nhỏ.

- Mistake: Comment mang tính chỉ trích cá nhân thay vì tập trung vào code.
  Fix: Luôn giữ thái độ chuyên nghiệp, tôn trọng đồng nghiệp.

## 12. Sample project
Thiết lập quy tắc: Một PR chỉ được merge khi có ít nhất 1 "Approved" và 0 "Changes Requested", đồng thời tất cả các test tự động phải "Pass".

## 13. Interview
### Core Q&A
1. Q: Bạn tìm kiếm điều gì khi review code của đồng nghiệp?
   A: Tôi ưu tiên theo thứ tự: 1. Đúng logic nghiệp vụ, 2. Tính bảo mật, 3. Hiệu năng, 4. Khả năng bảo trì (clean code) và 5. Coding standards của dự án.

2. Q: Làm thế nào để giải quyết xung đột ý kiến trong Code Review?
   A: Trước tiên tôi sẽ dựa vào coding standards của dự án làm trọng tài. Nếu vẫn không xong, tôi sẽ đề xuất một buổi thảo luận nhanh hoặc nhờ sự can thiệp của Technical Lead để đưa ra quyết định cuối cùng dựa trên lợi ích của dự án.

### Scenario
Tình huống: Đồng nghiệp gửi một PR rất tốt nhưng lại thiếu Unit Test. Bạn xử lý thế nào?
Trả lời: Tôi sẽ khen ngợi những điểm tốt trong code của họ, sau đó nhẹ nhàng nhắc nhở về việc bổ sung Unit Test để đảm bảo tính ổn định lâu dài. Tôi sẽ không Approved cho đến khi code có độ bao phủ test (coverage) tối thiểu theo quy định.

## 14. References
- Google's Code Review Guide: https://google.github.io/eng-practices/review/
- Palantir's Code Review Best Practices: https://github.com/palantir/gradle-pt-code-style/blob/develop/docs/CODE_REVIEWS.md

## 15. Real-world Code
Nghiên cứu các PR trên các repo nổi tiếng như React hoặc VS Code để học cách các chuyên gia hàng đầu review code cho nhau.

## 16. Community
- Discussion on "Human side of Code Review".
- Reddit: r/experienceddevs.
