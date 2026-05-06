---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/ci-cd"
related:
  - "[[github-actions-ci.md]]"
  - "[[github-secrets.md]]"
  - "[[github-environments.md]]"
  - "[[github-actions-triggers.md]]"
  - "[[docker-hub.md]]"
---

## 1. What
GitHub Actions CD (thường là `cd.yml`) là quy trình Tự động hóa Triển khai (Continuous Deployment hoặc Continuous Delivery). Nó tự động đưa mã nguồn đã qua kiểm duyệt từ môi trường CI lên các môi trường thực thi như Staging, Production (ví dụ: AWS, Azure, Vercel, hoặc VPS riêng).

## 2. Why
Trước khi có CD, việc triển khai thường làm thủ công (FTP, SSH rồi `git pull`). Cách này rất chậm, dễ sai sót (quên update biến môi trường, nhầm version) và gây gián đoạn dịch vụ. CD giúp quá trình phát hành phần mềm diễn ra nhanh chóng, có thể lặp lại và an toàn nhờ các bước rollback tự động nếu có lỗi.

## 3. Mental Model
Hãy tưởng tượng CD giống như một **"Nhân viên vận chuyển chuyên nghiệp"**. Sau khi kiện hàng đã qua trạm kiểm định (CI), nhân viên này sẽ:
1. Nhận kiện hàng đã đóng gói (Artifact).
2. Mang đến đúng địa chỉ yêu cầu (Môi trường Staging/Prod).
3. Đặt kiện hàng vào đúng vị trí và khởi động nó (Deployment).
Nếu khách hàng (User) báo kiện hàng hỏng, nhân viên này ngay lập tức đổi lại kiện hàng cũ trước đó (Rollback).

## 4. Where it fits
Vị trí trong luồng:
`GitHub CI (Pass) -> Trigger CD -> Build Docker Image/Artifact -> Push to Registry -> Update Infrastructure/Server`

Thường trigger sau khi một Pull Request được merge vào nhánh `main` hoặc khi có một `tag` mới được tạo.

## 5. When to use
- Khi muốn giảm thời gian từ lúc code xong đến lúc user sử dụng được (Time-to-market).
- Khi triển khai lên các môi trường hiện đại như Kubernetes, Serverless, hoặc PaaS (Vercel, Heroku).
- Khi hệ thống có nhiều microservices cần triển khai đồng bộ.

## 6. When NOT to use
- Các hệ thống cực kỳ nhạy cảm cần sự phê duyệt thủ công của nhiều cấp lãnh đạo (Trường hợp này dùng Continuous Delivery - dừng lại ở bước chờ Approve).
- Môi trường hạ tầng không có kết nối internet hoặc bị giới hạn bởi firewall khắt khe mà GitHub Cloud không thể truy cập.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Triển khai nhanh, nhất quán và ít lỗi. | Rủi ro cao nếu bộ test (CI) không đủ tốt, dễ mang lỗi lên Production. |
| Dễ dàng Rollback về phiên bản cũ. | Đòi hỏi kiến thức về bảo mật secrets và quản lý hạ tầng. |
| Giải phóng lập trình viên khỏi các tác vụ vận hành lặp lại. | Cần đầu tư thời gian setup ban đầu khá lớn. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| ArgoCD | Chuyên dụng cho Kubernetes (GitOps), tự động đồng bộ trạng thái cluster. |
| AWS CodeDeploy | Tối ưu nếu bạn nằm hoàn toàn trong hệ sinh thái AWS. |
| Jenkins | Mạnh mẽ về plugin nhưng quản lý script triển khai phức tạp hơn. |

## 9. How
Ví dụ một file `cd.yml` triển khai lên AWS S3 (Static Site):

```yaml
name: Deployment (CD)

on:
  workflow_run:
    workflows: ["Node.js CI"]
    types: [completed]
    branches: [main]

jobs:
  deploy:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    runs-on: ubuntu-latest
    environment: production

    steps:
    - name: Download Artifact
      uses: actions/download-artifact@v4
      with:
        name: build-output

    - name: Configure AWS Credentials
      uses: aws-actions/configure-aws-credentials@v4
      with:
        aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
        aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
        aws-region: us-east-1

    - name: Deploy to S3
      run: aws s3 sync ./dist s3://my-production-bucket --delete
```

## 10. Production concerns
### Scaling
Sử dụng Blue/Green Deployment hoặc Canary Deployment để giảm thiểu ảnh hưởng khi có lỗi trên diện rộng.

### Failure
Luôn lưu lại `SHA` của commit hoặc `Tag` của image. Nếu deploy lỗi, workflow CD mới phải có khả năng deploy lại version ổn định gần nhất ngay lập tức.

### Monitoring
Tích hợp `Health Check` sau khi deploy. Nếu endpoint `/health` trả về lỗi, CD phải báo động (Slack/Email) hoặc tự động trigger rollback.

## 11. Common mistakes
- Mistake: Deploy trực tiếp từ nhánh `develop` lên Production.
  Fix: Chỉ trigger CD Production từ nhánh `main` hoặc thông qua `git tags`.

- Mistake: Để lộ mật khẩu, API key trong file YAML.
  Fix: Luôn sử dụng `GitHub Secrets` và `Environments Secrets`.

## 12. Sample project
Thiết lập workflow CD để:
1. Build Docker Image cho một ứng dụng Java.
2. Push Image lên Docker Hub/AWS ECR.
3. SSH vào VPS và chạy `docker-compose up -d`.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa Continuous Delivery và Continuous Deployment là gì?
   A: Cả hai đều tự động hóa việc build và test. Tuy nhiên, Delivery dừng lại ở bước chuẩn bị sẵn sàng và cần con người nhấn nút Deploy. Deployment tự động đưa code lên Production ngay khi qua được CI.

2. Q: Làm thế nào để đảm bảo chỉ những người có quyền mới được deploy lên Production?
   A: Sử dụng tính năng "Environments" của GitHub để thiết lập "Required Reviewers" trước khi Job CD được chạy.

### Scenario
"Sau khi deploy thành công, bạn phát hiện ra một bug nghiêm trọng làm sập hệ thống. Bạn làm gì?"
-> Trả lời:
1. Ngay lập tức thực hiện Rollback về version ổn định trước đó (sử dụng GitHub Actions để chạy lại workflow cũ hoặc deploy tag cũ).
2. Sau khi hệ thống ổn định, mới tiến hành debug và sửa lỗi ở nhánh feature, không sửa trực tiếp trên Production.

## 14. References
- Official Docs: [About continuous deployment](https://docs.github.com/en/actions/deployment/about-deployment/about-continuous-deployment)
- GitHub Actions: [Deploying with GitHub Actions](https://docs.github.com/en/actions/deployment/deploying-with-github-actions)

## 15. Real-world Code
- [Vercel GitHub Integration](https://github.com/vercel/vercel)
- [Terraform GitHub Actions](https://github.com/hashicorp/setup-terraform)

## 16. Community
- Reddit: r/DevOps
- Stack Overflow: Tag [github-actions] [continuous-deployment]
- Blog: Simo Ahava's DevOps posts
