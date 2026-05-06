---
created: 2026-05-05
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/others"
  - "#topic/error-handling"
related:
  - "[[github-actions-malicious-workflow]]"
  - "[[github-secrets-and-variables]]"
  - "[[github-secret-injection]]"
---

## 1. What
GitHub Secrets là các biến môi trường được mã hóa (encrypted environment variables) dùng để lưu trữ các thông tin nhạy cảm như API keys, passwords, SSH keys. Chúng chỉ có thể được sử dụng trong các GitHub Actions workflows hoặc GitHub Apps.

## 2. Why
Nếu lưu trực tiếp secrets vào mã nguồn (Hardcoding), thông tin này sẽ lộ lọt trong lịch sử commit, dẫn đến nguy cơ bị hack tài khoản hoặc rò rỉ dữ liệu. GitHub Secrets ra đời để tách biệt thông tin cấu hình nhạy cảm khỏi code, đảm bảo chỉ các quy trình tự động được cấp quyền mới có thể truy cập.

## 3. Mental Model
Hãy coi GitHub Secrets giống như một chiếc két sắt an toàn đặt trong kho chứa đồ (Repository). Bạn có thể đưa đồ vật vào két (Add secret), nhưng sau khi đóng cửa két, bạn không bao giờ nhìn thấy đồ vật đó nữa (không thể xem lại giá trị). Chỉ những nhân viên đáng tin cậy (Workflow) có chìa khóa mới có thể lấy đồ ra sử dụng khi cần thiết.

## 4. Where it fits
User/Org Settings -> Security -> Secrets and variables -> Actions -> Workflows.
Dữ liệu được mã hóa bởi GitHub bằng Libsodium sealed box trước khi lưu trữ.

## 5. When to use
- Lưu trữ AWS Access Keys để deploy lên cloud.
- Lưu trữ Docker Hub credentials để push image.
- Lưu trữ Database Connection Strings cho quá trình chạy Integration Tests.
- Lưu trữ Token của các bên thứ ba (SendGrid, Stripe, Slack).

## 6. When NOT to use
- Không dùng để lưu trữ các cấu hình không nhạy cảm (dùng GitHub Variables thay thế).
- Không dùng cho dữ liệu cần thay đổi thường xuyên bởi người dùng bên ngoài hệ thống CI/CD.
- Không dùng làm database để lưu trữ dữ liệu ứng dụng.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Mã hóa mạnh mẽ, an toàn tuyệt đối | Không thể xem lại giá trị sau khi đã lưu (Write-only) |
| Tích hợp sẵn với hệ sinh thái GitHub | Giới hạn dung lượng (64 KB mỗi secret) |
| Hỗ trợ phân quyền theo Environment | Khó debug nếu giá trị bị sai (do bị che dấu trong logs) |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| HashiCorp Vault | Chuyên nghiệp hơn, quản lý tập trung nhiều hệ thống, có cơ chế rotate key tự động |
| AWS Secrets Manager | Tốt cho các ứng dụng chạy hoàn toàn trên AWS, hỗ trợ SDK mạnh mẽ |
| .env files (gitignored) | Chỉ dùng cho môi trường local, không an toàn cho CI/CD |

## 9. How
```yaml
# Ví dụ sử dụng secret trong GitHub Actions Workflow
name: CI/CD Pipeline
on: [push]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
```

## 10. Production concerns
### Scaling
Với các tổ chức lớn, nên sử dụng Organization Secrets để chia sẻ chung các keys (như Slack Webhook) cho nhiều repo thay vì cài đặt thủ công cho từng cái.

### Failure
Nếu một secret bị hết hạn (như API Token), workflow sẽ fail hàng loạt. Cần có cơ chế thông báo khi secrets sắp hết hạn hoặc sử dụng OIDC (OpenID Connect) để tránh dùng long-lived keys.

### Monitoring
GitHub cung cấp Audit Logs để theo dõi ai đã tạo, cập nhật hoặc xóa một secret (nhưng không theo dõi được lần đọc của workflow).

## 11. Common mistakes
- Mistake: In trực tiếp secret ra console để debug bằng lệnh `echo ${{ secrets.MY_KEY }}`.
  Fix: GitHub sẽ tự động che (mask) các secret trong log bằng dấu `***`, nhưng nên tránh in chúng ra hoàn toàn. Nếu cần debug, hãy kiểm tra độ dài hoặc checksum thay vì in giá trị.

- Mistake: Sử dụng cùng một secret cho cả môi trường Staging và Production.
  Fix: Sử dụng GitHub Environments để tách biệt secrets (Environment Secrets).

## 12. Sample project
Xây dựng một workflow tự động kiểm tra xem các secrets hiện tại có bị lộ trong code hay không bằng cách sử dụng công cụ `gitleaks` tích hợp vào GitHub Actions. Nếu phát hiện leak, workflow sẽ báo lỗi và ngăn chặn việc merge PR.

## 13. Interview
### Core Q&A
1. Q: Bạn có thể xem lại giá trị của một GitHub Secret sau khi đã lưu không?
   A: Không, GitHub Secrets là write-only. Bạn chỉ có thể cập nhật giá trị mới hoặc xóa nó. Điều này đảm bảo ngay cả người có quyền Admin repo cũng không thể lấy cắp key sau khi đã cấu hình.
2. Q: GitHub xử lý thế nào nếu bạn cố tình in một secret ra log?
   A: GitHub có bộ lọc tự động tìm kiếm các giá trị secret trong đầu ra của workflow và thay thế chúng bằng dấu `***`. Tuy nhiên, nếu bạn mã hóa secret (ví dụ Base64) trước khi in, bộ lọc này có thể bị vượt qua.

### Scenario
Một lập trình viên mới vô tình thêm một bước vào workflow để gửi toàn bộ biến môi trường (bao gồm cả secrets) tới một server lạ. Làm thế nào để ngăn chặn điều này?
Giải pháp: 1. Sử dụng tính năng "Require approval for all outside collaborators". 2. Giới hạn quyền của `GITHUB_TOKEN`. 3. Sử dụng "Environment protection rules" để yêu cầu reviewer phê duyệt trước khi job deploy (có truy cập secret) được chạy.

## 14. References
- Official Docs: https://docs.github.com/en/actions/security-guides/encrypted-secrets
- GitHub Repo: https://github.com/actions
- Spec / RFC: Libsodium documentation.
- Changelog: Introduction of Environment Secrets.

## 15. Real-world Code
https://github.com/github/advisory-database (Dữ liệu về các lỗ hổng bảo mật, bao gồm cả việc lộ secrets)

## 16. Community
- Reddit: r/GitHubActions
- Stack Overflow: Tag #github-actions #secrets
- Blog: GitHub Security Blog
- Talk: "Hardening your GitHub Actions" at GitHub Universe.
