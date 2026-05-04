---
created: 2026-04-30
tags:
  - "#type/pattern"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/version-control"
related:
  - "[[Common Git Commands]]"
  - "[[Merge vs Rebase in Git]]"
  - "[[Pull Request in Git]]"
---

## 1. What
Branching Strategy (Chiến lược phân nhánh) là một tập hợp các quy tắc mà các nhà phát triển tuân theo khi viết, gộp và triển khai code trong một hệ thống quản lý phiên bản như Git. Nó xác định cách các nhánh (branches) được tạo ra, đặt tên và tích hợp lại với nhau để đảm bảo luồng công việc mượt mà và code ổn định.

## 2. Why
Nếu không có một chiến lược rõ ràng, dự án dễ rơi vào tình trạng "Merge Hell" (Xung đột gộp mã nghiêm trọng), code lỗi bị đẩy lên môi trường sản xuất (production), hoặc các tính năng đang phát triển bị lẫn lộn với nhau. Một quy trình chuẩn giúp tách biệt các môi trường (development, staging, production) và cho phép nhiều người cùng phát triển các tính năng khác nhau một cách độc lập.

## 3. Mental Model
Hãy tưởng tượng dự án của bạn là một con đường cao tốc chính (Main branch). 
- Các tính năng mới giống như những con đường nhánh (Feature branches) rẽ ra để thi công. 
- Sau khi hoàn thành và kiểm tra độ an toàn (Pull Request/Code Review), các con đường nhánh này sẽ được nhập làn trở lại cao tốc chính. 
- Các trạm thu phí và kiểm soát chính là các môi trường Test/Staging trước khi xe tiến vào trung tâm thành phố (Production).

## 4. Where it fits
Branching Strategy nằm giữa quy trình lập trình (Coding) và quy trình triển khai (Deployment/CI-CD).
Local Dev -> Feature Branch -> Pull Request -> Code Review -> Develop Branch -> Staging -> Main/Master Branch -> Production.

## 5. When to use
- Khi làm việc nhóm từ 2 người trở lên.
- Khi dự án cần duy trì nhiều phiên bản cùng lúc (ví dụ: đang fix lỗi bản cũ trong khi phát triển bản mới).
- Khi muốn áp dụng quy trình Code Review để nâng cao chất lượng code.

## 6. When NOT to use
- Các dự án cá nhân cực nhỏ, chỉ có một mình làm và không cần triển khai tự động (có thể commit trực tiếp vào main, dù không khuyến khích).
- Khi thực hiện các thay đổi cực nhỏ về tài liệu (Documentation) mà không ảnh hưởng đến logic code.

## 7. Trade-offs
| Chiến lược | Pros | Cons |
|------------|------|------|
| **Gitflow** | Rất chặt chẽ, phù hợp cho sản phẩm có chu kỳ phát hành (release cycle) rõ ràng. | Khá phức tạp, nhiều loại nhánh, không phù hợp cho CI/CD tốc độ cao. |
| **Trunk-based** | Tốc độ cực nhanh, hỗ trợ Continuous Deployment tốt, giảm thiểu xung đột lớn. | Đòi hỏi trình độ team cao, hệ thống Automated Test cực tốt và Feature Flags. |
| **GitHub Flow** | Đơn giản, dễ hiểu, phù hợp cho web apps triển khai liên tục. | Thiếu sự phân cấp rõ ràng cho các môi trường phức tạp. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Feature Flags | Thay vì tách nhánh lâu dài, code được đẩy vào main nhưng được "ẩn" đi bằng biến cấu hình cho đến khi sẵn sàng. |
| GitLab Flow | Kết hợp giữa GitHub Flow và các môi trường (production branches) hoặc phiên bản (release branches). |

## 9. How
### Quy trình Gitflow chuẩn thực tế:
1. **Main**: Nhánh chứa code chính thức đang chạy trên Production.
2. **Develop**: Nhánh tích hợp các tính năng đã hoàn thành, dùng để test nội bộ.
3. **Feature/**: Nhánh con từ Develop để phát triển tính năng mới (Ví dụ: `feature/login-page`).
4. **Release/**: Nhánh chuẩn bị cho việc phát hành, dùng để bug fix cuối cùng trước khi merge vào Main.
5. **Hotfix/**: Nhánh con từ Main để sửa lỗi khẩn cấp trên Production.

```bash
# Bắt đầu tính năng mới
git checkout develop
git checkout -b feature/user-profile

# Sau khi xong, đẩy lên và tạo Pull Request vào develop
git push origin feature/user-profile

# Khi chuẩn bị release
git checkout develop
git checkout -b release/v1.1.0

# Sau khi test xong, merge release vào main và develop
git checkout main
git merge release/v1.1.0
git tag -a v1.1.0 -m "Release version 1.1.0"
```

## 10. Production concerns
### Scaling
Với team lớn (hàng trăm người), Trunk-based Development thường được ưu tiên để tránh việc các nhánh feature sống quá lâu dẫn đến xung đột không thể cứu vãn.

### Failure
Nếu một nhánh Release bị lỗi nặng, toàn bộ quy trình merge phải dừng lại để fix ngay trên nhánh Release đó trước khi đưa vào Main.

### Monitoring
Sử dụng các công cụ như SonarQube hoặc GitHub Actions để tự động kiểm tra code quality ngay khi tạo Pull Request từ các nhánh feature.

## 11. Common mistakes
- Mistake: Giữ nhánh feature quá lâu (vài tuần/tháng) mà không cập nhật code mới từ Develop.
  Fix: Luôn `git pull origin develop` và merge vào nhánh feature hàng ngày.

- Mistake: Commit code lỗi hoặc code chưa chạy được lên nhánh Develop/Main.
  Fix: Chỉ merge vào các nhánh chung sau khi đã qua Code Review và vượt qua tất cả Unit Tests.

## 12. Sample project
Thiết lập một dự án với quy tắc: Nhánh `main` và `develop` bị khóa (protected), mọi thay đổi phải thông qua Pull Request từ nhánh `feature/*` và cần ít nhất 1 người duyệt (approve).

## 13. Interview
### Core Q&A
1. Q: Tại sao chúng ta cần nhánh Hotfix thay vì sửa trực tiếp trên Develop?
   A: Vì nhánh Develop có thể đang chứa các tính năng mới đang dang dở, chưa sẵn sàng để lên Production. Nhánh Hotfix tách từ Main giúp ta sửa đúng lỗi đó và đẩy lên ngay mà không kéo theo các code chưa hoàn thiện khác.

2. Q: Giải thích sự khác biệt chính giữa Gitflow và Trunk-based Development?
   A: Gitflow tập trung vào việc quản lý các phiên bản phát hành với nhiều nhánh dài hạn. Trunk-based tập trung vào việc gộp code vào nhánh chính nhanh nhất có thể (thường trong ngày) để tối ưu hóa CI/CD.

### Scenario
Tình huống: Bạn đang phát triển tính năng trên nhánh `feature/A` thì được yêu cầu hỗ trợ đồng nghiệp fix bug trên nhánh `feature/B`. Bạn sẽ xử lý thế nào?
Trả lời: Sử dụng `git stash` để lưu code `feature/A`, chuyển sang `feature/B` hỗ trợ. Sau khi xong, quay lại `feature/A` và `git stash pop` để tiếp tục.

## 14. References
- Original Gitflow Post: https://nvie.com/posts/a-successful-git-branching-model/
- Trunk Based Development: https://trunkbaseddevelopment.com/
- GitHub Flow Guide: https://docs.github.com/en/get-started/using-github/github-flow

## 15. Real-world Code
Tham khảo quy trình của các công ty lớn:
- Google/Facebook: Thường dùng Trunk-based Development.
- Các dự án Enterprise truyền thống: Thường dùng Gitflow.

## 16. Community
- Reddit: r/devops (thảo luận về workflow)
- Stack Overflow: Tag [git-flow], [branching-strategy]
- Blog: Atlassian Git Tutorial (rất chi tiết về các workflow)
