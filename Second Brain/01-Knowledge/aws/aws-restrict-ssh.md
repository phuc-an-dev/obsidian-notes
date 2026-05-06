---
created: 2026-05-05
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/security"
related:
  - "[[aws-rds-public-accessible]]"
---

## 1. What
AWS Restrict SSH là việc áp dụng các quy tắc bảo mật (Security Group rules) để giới hạn quyền truy cập vào port 22 (SSH) của các EC2 instance. Thay vì mở cho toàn thế giới (`0.0.0.0/0`), chúng ta chỉ cho phép các dải IP cụ thể hoặc sử dụng các cơ chế kết nối thay thế.

## 2. Why
Port 22 là mục tiêu hàng đầu của các bot scanner trên internet. Nếu để SSH mở công khai, instance của bạn sẽ bị tấn công Brute-force hàng nghìn lần mỗi giờ. Ngay cả khi có private key, các lỗ hổng chưa được vá trong SSH daemon vẫn có thể bị khai thác.

## 3. Mental Model
Việc mở SSH `0.0.0.0/0` giống như việc bạn để cửa nhà mình ngay mặt đường và ai cũng có thể đến cầm nắm đấm cửa để thử vặn. Việc restrict SSH giống như việc bạn đặt ngôi nhà trong một khu dân cư có cổng bảo vệ (Gated Community), chỉ những ai có thẻ cư dân (IP tin cậy) mới được đến gần cửa nhà.

## 4. Where it fits
Internet -> Security Group (Port 22 Filter) -> EC2 Instance.

## 5. When to use
Mọi EC2 Linux instance đều cần được giới hạn quyền truy cập SSH ngay từ khi khởi tạo. Đây là tiêu chuẩn bảo mật cơ bản nhất (Security 101).

## 6. When NOT to use
Không bao giờ mở SSH `0.0.0.0/0` ngay cả trong môi trường lab. Nếu bạn không có IP tĩnh, hãy sử dụng AWS Systems Manager Session Manager thay vì mở port 22.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Giảm thiểu 99% các cuộc tấn công quét port tự động. | Gây khó khăn nếu IP của lập trình viên thay đổi liên tục (ví dụ: dùng mạng quán cafe). |
| Dễ dàng kiểm soát ai có quyền truy cập quản trị. | Cần quản lý danh sách IP (Whitelist) thủ công. |
| Tuân thủ các tiêu chuẩn bảo mật quốc tế. | Có thể gây mất kết nối nếu cấu hình sai IP của chính mình. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| AWS Systems Manager Session Manager | Tốt nhất. Không cần mở port 22, không cần public IP, login qua IAM role. |
| EC2 Instance Connect | Cho phép dùng trình duyệt để SSH, vẫn cần port 22 mở cho dải IP của AWS. |
| Bastion Host / Jump Server | Một server trung gian duy nhất mở SSH, các server khác chỉ cho phép SSH từ Bastion. |

## 9. How
```bash
# Lệnh AWS CLI để giới hạn SSH cho một IP cụ thể
aws ec2 authorize-security-group-ingress \
    --group-id sg-12345678 \
    --protocol tcp \
    --port 22 \
    --cidr 1.2.3.4/32
```

## 10. Production concerns
### Scaling
Khi số lượng nhân viên tăng lên, việc quản lý whitelist IP trong Security Group sẽ trở nên quá tải. Nên chuyển sang Session Manager.

### Failure
Nếu người quản trị duy nhất bị đổi IP và mất quyền SSH, họ sẽ cần dùng CloudFormation hoặc một IAM user khác có quyền sửa Security Group để cứu vãn.

### Monitoring
Sử dụng AWS Config Rule `restricted-common-ports` để tự động phát hiện các Security Group mở port 22 công khai.

## 11. Common mistakes
- Mistake: Mở port 22 cho toàn bộ dải IP của công ty nhưng dải đó bao gồm cả khách vãng lai.
  Fix: Chỉ mở cho dải IP VPN hoặc IP tĩnh của bộ phận kỹ thuật.

- Mistake: Quên xóa các rule SSH cũ của nhân viên đã nghỉ việc hoặc dự án đã kết thúc.
  Fix: Thực hiện rà soát Security Group định kỳ (Audit) hàng tháng.

## 12. Sample project
Thiết lập một Security Group chỉ cho phép SSH từ một IP cụ thể. Sau đó cài đặt SSM Agent lên EC2 và thực hiện kết nối mà không cần mở port 22 để so sánh trải nghiệm.

## 13. Interview
### Core Q&A
1. Q: Tại sao Session Manager lại an toàn hơn SSH truyền thống?
   A: Vì Session Manager không yêu cầu mở bất kỳ port inbound nào, không cần quản lý SSH Key, và mọi câu lệnh thực hiện đều được ghi log chi tiết trong CloudWatch/S3 cho mục đích audit.

### Scenario
Bạn được giao quản lý một hệ thống có 100 EC2 instance đang mở SSH công khai. Bạn sẽ làm gì để thắt chặt bảo mật mà không làm gián đoạn công việc của team dev?
Trả lời: 1. Khuyến khích team chuyển sang dùng SSM Session Manager. 2. Với những người vẫn cần SSH, yêu cầu họ cung cấp IP tĩnh và cập nhật SG. 3. Sử dụng script để tìm và thay thế `0.0.0.0/0` bằng các IP cụ thể. 4. Cuối cùng, đóng port 22 ở tất cả các SG và chỉ mở lại khi có yêu cầu (Just-in-time access).

## 14. References
- Official Docs: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/authorizing-access-to-an-instance.html
- GitHub Repo: N/A
- Spec / RFC: N/A
- Changelog: N/A

## 15. Real-world Code
Sử dụng Terraform module `terraform-aws-modules/security-group/aws` để quản lý rules một cách có cấu trúc.

## 16. Community
- Reddit: r/aws - thảo luận về việc "Say goodbye to port 22".
- Stack Overflow: Cách tự động cập nhật Security Group khi IP local thay đổi.
- Blog: Security best practices for EC2.
- Talk: AWS re:Invent - "Goodbye Bastion Hosts, Hello Session Manager".
