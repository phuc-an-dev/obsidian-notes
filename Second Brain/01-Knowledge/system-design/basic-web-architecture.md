---
created: 2026-04-24
tags:
  - "#type/pattern"
  - "#status/draft"
  - "#lang/system-design"
  - "#topic/architecture"
related:
  - "[[route53]]"
  - "[[cloudfront]]"
  - "[[s3]]"
  - "[[ec2]]"
  - "[[rds]]"
---

## 1. What
Đây là mô hình kiến trúc web tiêu chuẩn trên AWS, kết hợp giữa nội dung tĩnh (Static Content) và nội dung động (Dynamic Content).
- **Static**: Route53 -> CloudFront -> S3.
- **Dynamic**: Route53 -> CloudFront -> ALB -> EC2 -> RDS.

## 2. Why
Việc tách biệt nội dung tĩnh và động giúp tối ưu hóa hiệu năng và chi phí. Nội dung tĩnh được phân phối qua CDN (CloudFront) để giảm độ trễ, trong khi nội dung động được xử lý bởi các server (EC2) nằm sau bộ cân bằng tải (ALB) để đảm bảo tính sẵn sàng cao và khả năng mở rộng.

## 3. Mental Model
Hãy tưởng tượng một nhà hàng hiện đại:
- **S3 & CloudFront**: Giống như quầy phục vụ đồ ăn sẵn (bánh mì, nước ngọt). Khách hàng lấy ngay tại quầy gần cửa nhất mà không cần đợi đầu bếp.
- **ALB, EC2 & RDS**: Giống như khu vực bếp nấu theo yêu cầu. Người quản lý (ALB) chia việc cho các đầu bếp (EC2), và các đầu bếp lấy nguyên liệu từ kho thực phẩm (RDS) để chế biến.

## 4. Where it fits
User -> Route 53 (DNS) -> CloudFront (CDN)
   |-> Path /static/* -> S3 (Static Website)
   |-> Path /api/* -> Application Load Balancer -> EC2 Auto Scaling -> RDS (Database)

## 5. When to use
- Hầu hết các ứng dụng web hiện đại (React/Angular/Vue frontend + Java/NodeJS/Python backend).
- Các hệ thống yêu cầu cả tốc độ tải trang nhanh và khả năng xử lý logic phức tạp.
- Khi cần một kiến trúc có khả năng chịu lỗi và tự động mở rộng.

## 6. When NOT to use
- Ứng dụng cực kỳ đơn giản (chỉ cần 1 máy chủ EC2 duy nhất để tiết kiệm chi phí).
- Ứng dụng hoàn toàn Serverless (nên dùng API Gateway + Lambda + DynamoDB).
- Khi ngân sách cực thấp (vì ALB và RDS có chi phí cố định hàng tháng khá cao).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Hiệu năng tối ưu cho cả tĩnh và động | Cấu hình phức tạp (VPC, Subnets, Security Groups) |
| Khả năng mở rộng (Scaling) cực tốt | Chi phí duy trì các thành phần managed (ALB, RDS) |
| Bảo mật nhiều lớp (WAF, Private Subnets) | Đòi hỏi kiến thức vận hành DevOps tốt |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| AWS Amplify | Đơn giản hơn, tự động hóa mọi thứ nhưng khó tùy chỉnh sâu. |
| Serverless Architecture | Rẻ hơn khi traffic thấp, không cần quản lý server nhưng có giới hạn về runtime. |

## 9. How
Sơ đồ kết nối:
1. **Route 53**: Trỏ tên miền về CloudFront.
2. **CloudFront**:
   - Origin 1: S3 Bucket (chứa file build frontend).
   - Origin 2: Application Load Balancer (ALB).
3. **ALB**: Phân phối traffic vào các instance EC2 trong Private Subnets.
4. **EC2**: Chạy ứng dụng backend, kết nối đến RDS.
5. **RDS**: Lưu trữ dữ liệu quan hệ, nằm trong Subnet biệt lập.

## 10. Production concerns
### Scaling
- Frontend tự động scale nhờ S3/CloudFront.
- Backend scale bằng **Auto Scaling Groups** dựa trên CPU/RAM của EC2.
- RDS scale bằng cách nâng cấp instance hoặc thêm Read Replicas.

### Failure
- Multi-AZ cho RDS để failover tự động.
- ALB tự động bỏ qua các EC2 bị hỏng (Health Checks).

### Monitoring
Theo dõi toàn bộ flow qua CloudWatch và AWS X-Ray.

## 11. Common mistakes
- Mistake: Để EC2 và RDS trong Public Subnet.
  Fix: Luôn để chúng trong Private Subnet, chỉ ALB nằm trong Public Subnet.

- Mistake: Không cấu hình HTTPS từ CloudFront đến ALB.
  Fix: Sử dụng ACM để cấp chứng chỉ SSL cho cả CloudFront và ALB.

## 12. Sample project
Xây dựng một trang thương mại điện tử: Ảnh sản phẩm và giao diện React lưu trên S3/CloudFront. API xử lý đơn hàng chạy trên EC2, thông tin đơn hàng và người dùng lưu trong RDS MySQL.

## 13. Interview
### Core Q&A
1. Q: Tại sao lại đặt CloudFront đứng trước cả ALB thay vì để người dùng gọi trực tiếp ALB?
   A: Để tận dụng mạng lưới Edge Locations của AWS giúp giảm độ trễ DNS/TCP handshake, đồng thời có thể tích hợp AWS WAF để bảo vệ toàn bộ hệ thống khỏi tấn công từ lớp biên.

### Scenario
1. Q: Làm thế nào để đảm bảo người dùng không thể truy cập trực tiếp vào S3 mà phải đi qua CloudFront?
   A: Sử dụng **Origin Access Control (OAC)**. S3 bucket sẽ được cấu hình chỉ cho phép truy cập từ định danh của CloudFront.

## 14. References
- AWS Reference Architectures: https://aws.amazon.com/architecture/
- Standard Three-Tier Web Hierarchy: https://aws.amazon.com/getting-started/guides/deploy-webapp-ec2/

## 15. Real-world Code
Sử dụng các module Terraform của cộng đồng (như `terraform-aws-modules`) để triển khai nhanh chóng hạ tầng này.

## 16. Community
- YouTube: "Building a Three-Tier Architecture on AWS".
- Blog: "Architecture best practices for web apps on AWS".
