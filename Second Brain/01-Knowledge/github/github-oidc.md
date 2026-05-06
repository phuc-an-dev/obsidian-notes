---
created: 2026-05-05
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/others"
  - "#topic/devops"
related:
  - "[[github-secrets]]"
  - "[[github-secret-injection]]"
  - "[[iam]]"
---

## 1. What
OIDC (OpenID Connect) cho GitHub Actions là cơ chế xác thực bảo mật cho phép workflow của bạn truy cập trực tiếp vào các tài nguyên trên Cloud (AWS, GCP, Azure, HashiCorp Vault) mà không cần phải lưu trữ các thông tin đăng nhập dài hạn (như AWS Access Keys) trong GitHub Secrets.

## 2. Why
Việc sử dụng các Secret dài hạn (Long-lived Secrets) mang lại nhiều rủi ro:
- **Nguy cơ rò rỉ**: Nếu Secret bị lộ, kẻ xấu có quyền truy cập vĩnh viễn cho đến khi bạn phát hiện và rotate key.
- **Quản lý phức tạp**: Việc rotate key định kỳ cho hàng trăm repository là một gánh nặng vận hành.
- **Thiếu tính linh động**: Khó giới hạn quyền hạn chi tiết cho từng workflow cụ thể.

OIDC giải quyết vấn đề này bằng cách sử dụng các **Short-lived Tokens** (token ngắn hạn) được tạo ra động cho mỗi lần chạy workflow.

## 3. Mental Model
Hãy tưởng tượng OIDC giống như việc sử dụng **"Thẻ từ tạm thời"** tại các khách sạn cao cấp:
- Thay vì cấp cho mỗi nhân viên một chìa khóa đồng (Long-lived Secret) để mở cửa kho.
- Khách sạn (Cloud Provider) thiết lập một quy định: "Nếu ai đó mang theo chứng minh thư từ Công ty GitHub (OIDC Token) và thuộc Team Marketing, hãy cấp cho họ một thẻ từ có tác dụng trong 1 tiếng để vào kho".
- Nhân viên GitHub chỉ cần xuất trình "chứng minh thư" để nhận thẻ tạm thời và làm việc.

## 4. Where it fits
`GitHub Actions Runner -> Request OIDC Token -> Exchange for Cloud Credentials (STS) -> Access Cloud Resources`

## 5. When to use
- Deploy ứng dụng lên AWS (S3, EKS, Lambda).
- Quản lý hạ tầng bằng Terraform/Pulumi trên GCP/Azure.
- Truy cập các bí mật trong HashiCorp Vault.
- Bất kỳ tương tác nào giữa GitHub Actions và các Cloud Provider hỗ trợ OIDC.

## 6. When NOT to use
- Các dịch vụ bên thứ ba chưa hỗ trợ OIDC (phải dùng Secret truyền thống).
- Các dự án local không sử dụng CI/CD cloud.
- Khi cấu hình Cloud IAM quá phức tạp vượt quá khả năng quản lý của team.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Không còn nỗi lo lộ Access Key vĩnh viễn. | Cấu hình ban đầu phức tạp hơn (cần thiết lập Trust Relationship). |
| Tự động rotate token sau mỗi job. | Phụ thuộc vào tính sẵn sàng của cả GitHub và Cloud Provider. |
| Phân quyền cực kỳ chi tiết (Granular) dựa trên Metadata (repo, branch). | Cần hiểu rõ về IAM Policy và OIDC Claims. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| GitHub Secrets | Đơn giản, dễ dùng nhưng kém bảo mật hơn do key tồn tại lâu dài. |
| Self-hosted Runners | Có thể gán IAM Role trực tiếp cho máy ảo chạy runner, nhưng tốn công bảo trì hạ tầng. |

## 9. How
### Step 1: Cấu hình trên Cloud (Ví dụ với AWS)
1. Tạo **Identity Provider** trong IAM với URL: `https://token.actions.githubusercontent.com`.
2. Tạo **IAM Role** với Trust Policy giới hạn cho repo của bạn:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Federated": "arn:aws:iam::123456789012:oidc-provider/token.actions.githubusercontent.com" },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringLike": { "token.actions.githubusercontent.com:sub": "repo:my-org/my-repo:*" }
      }
    }
  ]
}
```

### Step 2: Cấu hình trong Workflow YAML
Cần cấp quyền `id-token: write` để GitHub có thể tạo token:
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read
    steps:
      - name: Configure AWS Credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/my-github-oidc-role
          aws-region: us-east-1
      
      - name: List S3 Buckets
        run: aws s3 ls
```

## 10. Production concerns
### Scaling
Sử dụng **Wildcards** trong OIDC Claims (như `repo:my-org/*`) để cho phép nhiều repo trong cùng một Organization sử dụng chung một Role nếu cần thiết (tuy nhiên nên hạn chế để đảm bảo bảo mật).

### Failure
Nếu GitHub OIDC service bị lỗi, workflow sẽ không thể lấy được credentials. Cần có kế hoạch fallback (nhưng tuyệt đối không nên quay lại dùng secret dài hạn làm fallback chính).

### Monitoring
Kiểm tra CloudTrail (AWS) hoặc Audit Logs để xem các yêu cầu `AssumeRoleWithWebIdentity`. Bạn có thể thấy chính xác workflow run ID nào đã thực hiện hành động.

## 11. Common mistakes
- **Mistake**: Quên cấp quyền `id-token: write` trong file YAML. Action sẽ báo lỗi "Could not get ID token".
  **Fix**: Luôn thêm block `permissions` ở cấp độ job hoặc workflow.

- **Mistake**: Trust Policy quá rộng (ví dụ: `repo:my-org/*`).
  **Fix**: Giới hạn chính xác repo và tốt nhất là giới hạn cả branch (ví dụ: `repo:my-org/my-repo:ref:refs/heads/main`).

## 12. Sample project
Thiết lập một quy trình deploy Lambda function lên AWS mà hoàn toàn không sử dụng bất kỳ GitHub Secret nào liên quan đến AWS Credentials.

## 13. Interview
### Core Q&A
1. Q: "Subject" (sub) claim trong OIDC của GitHub chứa thông tin gì?
   A: Nó chứa thông tin định danh của workflow đang chạy, bao gồm: `repo:<org>/<repo>`, `ref:<branch>`, `environment:<env>`, v.v. Đây là căn cứ để Cloud Provider quyết định có cho phép truy cập hay không.

2. Q: OIDC có an toàn hơn GitHub Secrets không? Tại sao?
   A: An toàn hơn nhiều. OIDC loại bỏ hoàn toàn việc lưu trữ key dài hạn. Ngay cả khi ai đó chiếm được máy chủ của GitHub, họ cũng không tìm thấy "master key" nào của cloud của bạn, vì mỗi token chỉ có giá trị trong vài phút.

### Scenario
**Tình huống**: Bạn muốn chỉ những PR đã được merge vào `main` mới được phép deploy lên Production qua OIDC. Bạn cấu hình thế nào?
**Giải quyết**: Trong Trust Policy của Cloud IAM Role, cấu hình phần `Condition` để kiểm tra `sub` claim phải match với `repo:my-org/my-repo:ref:refs/heads/main`.

## 14. References
- GitHub Docs: [About security hardening with OpenID Connect](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)
- AWS Blog: [Use IAM roles to connect GitHub Actions to actions in AWS](https://aws.amazon.com/blogs/security/use-iam-roles-to-connect-github-actions-to-actions-in-aws/)

## 15. Real-world Code
N/A

## 16. Community
- Reddit: `r/aws`, `r/github`
- Talk: "Keyless Authentication on GitHub" at GitHub Universe.
#
