---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/performance"
related:
  - "[[s3]]"
  - "[[MySQL on EC2 vs AWS RDS]]"
  - "[[Associate Elastic IP address in AWS]]"
---

## 1. What
AWS EC2 (Elastic Compute Cloud) là một dịch vụ web cung cấp năng lực tính toán có thể thay đổi quy mô (resizable compute capacity) trong đám mây. Nó cho phép người dùng thuê các máy chủ ảo (instances) để chạy các ứng dụng của riêng họ.

## 2. Why
Trước khi có EC2, các doanh nghiệp phải mua, lắp đặt và bảo trì phần cứng máy chủ vật lý, một quá trình tốn kém và mất thời gian. EC2 ra đời để biến hạ tầng tính toán thành một dịch vụ "pay-as-you-go", giúp triển khai máy chủ chỉ trong vài phút và có thể mở rộng/thu nhỏ linh hoạt theo nhu cầu.

## 3. Mental Model
Hãy tưởng tượng EC2 như việc bạn thuê một chiếc xe máy. Thay vì phải mua xe (mua server vật lý), bạn chỉ cần trả tiền thuê theo giờ hoặc theo km. Nếu bạn đi một mình, bạn thuê xe nhỏ; nếu đi đông, bạn thuê xe lớn hoặc thuê nhiều xe cùng lúc. Bạn có toàn quyền lái xe (quyền root/admin) và chịu trách nhiệm đổ xăng, bảo dưỡng máy móc bên trong (quản lý OS/App).

## 4. Where it fits
VPC (Virtual Private Cloud) -> Subnet -> **EC2 Instance** -> EBS (Elastic Block Store) -> Application.

## 5. When to use
- Chạy các ứng dụng web server (Node.js, Spring Boot, Nginx).
- Xử lý dữ liệu lớn hoặc tính toán khoa học yêu cầu GPU/CPU mạnh.
- Chạy các cơ sở dữ liệu tự quản lý (Self-managed databases).
- Các ứng dụng yêu cầu quyền truy cập sâu vào hệ điều hành.

## 6. When NOT to use
- Khi có thể dùng Serverless (AWS Lambda) để tiết kiệm chi phí cho các tác vụ ngắn hạn.
- Khi chỉ cần chạy container đơn giản (nên dùng AWS ECS hoặc Fargate).
- Khi chỉ cần hosting trang web tĩnh (nên dùng S3 + CloudFront).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Toàn quyền kiểm soát hệ điều hành | Phải tự quản lý bảo mật, cập nhật OS |
| Đa dạng loại cấu hình (Family types) | Chi phí có thể tăng cao nếu không quản lý tốt |
| Hỗ trợ nhiều hệ điều hành (AMI) | Mất thời gian cấu hình ban đầu hơn so với Managed Services |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| AWS Lambda | Serverless, rẻ hơn cho task ngắn, không cần quản lý OS. |
| AWS Fargate | Chạy container mà không cần quản lý EC2 instances. |
| DigitalOcean Droplets | Đơn giản hơn, chi phí cố định, ít tính năng enterprise hơn. |

## 9. How
Triển khai cơ bản qua AWS CLI:
```bash
aws ec2 run-instances \
    --image-id ami-0abcdef1234567890 \
    --count 1 \
    --instance-type t2.micro \
    --key-name MyKeyPair \
    --security-group-ids sg-903004f8 \
    --subnet-id subnet-6e7f829e
```

## 10. Production concerns
### Scaling
Sử dụng **Auto Scaling Groups (ASG)** kết hợp với **Elastic Load Balancer (ELB)** để tự động thêm/bớt instance dựa trên traffic hoặc CPU.

### Failure
Sử dụng **Multi-AZ deployment** để đảm bảo ứng dụng vẫn hoạt động nếu một Data Center của AWS gặp sự cố.

### Monitoring
Sử dụng **Amazon CloudWatch** để theo dõi CPU, Disk I/O, và Network traffic. Cấu hình Alarms để thông báo khi hệ thống quá tải.

## 11. Common mistakes
- Mistake: Để Security Group mở port 22 (SSH) cho toàn bộ internet (0.0.0.0/0).
  Fix: Chỉ cho phép IP cụ thể của bạn hoặc dùng AWS Systems Manager Session Manager.

- Mistake: Lưu trữ dữ liệu quan trọng trực tiếp trên ổ đĩa tạm thời (Instance Store).
  Fix: Luôn sử dụng Amazon EBS cho dữ liệu cần lưu trữ bền vững.

## 12. Sample project
Thiết lập một Cluster gồm 2 máy chủ EC2 chạy Nginx, nằm sau một Application Load Balancer, có khả năng tự phục hồi nếu một máy chủ bị chết.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa Stop và Terminate một EC2 Instance là gì?
   A: Stop giống như tắt máy tính (dữ liệu trên EBS vẫn giữ lại, không tốn phí tính toán). Terminate là xóa vĩnh viễn máy chủ (EBS thường bị xóa theo, không thể khôi phục lại máy chủ đó).

### Scenario
1. Q: Làm thế nào để tiết kiệm chi phí cho các máy chủ chạy môi trường Development chỉ dùng vào giờ hành chính?
   A: Tôi sẽ sử dụng AWS Instance Scheduler để tự động tắt máy vào 6h tối và bật lại vào 8h sáng, hoặc sử dụng Spot Instances nếu ứng dụng có khả năng chịu lỗi tốt.

## 14. References
- Official Docs: https://docs.aws.amazon.com/ec2/
- Instance Types: https://aws.amazon.com/ec2/instance-types/
- Pricing: https://aws.amazon.com/ec2/pricing/

## 15. Real-world Code
Nghiên cứu các Terraform module hoặc AWS CDK code để triển khai hạ tầng EC2 một cách chuẩn hóa (Infrastructure as Code).

## 16. Community
- Reddit: r/aws
- Stack Overflow: [amazon-ec2] tag
- Blog: AWS Architecture Blog
