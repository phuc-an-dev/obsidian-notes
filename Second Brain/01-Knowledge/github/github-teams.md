---
created: 2026-05-05
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/others"
  - "#topic/devops"
related:
  - "[[github-organization]]"
  - "[[github-repository-roles]]"
  - "[[github-collaborators]]"
  - "[[github-branch-protection]]"
---

## 1. What
GitHub Teams là một tính năng trong GitHub Organization cho phép nhóm các thành viên lại với nhau để quản lý quyền truy cập repository và điều phối công việc tập trung. Thay vì gán quyền cho từng cá nhân (Individual), ta gán quyền cho một Team và mọi thành viên trong Team đó sẽ thừa hưởng quyền tương ứng.

## 2. Why
Khi một tổ chức (Organization) phát triển về số lượng nhân sự và số lượng repository:
- **Quản lý thủ công**: Việc cấp quyền Read/Write/Admin cho từng người vào từng repo trở nên cực kỳ tốn thời gian và dễ sai sót.
- **Onboarding/Offboarding**: Khi có nhân viên mới, thay vì add họ vào 20 repos, ta chỉ cần add họ vào 1 Team. Ngược lại khi họ nghỉ việc, chỉ cần remove khỏi Team.
- **Truyền thông**: Khó khăn trong việc tag (mention) đúng nhóm người cần review code hoặc thảo luận trong Issue.

## 3. Mental Model
Hãy tưởng tượng Team như một **"Danh sách gửi thư" (Mailing List)** kết hợp với **"Chùm chìa khóa"**. 
- Bạn gom một nhóm người vào danh sách.
- Bạn gắn chùm chìa khóa (quyền truy cập repo) vào danh sách đó. 
- Bất kỳ ai nằm trong danh sách đều có thể mở khóa các phòng (repo) mà danh sách đó được phép.

## 4. Where it fits
Cấu trúc phân cấp trong GitHub:
`Organization -> Teams (Parent/Child) -> Members`
`Organization -> Repositories <- Permissions granted to Teams`

Team cũng hỗ trợ cấu trúc phân cấp (Nested Teams):
`Engineering (Parent Team) -> Backend (Child Team), Frontend (Child Team)`

## 5. When to use
- Quản lý quyền cho các phòng ban kỹ thuật (Mobile, Devops, QA).
- Phân chia quyền theo dự án cụ thể.
- Cần tính năng Code Owners (tự động yêu cầu review từ một Team cụ thể).
- Cần đồng bộ hóa thành viên từ các hệ thống Identity Provider (IdP) như Okta, Azure AD thông qua SCIM.

## 6. When NOT to use
- Tài khoản cá nhân (Personal Account): Teams chỉ khả dụng cho Organizations.
- Nhóm làm việc quá nhỏ (dưới 3 người) và chỉ có 1-2 repo: Dùng Direct Collaborators sẽ nhanh hơn.
- Các cộng tác viên bên ngoài (Outside Collaborators) chỉ tham gia vào 1 repo duy nhất: Không nên đưa họ vào Team của tổ chức để đảm bảo bảo mật.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Quản lý quyền tập trung, giảm thiểu sai sót (Human Error). | Đòi hỏi tài khoản Organization (Free/Team/Enterprise). |
| Hỗ trợ Nested Teams giúp phản ánh đúng sơ đồ tổ chức. | Cấu trúc quá nhiều tầng (Deeply Nested) có thể gây khó hiểu về quyền thừa hưởng. |
| Mention nhanh qua `@org/team-name`. | Các tính năng nâng cao (như đồng bộ IdP) yêu cầu gói Enterprise. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Individual Collaborators | Phù hợp cho repo cá nhân hoặc quy mô cực nhỏ. Khó scale. |
| GitHub Apps | Có thể dùng API để tự động hóa việc cấp quyền, nhưng phức tạp để setup. |

## 9. How
### Tạo Team qua UI
1. Vào `Your Organization` -> `Teams` -> `New team`.
2. Đặt tên, chọn Parent team (nếu có) và Visibility (Visible/Secret).

### Cấp quyền cho Team vào Repo
1. Tại trang Team -> `Repositories` -> `Add repository`.
2. Chọn quyền: `Read`, `Triage`, `Write`, `Maintain`, hoặc `Admin`.

### Sử dụng trong Code Owners (`.github/CODEOWNERS`)
```text
# Tự động yêu cầu team-alpha review các file trong folder src/
src/ @my-org/team-alpha
```

## 10. Production concerns
### Scaling
- Sử dụng **Nested Teams** để phản ánh cấu trúc công ty. Ví dụ: Team `Engineering` có quyền `Read` toàn bộ repo, nhưng Team `DevOps` (con của Engineering) có quyền `Admin` vào các repo hạ tầng.

### Failure
- Nếu xóa nhầm một Team, toàn bộ quyền của các thành viên trong đó vào các repo liên kết sẽ bị mất ngay lập tức. Cần cẩn trọng khi thực hiện thao tác xóa.

### Monitoring
- Sử dụng **Audit Log** của Organization để theo dõi ai đã add/remove thành viên khỏi Team hoặc thay đổi quyền của Team trên Repository.

## 11. Common mistakes
- **Mistake**: Để Team ở chế độ `Secret` khiến các thành viên khác trong Organization không thể mention hoặc yêu cầu review.
  **Fix**: Dùng `Visible` cho các team nội bộ cần phối hợp, chỉ dùng `Secret` cho các nhóm nhạy cảm (ví dụ: Security Team, HR).

- **Mistake**: Cấp quyền `Admin` cho quá nhiều Team trên cùng một repository.
  **Fix**: Tuân thủ nguyên tắc "Least Privilege". Chỉ cấp `Write` hoặc `Maintain` cho các team phát triển, chỉ `Admin` cho các team quản lý repo chính.

## 12. Sample project
Thiết lập hệ thống phân quyền cho một công ty giả định:
- Team `All-Staff`: Quyền `Read` các repo documentation.
- Team `Backend-Engineers`: Quyền `Write` các repo microservices.
- Team `Platform-Admins`: Quyền `Admin` các repo Terraform/Infrastructure.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa `Maintain` và `Admin` permission của một Team là gì?
   A: `Admin` có toàn quyền bao gồm cả việc xóa repository và quản lý quyền truy cập. `Maintain` có quyền quản lý nội dung repo, quản lý Issues/PRs và một số setting nhưng không được xóa repo hay thay đổi quyền sở hữu nhạy cảm.

2. Q: Làm thế nào để tự động thêm thành viên vào một Team khi họ join Organization?
   A: GitHub không có tính năng tự động hoàn toàn cho mọi member, nhưng có thể dùng GitHub Actions hoặc đồng bộ qua IdP (Okta/Azure AD) với gói Enterprise.

### Scenario
**Tình huống**: Bạn có 50 lập trình viên và 100 repositories. Mỗi khi có dự án mới, bạn mất 1 tiếng để add từng người. Bạn sẽ giải quyết thế nào?
**Giải quyết**: Tạo các Functional Teams (ví dụ: `Backend-Team`, `Frontend-Team`). Cấp quyền cho các Team này vào các nhóm repo tương ứng. Khi có thành viên mới, chỉ cần add họ vào đúng Team một lần duy nhất.

## 14. References
- Official Docs: [About teams - GitHub Docs](https://docs.github.com/en/organizations/organizing-members-into-teams/about-teams)
- GitHub Repo: N/A
- Spec / RFC: N/A
- Changelog: [GitHub Blog - Teams category](https://github.blog/category/community/teams/)

## 15. Real-world Code
Hầu hết các dự án Open Source lớn trên GitHub đều sử dụng Teams để quản lý Maintainers (ví dụ: Facebook, Google, Microsoft).

## 16. Community
- Reddit: `r/github`
- Stack Overflow: Tag `github-teams`
- Blog: `The GitHub Blog`
