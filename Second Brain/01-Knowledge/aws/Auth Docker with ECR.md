---
created: 2026-05-07
tags:
  - "#type/tutorial"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/security"
related:
  - "[[iam.md]]"
  - "[[Docker Daemon.md]]"
  - "[[Common Docker Commands.md]]"
  - "[[Amazon Elastic Container Registry (ECR).md]]"
---

## 1. What
Xác thực Docker với ECR là quá trình cấp quyền cho Docker client (trên máy local hoặc server) để có thể đẩy (push) hoặc tải (pull) các image từ Amazon Elastic Container Registry (ECR). Quá trình này sử dụng một mã thông báo (authorization token) tạm thời được tạo ra bởi AWS CLI.

## 2. Why
Khác với Docker Hub công cộng, ECR là một dịch vụ lưu trữ private. Để đảm bảo an ninh:
- AWS không cho phép sử dụng mật khẩu cố định cho Docker login.
- Cần một cơ chế xác thực dựa trên quyền hạn IAM (Identity and Access Management).
- Việc xác thực này giúp ngăn chặn truy cập trái phép vào các image chứa mã nguồn và bí mật kinh doanh của công ty.

## 3. Mental Model
Hãy tưởng tượng ECR là một **kho hàng bảo mật cao**.
- Docker client là **xe tải** muốn vào lấy hàng.
- Bình thường, bảo vệ không cho xe vào.
- Bạn phải dùng **thẻ nhân viên** (AWS Access Keys/IAM Role) đến gặp quản lý kho (AWS CLI) để lấy một **vé thông hành** (Authorization Token).
- Chiếc vé này chỉ có giá trị trong 12 giờ. Sau khi có vé, bảo vệ (Docker login) sẽ mở cổng cho xe tải của bạn vào.

## 4. Where it fits
Developer/CI-CD Runner -> **AWS CLI (get-login-password)** -> **Docker CLI (login)** -> **Amazon ECR**.

## 5. When to use
- Khi bạn cần upload image từ máy local lên ECR sau khi build.
- Khi server (EC2, EKS, Lambda) cần pull image về để triển khai ứng dụng.
- Trong các quy trình CI/CD (GitHub Actions, GitLab CI, Jenkins) để tự động hóa việc đóng gói và lưu trữ image.

## 6. When NOT to use
- Khi sử dụng các repository công cộng trên Docker Hub hoặc ECR Public Gallery (không yêu cầu login cho việc pull).
- Khi sử dụng các registry khác như GitHub Container Registry (GHCR) hoặc Google Artifact Registry (cơ chế xác thực sẽ khác).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Bảo mật cực cao nhờ tích hợp chặt chẽ với IAM. | Token mặc định chỉ có hiệu lực trong 12 giờ. |
| Không cần quản lý mật khẩu Docker thủ công. | Phụ thuộc vào việc cài đặt và cấu hình AWS CLI. |
| Kiểm soát quyền truy cập chi tiết đến từng repository. | Cần thực hiện login lại định kỳ hoặc tự động hóa. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Amazon ECR Credential Helper | Tự động hóa hoàn toàn, không cần chạy lệnh login thủ công, an toàn hơn vì không lưu token trong file config. |
| AWS Management Console | Chỉ dùng để xem, không thể dùng để push/pull image trực tiếp. |

## 9. How
Lệnh tiêu chuẩn để xác thực Docker với ECR (AWS CLI v2):

```bash
# Lấy password và thực hiện login trực tiếp
aws ecr get-login-password --region <region> | docker login --username AWS --password-stdin <aws_account_id>.dkr.ecr.<region>.amazonaws.com
```

Ví dụ thực tế cho vùng Singapore (ap-southeast-1):
```bash
aws ecr get-login-password --region ap-southeast-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.ap-southeast-1.amazonaws.com
```

Sau khi lệnh này thành công, bạn có thể thực hiện `docker push` hoặc `docker pull`.

## 10. Production concerns
### IAM Permissions
User hoặc Role thực hiện lệnh phải có quyền `ecr:GetAuthorizationToken`. Để push/pull, cần thêm các quyền như `ecr:BatchCheckLayerAvailability`, `ecr:PutImage`, `ecr:GetDownloadUrlForLayer`, v.v.

### Token Expiration
Vì token hết hạn sau 12 giờ, các script chạy dài hạn hoặc các worker nodes cần có cơ chế tự động refresh token (thường dùng IAM Role gắn trực tiếp vào EC2/EKS để tự động hóa).

### IP Restrictions
Có thể cấu hình IAM Policy để chỉ cho phép login từ các dãy IP hoặc VPC cụ thể nhằm tăng cường bảo mật.

## 11. Common mistakes
- Mistake: Sử dụng lệnh `aws ecr get-login` (đây là lệnh cũ của AWS CLI v1, hiện đã bị khai tử).
  Fix: Luôn sử dụng `aws ecr get-login-password` kết hợp với pipe `|` sang `docker login`.

- Mistake: Lưu trực tiếp output của token vào biến môi trường hoặc file.
  Fix: Sử dụng `--password-stdin` để tránh việc mật khẩu/token bị ghi lại trong lịch sử bash (history).

## 12. Sample project
Thiết lập GitHub Actions để push image lên ECR:
```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v1
  with:
    aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
    aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
    aws-region: ap-southeast-1

- name: Login to Amazon ECR
  id: login-ecr
  uses: aws-actions/amazon-ecr-login@v1

- name: Build, tag, and push image to Amazon ECR
  env:
    ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
    ECR_REPOSITORY: my-app
    IMAGE_TAG: ${{ github.sha }}
  run: |
    docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
    docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
```

## 13. Interview
### Core Q&A
1. Q: Thời gian tồn tại mặc định của ECR authorization token là bao lâu?
   A: 12 giờ. Đây là giới hạn cứng của AWS và không thể thay đổi.

2. Q: Tại sao username luôn là `AWS` khi thực hiện `docker login` vào ECR?
   A: Đây là quy định của AWS ECR. Thực thể thực sự được xác thực chính là token bạn truyền qua stdin, còn username `AWS` đóng vai trò là hằng số chỉ định cho Docker biết đang dùng cơ chế auth của ECR.

### Scenario
"Server của bạn đột nhiên không thể pull image từ ECR sau nửa ngày hoạt động bình thường. Bạn kiểm tra gì đầu tiên?"
-> Trả lời: Tôi sẽ kiểm tra xem token đã hết hạn chưa (vì thời hạn 12h). Nếu server dùng IAM Role, tôi sẽ kiểm tra xem role đó có đủ quyền `ecr:GetAuthorizationToken` hay không và kiểm tra logs của quá trình login tự động.

## 14. References
- Official Docs: [Authenticating with Amazon ECR](https://docs.aws.amazon.com/AmazonECR/latest/userguide/Registries.html#registry_auth)
- AWS CLI Command: [get-login-password reference](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/ecr/get-login-password.html)

## 15. Real-world Code
Script shell tự động login cho các máy trạm:
```bash
#!/bin/bash
REGION="ap-southeast-1"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

echo "Logging in to ECR in $REGION for account $ACCOUNT_ID..."
aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com
```

## 16. Community
- Stack Overflow: Tag [amazon-ecr] [docker-login].
- AWS News Blog: Thông báo về việc hỗ trợ Docker Credential Helper.
