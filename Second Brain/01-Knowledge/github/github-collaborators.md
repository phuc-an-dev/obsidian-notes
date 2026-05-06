---
created: 2026-05-05
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/github"
related:
  - "[[github-branch-protection]]"
  - "[[github-teams]]"
---

## 1. What
GitHub Collaborators (hay Outside Collaborators) là những người dùng được cấp quyền truy cập vào một repository cụ thể mà không cần phải là thành viên chính thức của Organization sở hữu repository đó. Đây là cách quản lý quyền truy cập ở cấp độ repository thay vì cấp độ organization.

## 2. Why
Trước khi có tính năng này, để một người ngoài (ví dụ freelancer hoặc đối tác) có thể đóng góp code, quản trị viên thường phải thêm họ vào Organization, điều này vô tình cấp cho họ quyền xem các thông tin nội bộ không cần thiết. GitHub Collaborators ra đời để giải quyết bài toán cấp quyền tối thiểu (Least Privilege) và quản lý chi phí (billing) linh hoạt hơn cho các dự án ngắn hạn.

## 3. Mental Model
Hãy tưởng tượng Organization của bạn là một tòa nhà văn phòng hiện đại. Thành viên trong Organization là nhân viên có thẻ từ để đi vào mọi phòng ban thông thường. GitHub Collaborator giống như một vị khách được cấp "Guest Pass" chỉ có giá trị để vào duy nhất một phòng họp cụ thể (Repository) để làm việc, và họ không thể đi lang thang sang các phòng khác trong tòa nhà.

## 4. Where it fits
Nằm trong phần cài đặt bảo mật và quyền truy cập của từng repository: `Repository Settings -> Access -> Collaborators`. Nó hoạt động bên dưới lớp bảo mật của Organization nhưng độc lập với cấu hình GitHub Teams.

## 5. When to use
- Khi thuê Freelancer thực hiện một task cụ thể trong một repo nhất định.
- Khi làm việc với đối tác bên thứ ba trong các dự án Outsourcing.
- Khi cần một chuyên gia bảo mật vào audit code của một ứng dụng duy nhất.
- Khi cộng tác trong các dự án Open Source cần quyền Write cho một vài cá nhân tin cậy.

## 6. When NOT to use
- Khi người đó là nhân viên chính thức và cần truy cập vào nhiều dự án khác nhau (nên dùng GitHub Teams).
- Khi bạn cần quản lý quyền tập trung và đồng bộ qua hệ thống LDAP hoặc SSO của doanh nghiệp.
- Khi dự án yêu cầu sự kiểm soát nghiêm ngặt về Intellectual Property (IP) mà chỉ thành viên Organization mới được phép chạm vào.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Dễ dàng thiết lập nhanh chóng cho từng repo. | Khó quản lý tập trung nếu số lượng collaborator lớn. |
| Đảm bảo nguyên tắc Least Privilege (quyền tối thiểu). | Có thể phát sinh chi phí billing riêng cho từng người (với private repo). |
| Không làm loãng danh sách thành viên trong Organization. | Khó theo dõi tổng thể ai đang truy cập vào những gì trong toàn bộ Org. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| GitHub Teams | Quản lý theo nhóm, phù hợp cho nhân viên nội bộ cần truy cập nhiều repo. |
| Enterprise Managed Users | Quản lý tập trung từ hệ thống danh tính bên ngoài (Azure AD, Okta). |
| Repository Rulesets | Cấu hình quy tắc bảo mật chung thay vì quản lý theo cá nhân. |

## 9. How
Để thêm một collaborator bằng GitHub CLI:
```bash
# Thêm người dùng với quyền 'push' (write) vào repository hiện tại
gh repo invite add <username> --permission write

# Liệt kê tất cả collaborators của một repo
gh api repos/:owner/:repo/collaborators --jq '.[].login'
```

## 10. Production concerns
### Scaling
Khi số lượng Collaborators lên đến hàng trăm, việc quản lý thủ công sẽ trở nên bất khả thi. Cần chuyển sang mô hình "Infrastructure as Code" (Terraform) để quản lý file cấu hình thay vì click UI.

### Failure
Nếu một Collaborator bị hack tài khoản cá nhân, họ có thể trở thành vector tấn công vào repo. Luôn yêu cầu 2FA (Two-Factor Authentication) cho tất cả mọi người có quyền Write.

### Monitoring
Sử dụng GitHub Audit Logs để theo dõi hành vi của các Collaborators. Thường xuyên chạy script để tìm kiếm và thu hồi quyền của những người đã lâu không hoạt động (Stale access).

## 11. Common mistakes
- Mistake: Cấp quyền Admin cho Outside Collaborator chỉ để họ merge Pull Request.
  Fix: Chỉ cấp quyền Write hoặc Maintain, sử dụng Branch Protection để quản lý việc merge.

- Mistake: Quên thu hồi quyền sau khi dự án kết thúc.
  Fix: Thiết lập lịch review quyền truy cập hàng tháng hoặc dùng GitHub API để tự động hóa việc expire access.

## 12. Sample project
Tạo một kịch bản: Bạn có một repository chứa landing page bằng React. Bạn thuê một freelancer fix bug CSS. Hãy thiết lập Collaborator cho họ với quyền Write, nhưng đồng thời setup Branch Protection để họ không được push trực tiếp vào main mà phải qua Pull Request và được bạn duyệt.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt lớn nhất giữa một Member và một Outside Collaborator là gì?
   A: Member thuộc về Organization, có thể được quản lý qua Teams và mặc định có thể thấy danh sách các thành viên khác hoặc repo công khai của Org. Outside Collaborator chỉ thấy duy nhất repo mà họ được mời và không có quyền hạn nào khác trong Organization.

2. Q: Làm thế nào để đảm bảo an toàn khi làm việc với Outside Collaborators?
   A: Cần thực hiện 3 bước: 1. Chỉ cấp quyền tối thiểu (thường là Write). 2. Bắt buộc 2FA cấp độ Organization (điều này sẽ yêu cầu collaborator cũng phải bật 2FA). 3. Sử dụng Branch Protection Rules để kiểm soát mọi thay đổi code của họ qua quy trình Review.

### Scenario
Tình huống: "Công ty bạn có 50 repositories và bạn cần thuê một đơn vị audit bảo mật vào kiểm tra 5 repos quan trọng nhất trong 2 tuần. Bạn sẽ quản lý quyền truy cập như thế nào?"
Giải pháp: Tôi sẽ tạo một file cấu hình Terraform định nghĩa 5 repos đó và mời các account của đơn vị audit dưới dạng Outside Collaborators với quyền Read-only. Sau 2 tuần, tôi chỉ cần xóa các dòng định nghĩa đó trong Terraform và apply lại để thu hồi toàn bộ quyền một cách đồng bộ và an toàn.

## 14. References
- Official Docs: https://docs.github.com/en/account-and-profile/setting-up-and-managing-your-personal-account-on-github/managing-access-to-your-personal-repositories/inviting-collaborators-to-a-personal-repository
- GitHub Repo: https://github.com/cli/cli
- Spec / RFC: GitHub Permissions Matrix

## 15. Real-world Code
Sử dụng Terraform để quản lý GitHub Collaborator:
```hcl
resource "github_repository_collaborator" "freelancer" {
  repository = "my-awesome-app"
  username   = "freelancer123"
  permission = "push"
}
```

## 16. Community
- Reddit: r/github
- Stack Overflow: github-api tags
- Blog: The GitHub Blog - Security entries
- Talk: GitHub Universe sessions on Security.
