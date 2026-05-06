---
created: 2026-05-05
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/others"
  - "#topic/github"
related:
  - "[[github-teams]]"
  - "[[github-collaborators]]"
---

## 1. What
GitHub Organization là một tài khoản cấp doanh nghiệp hoặc nhóm cho phép nhiều người dùng cộng tác trên nhiều dự án cùng một lúc. Nó cung cấp các tính năng quản lý tập trung cho thành viên, repository, thanh toán và bảo mật mà tài khoản cá nhân (Personal Account) không có.

## 2. Why
Khi làm việc theo nhóm hoặc công ty:
- **Sở hữu tập trung**: Repository thuộc về Organization, không thuộc về cá nhân. Nếu một lập trình viên rời đi, mã nguồn vẫn nằm trong tầm kiểm soát của tổ chức.
- **Quản lý quyền quy mô lớn**: Hỗ trợ Teams để phân quyền cho hàng trăm người thay vì add từng người vào từng repo.
- **Bảo mật**: Yêu cầu bắt buộc 2FA cho toàn bộ thành viên, quản lý SSH keys/Personal Access Tokens tập trung.
- **Branding**: Cho phép tạo profile chuyên nghiệp cho công ty.

## 3. Mental Model
Hãy tưởng tượng Organization như một **"Tòa nhà văn phòng"**.
- Tòa nhà có chủ sở hữu (Owners).
- Trong tòa nhà có nhiều phòng (Repositories).
- Nhân viên (Members) được cấp thẻ ra vào để vào các phòng nhất định.
- Các nhân viên được chia vào các tổ/đội (Teams) để dễ quản lý.

## 4. Where it fits
Cấu trúc quản lý của GitHub:
`GitHub Enterprise (Optional) -> Organization -> Teams & Repositories -> Members`

## 5. When to use
- Công ty, startup cần quản lý mã nguồn tập trung.
- Nhóm dự án mã nguồn mở (Open Source) có nhiều người đóng góp.
- Khi cần các tính năng Enterprise như SAML SSO hoặc GitHub Actions self-hosted runners cấp độ tổ chức.

## 6. When NOT to use
- Dự án cá nhân, portfolio hoặc bài tập về nhà.
- Nhóm bạn học làm 1-2 repo nhỏ (Personal Account với Collaborators là đủ).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Quản lý tập trung, an toàn dữ liệu. | Cần trả phí nếu muốn dùng tính năng nâng cao (Team/Enterprise plan). |
| Khả năng mở rộng (Scale) tốt. | Phức tạp hơn trong việc thiết lập cấu trúc Teams ban đầu. |
| Tích hợp tốt với các công cụ CI/CD/IdP. | Một số tính năng giới hạn cho Public repo nếu dùng gói Free. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Personal Account | Miễn phí, đơn giản nhưng khó quản lý khi số lượng repo và người tăng lên. |
| GitLab Groups | Tương đương, nhưng nằm trong hệ sinh thái GitLab. |
| Bitbucket Workspaces | Tương đương, tích hợp sâu với Jira/Confluence. |

## 9. How
### Tạo Organization
1. Click dấu `+` ở góc trên bên phải -> `New organization`.
2. Chọn plan (Free, Team, Enterprise).
3. Đặt tên và email liên lạc.

### Mời thành viên
1. Vào Organization -> `People` -> `Invite member`.
2. Nhập username hoặc email của họ.
3. Chọn role trong Organization (Owner hoặc Member).

## 10. Production concerns
### Scaling
- Sử dụng **Identity Provider (IdP)** như Okta hoặc Azure AD để tự động sync thành viên (SCIM).
- Chia nhỏ Organization nếu công ty quá lớn (ví dụ: một Org cho Engineering, một Org cho Data Science).

### Failure
- Mất quyền truy cập của Owner duy nhất: Luôn phải có ít nhất 2 Owners cho một Organization để tránh bị lock-out.

### Monitoring
- Kiểm tra **Audit Log** thường xuyên để phát hiện các hành vi bất thường (thêm member lạ, đổi quyền repo nhạy cảm).

## 11. Common mistakes
- **Mistake**: Chỉ có 1 Owner duy nhất. Nếu người này nghỉ việc hoặc mất account, Org sẽ bị "mồ côi".
  **Fix**: Luôn duy trì tối thiểu 2-3 Owners tin cậy.

- **Mistake**: Cấp quyền Owner cho tất cả mọi người để "tiện làm việc".
  **Fix**: Owner có quyền xóa toàn bộ Org. Chỉ cấp cho người quản trị thực sự, những người khác chỉ nên là Member.

## 12. Sample project
Thiết lập Org cho dự án Startup:
- Plan: Free (cho phép private repo không giới hạn).
- 2 Owners (CTO và Lead Dev).
- Teams: `Backend`, `Frontend`, `DevOps`.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa Owner và Member trong một Organization?
   A: Owner có toàn quyền quản lý Org, bao gồm xóa Org, mời/xóa member, thay đổi billing. Member chỉ có quyền xem các member khác và tham gia vào các repo/team được cấp phép.

2. Q: Làm sao để bắt buộc mọi người trong Org dùng 2FA?
   A: Vào `Settings` -> `Authentication security` -> Check `Require two-factor authentication for everyone in your organization`.

### Scenario
**Tình huống**: Bạn làm việc tại một công ty mà các dự án đều nằm trong Account cá nhân của sếp. Sếp muốn bạn chuyển sang Organization. Bạn sẽ làm gì?
**Giải quyết**: 1. Tạo Organization mới. 2. Yêu cầu sếp chuyển quyền sở hữu (Transfer) các repository từ account cá nhân sang Org. 3. Tạo các Team và mời các dev khác vào Org.

## 14. References
- Official Docs: [Creating a new organization from scratch](https://docs.github.com/en/organizations/collaborating-with-groups-in-organizations/creating-a-new-organization-from-scratch)
- GitHub Repo: N/A
- Spec / RFC: N/A
- Changelog: [GitHub Roadmap](https://github.com/github/roadmap)

## 15. Real-world Code
N/A (Organization là thực thể quản lý qua UI/API).

## 16. Community
- Reddit: `r/github`
- Blog: `The GitHub Blog`
