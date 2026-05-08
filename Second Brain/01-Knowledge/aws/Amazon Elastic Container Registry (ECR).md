---
created: 2026-05-07
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/aws"
related:
  - "[[iam.md]]"
  - "[[Docker Image.md]]"
  - "[[Auth Docker with ECR.md]]"
---

## 1. What
Amazon Elastic Container Registry (ECR) là một dịch vụ lưu trữ (registry) container image được quản lý hoàn toàn bởi AWS. Nó cho phép các nhà phát triển dễ dàng lưu trữ, quản lý và triển khai các Docker container images một cách an toàn và tin cậy.

## 2. Why
Trước khi có ECR, các doanh nghiệp dùng AWS thường phải tự vận hành Docker Registry riêng trên EC2 hoặc dùng Docker Hub. ECR ra đời để:
- **Tích hợp sâu**: Tự động kết nối với ECS, EKS, và Lambda mà không cần quản lý credentials phức tạp.
- **Bảo mật**: Sử dụng IAM để kiểm soát quyền truy cập chi tiết.
- **Tính sẵn sàng**: Dữ liệu được lưu trữ trên S3 với độ bền cao (11 nines) và tự động replicate trong Region.

## 3. Mental Model
Hãy tưởng tượng ECR là một **"Thư viện bảo mật cực cao"** dành riêng cho các "Bản thiết kế" (Docker Images).
- Bạn không thể tự ý bước vào; bạn cần thẻ từ (IAM) và phải được lễ tân (AWS CLI) cấp vé tạm thời.
- Mỗi bản thiết kế được đặt trong một ngăn tủ riêng (Repository).
- Thư viện này có máy quét tự động (Image Scanning) để kiểm tra xem bản thiết kế của bạn có "lỗ hổng" nào không trước khi cho phép mang ra công trường (Deployment).

## 4. Where it fits
Build Server (CI/CD) -> `docker push` -> **Amazon ECR** -> `docker pull` -> EKS / ECS / Fargate / Lambda.

## 5. When to use
- Khi bạn chạy các ứng dụng container trên hạ tầng AWS.
- Khi cần lưu trữ image riêng tư (Private) cho doanh nghiệp.
- Khi muốn tự động quét lỗ hổng bảo mật của image ngay trong registry.
- Khi cần quản lý vòng đời image (ví dụ: tự động xóa các image cũ sau 30 ngày).

## 6. When NOT to use
- Khi bạn muốn chia sẻ image công khai cho toàn thế giới (nên dùng Docker Hub hoặc ECR Public Gallery).
- Trong các môi trường thuần on-premise không có kết nối internet tới AWS.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Quản lý hoàn toàn bởi AWS, không cần cài đặt. | Chi phí lưu trữ và băng thông truyền tải (data transfer) có thể cao nếu không quản lý tốt. |
| Tốc độ pull cực nhanh khi chạy trong cùng Region. | Phụ thuộc vào hệ sinh thái AWS (Vendor lock-in). |
| Quét bảo mật tích hợp sẵn. | Cơ chế auth bằng token 12h đôi khi gây phiền phức cho các node on-premise. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Docker Hub | Phổ biến nhất, cộng đồng lớn, nhưng tích hợp với AWS kém hơn ECR. |
| GitHub Container Registry (GHCR) | Tốt nếu bạn dùng GitHub Actions làm CI/CD chính. |
| JFrog Artifactory | Giải pháp Enterprise hỗ trợ nhiều loại artifact, không chỉ container. |

## 9. How
Quy trình cơ bản sử dụng ECR:
1. **Tạo repository**: 
   ```bash
   aws ecr create-repository --repository-name my-app
   ```
2. **Authenticate**: (Xem chi tiết tại [[Auth Docker with ECR.md]])
3. **Tag image**:
   ```bash
   docker tag my-app:latest <aws_account_id>.dkr.ecr.<region>.amazonaws.com/my-app:v1
   ```
4. **Push image**:
   ```bash
   docker push <aws_account_id>.dkr.ecr.<region>.amazonaws.com/my-app:v1
   ```

## 10. Production concerns
### Lifecycle Policies
Sử dụng chính sách vòng đời để tự động dọn dẹp các "untagged images" hoặc giới hạn số lượng bản build cũ nhằm tối ưu chi phí lưu trữ.

### Image Scanning
Bật "Scan on push" để AWS tự động quét mã nguồn bên trong image và đưa ra báo cáo về các lỗ hổng (CVE) có mức độ nghiêm trọng (High/Critical).

### Cross-Region Replication
Nếu bạn triển khai ứng dụng trên nhiều Region, hãy bật tính năng Replication để ECR tự động copy image sang các Region khác, giảm độ trễ khi pull.

## 11. Common mistakes
- Mistake: Không sử dụng Lifecycle Policies dẫn đến chi phí lưu trữ phình to theo thời gian.
  Fix: Luôn cấu hình quy tắc xóa các image cũ hơn N ngày.

- Mistake: Để repository ở chế độ Public một cách vô ý.
  Fix: Kiểm tra kỹ thiết lập "Visibility" khi tạo repository.

## 12. Sample project
Thiết lập một "Secure Pipeline": CI server build image -> Chạy `trivy` scan local -> Push lên ECR -> ECR tự động scan lần 2 -> Nếu kết quả scan của ECR có lỗi Critical, Lambda sẽ chặn việc deploy lên EKS.

## 13. Interview
### Core Q&A
1. Q: Làm thế nào để kiểm soát ai có thể pull image từ ECR?
   A: Sử dụng IAM Policy (Identity-based) gắn cho User/Role hoặc ECR Repository Policy (Resource-based) gắn trực tiếp trên repository.

2. Q: Image Scanning trong ECR dựa trên công cụ nào?
   A: ECR sử dụng cơ sở dữ liệu lỗ hổng bảo mật từ dự án mã nguồn mở **Clair** (cho Basic scanning) hoặc **Amazon Inspector** (cho Enhanced scanning).

### Scenario
"Image của bạn đã được push lên ECR thành công, nhưng EKS không thể pull về và báo lỗi ImagePullBackOff. Bạn sẽ kiểm tra gì?"
-> Trả lời: 
1. Kiểm tra IAM Role của EKS Worker Node có quyền `ecr:BatchGetImage` và `ecr:GetDownloadUrlForLayer` chưa.
2. Kiểm tra xem EKS và ECR có cùng Region không, nếu khác phải cấu hình auth đặc biệt.
3. Kiểm tra Repository Policy có đang chặn truy cập từ VPC của EKS hay không.

## 14. References
- AWS ECR User Guide: https://docs.aws.amazon.com/AmazonECR/latest/userguide/
- Pricing: https://aws.amazon.com/ecr/pricing/

## 15. Real-world Code
Sử dụng Terraform để tạo ECR repository với Lifecycle Policy:
```hcl
resource "aws_ecr_repository" "app" {
  name                 = "my-app"
  image_tag_mutability = "IMMUTABLE"
  image_scanning_configuration {
    scan_on_push = true
  }
}

resource "aws_ecr_lifecycle_policy" "cleanup" {
  repository = aws_ecr_repository.app.name
  policy = jsonencode({
    rules = [{
      rulePriority = 1
      description  = "Keep last 30 images"
      selection = {
        tagStatus     = "any"
        countType     = "imageCountMoreThan"
        countNumber   = 30
      }
      action = { type = "expire" }
    }]
  })
}
```

## 16. Community
- YouTube: "Amazon ECR Deep Dive" - AWS Events.
- Stack Overflow: Tag [amazon-ecr].
- Blog: AWS Containers Blog.
