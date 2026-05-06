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
  - "[[github-actions-workflow-protection]]"
---

## 1. What
GitHub Repository Roles là các cấp độ quyền hạn (Permission Levels) mà một cá nhân hoặc một Team được gán trên một repository cụ thể. Các role này định nghĩa những hành động nào họ có thể thực hiện (Read, Write, Admin, v.v.).

## 2. Why
Việc gán role đúng giúp:
- **Bảo mật mã nguồn**: Ngăn chặn người không có thẩm quyền xóa code hoặc thay đổi cấu hình repo.
- **Luồng làm việc (Workflow)**: Phân chia rõ ràng ai là người code (Writer), ai là người review/quản lý (Maintainer), ai là người chỉ đọc tài liệu (Reader).
- **Tránh sai sót**: Hạn chế số lượng người có quyền `Admin` để tránh việc vô tình xóa repository.

## 3. Mental Model
Hãy coi Repository như một **"Dự án xây dựng"**:
- **Read**: Người tham quan (chỉ được xem bản vẽ).
- **Triage**: Người giám sát sơ bộ (được dọn dẹp, phân loại gạch đá/issue nhưng không được xây).
- **Write**: Công nhân (được xây tường, đổ bê tông/push code).
- **Maintain**: Quản đốc (được chỉnh sửa bản vẽ, duyệt công việc của công nhân).
- **Admin**: Chủ thầu (có quyền đập đi xây lại hoặc bán dự án).

## 4. Where it fits
Role được gán tại:
`Repository Settings -> Collaborators and teams -> Add people/Add teams`.

## 5. When to use
Gán role cho Team là cách quản lý tốt nhất trong Organization.
- **Read**: Dành cho các bộ phận hỗ trợ (Sales, Support) cần xem tài liệu hoặc code để hiểu sản phẩm.
- **Triage**: Dành cho các contributor bên ngoài hoặc QA để quản lý Issues/PRs mà không cần quyền sửa code.
- **Write**: Dành cho các lập trình viên thực hiện công việc hàng ngày.
- **Maintain**: Dành cho Tech Lead hoặc Project Manager để quản lý release, tag và setting repo.
- **Admin**: Dành cho DevOps hoặc IT Admin để quản lý quyền truy cập và webhooks.

## 6. When NOT to use
- Không gán role `Admin` cho tất cả mọi người trong Team.
- Không gán role trực tiếp cho từng cá nhân (Individual) nếu họ đã thuộc về một Team trong Organization (nên dùng Team role).

## 7. Trade-offs
| Role | Quyền chính | Đánh đổi/Rủi ro |
|------|-------------|-----------------|
| Read | View, Clone, Fork. | Không thể đóng góp trực tiếp. |
| Write | Push code, PRs. | Có thể làm hỏng code nếu không có Branch Protection. |
| Maintain | Manage Tags, Settings. | Có thể thay đổi cấu hình repo mà không cần hỏi ý kiến. |
| Admin | Toàn quyền, kể cả xóa repo. | Rủi ro bảo mật cao nhất. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Custom Roles (Enterprise) | Cho phép chọn chính xác từng permission nhỏ. Rất linh hoạt nhưng phức tạp. |
| Organization Roles | Owner/Member (áp dụng cho toàn Org, không phải từng repo). |

## 9. How
### Gán Role cho Team (UI)
1. Vào Repository -> `Settings`.
2. Chọn `Collaborators and teams` ở menu bên trái.
3. Click `Add teams`.
4. Tìm tên Team và chọn Role từ dropdown (Read, Triage, Write, Maintain, Admin).

### Thay đổi Role của Team
1. Tại trang `Collaborators and teams`, tìm Team đã add.
2. Click vào dropdown role hiện tại và chọn role mới.

## 10. Production concerns
### Scaling
- Trong Enterprise, sử dụng **Custom Repository Roles** để tạo ra các role trung gian (ví dụ: `Security Auditor` chỉ có quyền xem security alerts).

### Failure
- Nếu một người thuộc nhiều Team có role khác nhau trên cùng một repo, họ sẽ nhận được **quyền cao nhất (highest permission)**.

### Monitoring
- Audit Log ghi lại mọi thay đổi về việc gán role. Cần review định kỳ (Quarterly Access Review).

## 11. Common mistakes
- **Mistake**: Gán quyền `Write` cho Team nhưng quên thiết lập **Branch Protection**.
  **Fix**: Luôn kết hợp `Write` role với Branch Protection để yêu cầu review trước khi merge.

- **Mistake**: Một thành viên cần thêm quyền nhưng lại đi gán role Admin cho cả Team của họ.
  **Fix**: Tạo một Team mới hoặc cân nhắc gán Individual role tạm thời (dù không khuyến khích).

## 12. Sample project
Thiết lập quyền cho Repo `Core-API`:
- Team `Backend-Developers`: `Write`.
- Team `QA-Engineers`: `Triage`.
- Team `Security-Ops`: `Read` + Security Alerts access.
- Team `DevOps-Lead`: `Admin`.

## 13. Interview
### Core Q&A
1. Q: Nếu User A thuộc Team X (quyền Read) và Team Y (quyền Write) trên cùng một repo, User A có quyền gì?
   A: User A sẽ có quyền **Write** vì GitHub luôn ưu tiên quyền cao nhất được gán.

2. Q: Role `Maintain` khác gì với `Write`?
   A: `Maintain` có thể quản lý repository settings, manage labels, milestones, và bypass branch protection (tùy config), trong khi `Write` chủ yếu tập trung vào việc tương tác với code và PRs.

### Scenario
**Tình huống**: Bạn muốn cho phép một nhóm Outsourced Devs vào fix bug nhưng không muốn họ merge trực tiếp vào `main`.
**Giải quyết**: 1. Tạo một Team cho Outsourced Devs. 2. Gán quyền `Write` cho Team này trên repo. 3. Thiết lập Branch Protection trên nhánh `main` yêu cầu ít nhất 1 review từ Team nội bộ.

## 14. References
- Official Docs: [Repository roles for an organization](https://docs.github.com/en/organizations/managing-access-to-your-organizations-repositories/repository-roles-for-an-organization)
- GitHub Repo: N/A

## 15. Real-world Code
N/A

## 16. Community
- Stack Overflow: Tag `github-permissions`
- Blog: `GitHub Security Blog`
