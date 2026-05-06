---
created: 2026-05-05
tags:
  - "#type/pattern"
  - "#status/draft"
  - "#lang/others"
  - "#topic/devops"
related:
  - "[[github-actions-malicious-workflow]]"
  - "[[github-branch-protection]]"
  - "[[github-repository-roles]]"
---

## 1. What
Ràng buộc quyền thay đổi file CI/CD (GitHub Actions) là các biện pháp kỹ thuật nhằm kiểm soát ai có quyền chỉnh sửa các tệp YAML trong thư mục `.github/workflows/`. Mục tiêu là đảm bảo rằng các quy trình deploy quan trọng không bị thay đổi trái phép bởi những người không có chuyên môn hoặc kẻ tấn công.

## 2. Why
Nếu không có ràng buộc:
- **Rò rỉ Secrets**: Một dev có thể sửa workflow để in ra các `SECRETS` (AWS Key, Database Password) vào log.
- **Tấn công Supply Chain**: Kẻ xấu sửa file deploy để chèn mã độc vào file build trước khi đẩy lên server.
- **Phá vỡ quy trình**: Vô tình làm hỏng các bước kiểm tra (unit test, linting) dẫn đến code lỗi vẫn được deploy.
- **Chi phí**: Sửa đổi workflow để chạy các tác vụ tiêu tốn nhiều tài nguyên (như đào coin).

## 3. Mental Model
Hãy tưởng tượng tệp workflow là **"Bản thiết kế dây chuyền sản xuất tự động"**.
- Bất kỳ ai cũng có thể đề xuất sửa đổi (tạo PR).
- Nhưng chỉ **Kỹ sư trưởng (DevOps/SRE)** mới có quyền ký duyệt bản thiết kế đó trước khi nó được đưa vào vận hành thực tế.

## 4. Where it fits
Lớp bảo mật này nằm giữa **Source Control** và **Deployment Pipeline**.
`Developer -> Edit .github/workflows/*.yml -> PR -> CODEOWNERS Approval -> Branch Protection -> Merge -> Execution`

## 5. When to use
- Dự án có chứa thông tin nhạy cảm (Production Secrets).
- Team đông người, có nhiều trình độ khác nhau.
- Dự án mã nguồn mở nhận đóng góp từ cộng đồng.
- Các ngành yêu cầu tuân thủ bảo mật cao (Fintech, Healthcare).

## 6. When NOT to use
- Dự án cá nhân chỉ có một mình bạn làm chủ.
- Môi trường Sandbox/Research nơi cần sự linh hoạt tối đa để thử nghiệm CI/CD.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Đảm bảo tính toàn vẹn của quy trình Deploy. | Làm chậm tốc độ thay đổi CI/CD (cần chờ review). |
| Ngăn chặn các lỗi bảo mật nghiêm trọng. | Tạo ra nút thắt cổ chai (bottleneck) nếu team DevOps quá ít người. |
| Audit trail rõ ràng cho mọi thay đổi pipeline. | Đòi hỏi cấu hình phức tạp (CODEOWNERS, Rulesets). |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Reusable Workflows | Để logic CI/CD ở một repo riêng biệt và bảo vệ repo đó. Rất mạnh mẽ cho quy mô lớn. |
| GitHub Actions App | Dùng app bên thứ ba để quản lý và enforce policies. |
| Self-hosted Runners | Chặn thay đổi ở mức hạ tầng runner (nhưng file YAML vẫn có thể bị sửa). |

## 9. How
### Cách 1: Sử dụng `CODEOWNERS` (Hiệu quả nhất)
Tạo file `.github/CODEOWNERS` để chỉ định team DevOps phải review mọi thay đổi trong folder workflow.
```text
# Chỉ team devops mới có quyền duyệt thay đổi cho workflow
.github/workflows/ @my-org/devops-team
```
*Lưu ý: Phải bật "Require review from Code Owners" trong Branch Protection.*

### Cách 2: Branch Protection Rules
Thiết lập Ruleset hoặc Branch Protection cho các nhánh chính (main, production):
- **Restrict updates**: Chỉ cho phép một số người/team nhất định push code.
- **Require status checks**: Đảm bảo các check bảo mật phải pass trước khi merge.

### Cách 3: Giới hạn Token Permissions (Principle of Least Privilege)
Trong file YAML, hãy giới hạn quyền của `GITHUB_TOKEN`:
```yaml
permissions:
  contents: read    # Chỉ cho đọc code
  deployments: write # Chỉ cho phép tạo deployment
```

## 10. Production concerns
### Scaling
Với hàng trăm repo, hãy dùng **Organization-wide Repository Rulesets** để áp dụng chung một quy tắc bảo vệ file workflow cho toàn bộ công ty mà không cần sửa từng repo.

### Failure
Nếu toàn bộ team DevOps vắng mặt, việc sửa lỗi CI/CD khẩn cấp sẽ bị đình trệ. Cần có quy trình "Break-glass" (Owner của Org có thể bypass).

### Monitoring
Sử dụng **GitHub Advanced Security (GHAS)** để quét các tệp workflow tìm kiếm các cấu hình sai (misconfigurations) hoặc lộ secret.

## 11. Common mistakes
- **Mistake**: Quên bật "Require review from Code Owners" trong setting nhánh, dẫn đến file `CODEOWNERS` chỉ có tác dụng gợi ý thay vì bắt buộc.
  **Fix**: Vào Settings -> Branches -> Edit rule -> Tích chọn `Require review from Code Owners`.

- **Mistake**: Cho phép bypass branch protection cho Admin nhưng chính Admin đó lại bị hack account.
  **Fix**: Tuyệt đối không để Admin bypass các quy tắc bảo mật quan trọng trên nhánh Production.

## 12. Sample project
Tạo một repo với cấu trúc:
- `.github/workflows/deploy.yml`: Chứa logic deploy lên Production.
- `.github/CODEOWNERS`: Gán quyền quản lý cho `@org/security-admins`.
- Cấu hình Branch Protection yêu cầu 2 reviewers và phải có 1 từ Code Owners.

## 13. Interview
### Core Q&A
1. Q: Làm thế nào để ngăn chặn một dev sửa file workflow để "leak" secrets ra console log?
   A: Sử dụng `CODEOWNERS` để yêu cầu team bảo mật review mọi thay đổi file workflow, kết hợp với tính năng "Secret scanning" và không bao giờ cho phép in biến môi trường ra log.

2. Q: "Reusable Workflows" giúp gì trong việc bảo mật quyền thay đổi?
   A: Nó giúp tập trung logic deploy vào một repo duy nhất. Các dự án con chỉ "gọi" workflow này. Ta chỉ cần bảo vệ repo chứa workflow gốc là xong.

### Scenario
**Tình huống**: Bạn phát hiện một dev vừa merge một PR có thay đổi file `.github/workflows/ci.yml` mà không qua review của team DevOps. Bạn xử lý thế nào?
**Giải quyết**: 1. Revert ngay commit đó. 2. Kiểm tra xem tại sao hệ thống lại cho phép merge (do thiếu Branch Protection hay do người đó có quyền bypass). 3. Cập nhật lại `CODEOWNERS` và khóa chặt Branch Protection.

## 14. References
- Official Docs: [About code owners](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners)
- Security Guide: [Security hardening for GitHub Actions](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)

## 15. Real-world Code
N/A

## 16. Community
- Reddit: `r/DevOps`
- Talk: "Hardening your GitHub Actions Workflows" by Octocat at Universe.
