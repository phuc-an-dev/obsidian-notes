---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/ci-cd"
related:
  - "[[github-actions-cd.md]]"
  - "[[github-secrets.md]]"
  - "[[github-actions-triggers.md]]"
  - "[[github-environment-secrets.md]]"
---

## 1. What
GitHub Environments là một tính năng trong GitHub Actions cho phép người dùng định nghĩa các đích đến của việc triển khai (như `production`, `staging`, `development`). Mỗi môi trường có thể có các quy tắc bảo vệ riêng (Environment protection rules) và các biến bí mật riêng (Environment secrets).

## 2. Why
Trong quy trình CD thực tế, việc triển khai lên Production không nên diễn ra một cách tự động hoàn toàn mà cần có sự kiểm soát chặt chẽ. GitHub Environments ra đời để giải quyết các vấn đề:
- **Kiểm soát quyền**: Chỉ cho phép triển khai khi có sự phê duyệt của người có thẩm quyền.
- **Quản lý biến môi trường**: Tách biệt bí mật (Secrets) giữa môi trường Test và Production.
- **Giám sát**: Theo dõi lịch sử triển khai cho từng môi trường cụ thể một cách trực quan trên giao diện GitHub.

## 3. Mental Model
Hãy tưởng tượng GitHub Environments giống như các **"Cánh cửa bảo mật"** dẫn vào các khu vực khác nhau trong một tòa nhà:
- Khu vực `Staging` là phòng thí nghiệm, cửa mở tự động khi bạn có thẻ nhân viên (CI pass).
- Khu vực `Production` là kho tiền, cửa này yêu cầu phải có hai người cùng quẹt thẻ (Required Reviewers) và phải đợi một khoảng thời gian xác minh (Wait timer) mới được vào.

## 4. Where it fits
Vị trí trong luồng CD:
`Code -> CI (Build/Test) -> Artifact -> Jobs (Environment: Staging) -> Deployment -> Jobs (Environment: Production) -> Approval -> Deployment`

## 5. When to use
- Khi cần thiết lập quy trình phê duyệt thủ công (Manual Approval) trước khi deploy lên Production.
- Khi muốn giới hạn chỉ những nhánh (branches) nhất định mới được deploy lên một môi trường cụ thể.
- Khi các môi trường khác nhau sử dụng các API Key hoặc Database Password khác nhau.

## 6. When NOT to use
- Đối với các dự án cá nhân đơn giản chỉ có một môi trường duy nhất.
- Khi sử dụng các repository Private trên tài khoản cá nhân miễn phí (GitHub giới hạn tính năng protection rules cho tài khoản Pro/Team/Enterprise).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Tăng tính an toàn và bảo mật tuyệt đối cho Production. | Cấu hình phức tạp hơn so với dùng Repository Secrets thông thường. |
| Hỗ trợ Audit Log chi tiết (ai đã approve, deploy khi nào). | Một số tính năng bảo vệ yêu cầu tài khoản trả phí. |
| Giảm thiểu rủi ro deploy nhầm nhánh hoặc nhầm cấu hình. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Repository Secrets | Đơn giản nhưng không có quy tắc bảo vệ (Approval) và dùng chung cho mọi nhánh. |
| Terraform / Ansible | Quản lý hạ tầng tốt nhưng không có giao diện quản lý quy trình phê duyệt deploy trực quan như GitHub. |

## 9. How
Cấu hình sử dụng Environment trong file YAML của GitHub Actions:

```yaml
jobs:
  deploy-prod:
    runs-on: ubuntu-latest
    environment: 
      name: production
      url: https://myapp.com  # URL hiển thị trong danh sách deployment
    
    steps:
    - name: Deploy to Server
      run: echo "Deploying with secret: ${{ secrets.DB_PASSWORD }}"
```

Lưu ý: `${{ secrets.DB_PASSWORD }}` sẽ được ưu tiên lấy từ Environment `production` trước, nếu không có mới lấy từ Repository Secrets.

## 10. Production concerns
### Required Reviewers
Thiết lập tối đa 6 người hoặc team có quyền phê duyệt. Workflow sẽ tạm dừng (PENDING) cho đến khi có người nhấn nút Approve trên GitHub.

### Wait Timer
Cho phép thiết lập thời gian chờ (ví dụ 15 phút) sau khi workflow được kích hoạt mới bắt đầu thực thi, giúp có thời gian để hủy nếu phát hiện lỗi muộn.

### Deployment Branches
Giới hạn chỉ nhánh `main` mới được phép triển khai lên môi trường `production`.

## 11. Common mistakes
- Mistake: Không định nghĩa `environment` trong file YAML nhưng lại setup Secrets trong phần Environment của GitHub. Kết quả là workflow không lấy được secret.
  Fix: Luôn khai báo `environment: name` ở cấp độ Job.

- Mistake: Nhầm lẫn giữa Environment Secrets và Repository Secrets.
  Fix: Environment Secrets có độ ưu tiên cao hơn và được bảo vệ bởi các quy tắc approval.

## 12. Sample project
Thiết lập quy trình cho một ứng dụng React:
1. Deploy tự động lên môi trường `preview` mỗi khi có PR.
2. Deploy lên môi trường `production` từ nhánh `main` nhưng yêu cầu Lead Developer phê duyệt.

## 13. Interview
### Core Q&A
1. Q: Làm thế nào để giới hạn một môi trường chỉ cho phép deploy từ nhánh `main`?
   A: Truy cập Settings -> Environments -> Chọn môi trường -> Deployment branches -> Chọn "Required deployment branches".

2. Q: GitHub Environments có hỗ trợ rollback không?
   A: GitHub hiển thị lịch sử deployment cho từng môi trường. Bạn có thể nhấn vào một bản triển khai cũ và chọn "Re-run jobs" để thực hiện rollback thủ công.

### Scenario
"Hệ thống của bạn có 3 môi trường: Dev, Staging, Prod. Làm sao để đảm bảo file cấu hình `.env` của Prod không bao giờ bị dùng nhầm ở Dev?"
-> Trả lời: Tôi sẽ sử dụng GitHub Environments. Tôi tạo 3 environment tương ứng trên GitHub và đưa các biến vào Environment Secrets. Trong workflow YAML, tôi khai báo `environment` cho từng job. GitHub sẽ đảm bảo job Dev chỉ lấy được secrets của Dev và job Prod chỉ lấy được secrets của Prod.

## 14. References
- Official Docs: [Using environments for deployment](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)
- GitHub Blog: [Deployment protection rules](https://github.blog/changelog/2023-04-18-github-actions-deployment-protection-rules-is-now-ga/)

## 15. Real-world Code
Nghiên cứu cách các dự án mã nguồn mở lớn (như Next.js) sử dụng Environments để quản lý các bản preview deployment cho từng Pull Request.

## 16. Community
- Reddit: r/GitHubActions
- Stack Overflow: Tag [github-actions] [environments]
