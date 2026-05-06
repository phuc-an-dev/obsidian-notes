---
created: 2026-05-05
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/compute"
related:
  - "[[aws-ec2-security-groups.md]]"
  - "[[aws-ec2-key-pairs.md]]"
  - "[[Instance Metadata Service (IMDS).md]]"
---

## 1. What
Amazon Elastic Compute Cloud (EC2) là một dịch vụ web cung cấp khả năng tính toán có thể thay đổi quy mô (resizable compute capacity) trong đám mây AWS. Nó cho phép người dùng thuê các máy chủ ảo (Instances) để chạy các ứng dụng mà không cần đầu tư vào phần cứng vật lý.

## 2. Why
Trước khi có EC2, việc triển khai ứng dụng đòi hỏi phải mua sắm máy chủ vật lý, thiết lập trung tâm dữ liệu và quản lý phần cứng, tốn nhiều thời gian và chi phí đầu tư ban đầu lớn (CapEx). EC2 ra đời để chuyển đổi chi phí này thành chi phí vận hành (OpEx), giúp triển khai server chỉ trong vài phút và tự động hóa việc mở rộng.

## 3. Mental Model
Hãy coi EC2 như việc bạn đi thuê xe ô tô. Bạn không cần mua xe (Hardware), bạn chỉ cần chọn loại xe phù hợp (Instance Type): xe tải để chở hàng nặng (Compute Optimized), xe đua để chạy nhanh (Memory Optimized) hoặc xe phổ thông (General Purpose). Bạn cũng có thể chọn cách trả tiền: thuê theo giờ (On-Demand), đặt trước dài hạn (Reserved) hoặc thuê xe cũ giá rẻ nhưng có thể bị lấy lại bất cứ lúc nào (Spot).

## 4. Where it fits
User -> Internet Gateway -> Load Balancer -> EC2 Instance (VPC Subnet) -> EBS Volume (Storage).

## 5. When to use
- Triển khai các ứng dụng web (Nginx, Apache, Node.js, Spring Boot).
- Chạy các cơ sở dữ liệu tự quản lý (MySQL, PostgreSQL, MongoDB).
- Xử lý dữ liệu lớn (Big Data) và tính toán hiệu năng cao (HPC).
- Môi trường phát triển (Dev/Test) cần thay đổi cấu hình liên tục.

## 6. When NOT to use
- Các tác vụ cực ngắn và không liên tục (nên dùng AWS Lambda để tiết kiệm).
- Ứng dụng container hóa đơn giản (nên dùng Fargate để không phải quản lý máy chủ).
- Các trang web tĩnh (nên dùng S3 + CloudFront).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Kiểm soát hoàn toàn hệ điều hành (Root access). | Phải tự quản lý bảo mật, vá lỗi (Patching) OS. |
| Thay đổi cấu hình CPU/RAM dễ dàng. | Phải tự thiết lập cơ chế High Availability (Multi-AZ). |
| Đa dạng lựa chọn về cấu hình phần cứng. | Chi phí có thể cao nếu không quản lý tốt Instance Lifecycle. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| AWS Lambda | Serverless, không cần quản lý server, chỉ chạy khi có event. |
| AWS Fargate | Container-as-a-service, không cần quản lý EC2 bên dưới. |
| AWS Lightsail | Dành cho các dự án nhỏ, cấu hình đơn giản và giá cố định. |

## 9. How
Sử dụng AWS CLI để khởi tạo một EC2 instance:
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
Sử dụng Auto Scaling Group (ASG) để tự động thêm/bớt instance dựa trên CloudWatch Metrics (CPU, Network).

### Failure
Luôn triển khai EC2 trên ít nhất 2 Availability Zones (Multi-AZ) để đảm bảo khi một Data Center gặp sự cố, hệ thống vẫn hoạt động.

### Monitoring
Theo dõi CPU Utilization, Disk I/O, và Network In/Out thông qua Amazon CloudWatch.

## 11. Common mistakes
- Mistake: Sử dụng On-Demand cho các ứng dụng chạy ổn định 24/7 trong thời gian dài.
  Fix: Chuyển sang Reserved Instances hoặc Savings Plans để tiết kiệm tới 72% chi phí.

- Mistake: Lưu trữ dữ liệu quan trọng trực tiếp trên Instance Store (ổ cứng tạm thời).
  Fix: Luôn sử dụng EBS Volumes cho dữ liệu cần bền vững vì Instance Store sẽ mất dữ liệu khi stop/terminate instance.

## 12. Sample project
Thiết lập một hệ thống WordPress chịu tải cao: Sử dụng 1 EC2 chạy Nginx/PHP, tách Database sang RDS, lưu trữ ảnh trên S3 và cấu hình Auto Scaling Group để xử lý khi có lượng truy cập đột biến.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa On-Demand, Reserved Instances và Spot Instances là gì?
   A: On-Demand trả theo giây, linh hoạt nhất. Reserved Instances yêu cầu cam kết 1-3 năm để được giảm giá sâu. Spot Instances dùng tài nguyên dư thừa của AWS với giá rẻ nhất (giảm tới 90%) nhưng có thể bị AWS thu hồi trong 2 phút báo trước.
2. Q: Instance Store và EBS khác nhau như thế nào?
   A: Instance Store được gắn vật lý vào máy chủ chủ (Host), tốc độ rất cao nhưng dữ liệu mất khi instance bị stop hoặc lỗi phần cứng. EBS là network drive, bền vững hơn, có thể snapshot và tách rời khỏi instance.

### Scenario
Hệ thống của bạn đang chạy một batch job xử lý video vào mỗi đêm lúc 2 giờ sáng và không quá quan trọng về thời gian hoàn thành chính xác. Bạn sẽ chọn Purchase Option nào?
Trả lời: Nên chọn Spot Instances vì đây là tác vụ có thể chịu được gián đoạn (fault-tolerant) và không yêu cầu tính liên tục khắt khe, giúp tối ưu hóa chi phí đến mức tối đa.

## 14. References
- Official Docs: https://docs.aws.amazon.com/ec2/
- GitHub Repo: https://github.com/aws/aws-cli
- Spec / RFC: N/A
- Changelog: https://aws.amazon.com/releasenotes/?tag=releasenotes%23keywords%23ec2

## 15. Real-world Code
- Terraform module for EC2: https://github.com/terraform-aws-modules/terraform-aws-ec2-instance

## 16. Community
- Reddit: https://www.reddit.com/r/aws/
- Stack Overflow: https://stackoverflow.com/questions/tagged/amazon-ec2
- Blog: https://aws.amazon.com/blogs/compute/
- Talk: AWS re:Invent EC2 Deep Dive sessions on YouTube.
