---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/security"
related:
  - "[[aws-ec2-security-groups.md]]"
  - "[[aws-ec2-instance.md]]"
  - "[[ssh-tools-ec2-macos.md]]"
---

## 1. What
AWS Inbound Rules là các quy tắc cấu hình trong Security Group (SG) của AWS, xác định loại lưu lượng truy cập (traffic) nào được phép đi vào các tài nguyên AWS (như EC2 instance, RDS database). Mỗi quy tắc bao gồm các thành phần: Protocol, Port Range, và Source (nguồn gốc của traffic).

## 2. Why
Trong môi trường cloud, mọi tài nguyên đều mặc định bị cô lập để đảm bảo an toàn. Nếu không có Inbound Rules, không ai có thể truy cập vào ứng dụng hoặc server của bạn. Inbound Rules đóng vai trò là hàng rào bảo mật đầu tiên, giúp lọc bỏ các truy cập trái phép và chỉ cho phép những traffic cần thiết đi qua.

## 3. Mental Model
Hãy tưởng tượng Inbound Rules giống như một **"Danh sách khách mời" (Guest List)** tại cửa một câu lạc bộ:
- Chỉ những người có tên trong danh sách (khớp với IP, Port, Protocol) mới được bảo vệ (Security Group) cho phép vào cửa.
- Nếu bạn không có tên trong danh sách, bạn bị từ chối mặc định.
- Đặc biệt: Vì Security Group có tính "Stateful", nếu bạn đã được cho vào cửa, bạn sẽ được phép đi ra mà không cần kiểm tra lại danh sách ở cửa ra.

## 4. Where it fits
Vị trí trong kiến trúc mạng AWS:
`Internet -> Internet Gateway -> VPC -> Subnet -> Security Group (Inbound Rules) -> EC2 Instance`

## 5. When to use
- Khi cần mở cổng 80/443 để cho phép người dùng truy cập website.
- Khi cần mở cổng 22 (SSH) hoặc 3389 (RDP) để quản trị server.
- Khi cần cho phép Web Server kết nối tới Database Server (mở cổng 3306/5432).

## 6. When NOT to use
- Khi bạn muốn chặn (Deny) một dải IP cụ thể (Security Group chỉ hỗ trợ "Allow", để chặn IP cụ thể bạn phải dùng Network ACL).
- Khi traffic là "Outbound" (luồng đi ra từ instance) - trường hợp này dùng Outbound Rules.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Stateful: Tự động cho phép traffic phản hồi đi ra. | Chỉ hỗ trợ quy tắc cho phép (Allow), không có "Deny". |
| Có thể dùng Security Group ID làm Source (Source Group Referencing). | Có giới hạn về số lượng quy tắc trên mỗi Security Group. |
| Thay đổi có hiệu lực ngay lập tức. | Khó quản lý nếu cấu hình quá nhiều quy tắc rời rạc cho từng IP. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Network ACL (NACL) | Hoạt động ở tầng Subnet, stateless, hỗ trợ cả Allow và Deny. |
| AWS WAF | Bảo mật ở tầng ứng dụng (Layer 7), chống tấn công SQL Injection, XSS. |

## 9. How
Cấu hình Inbound Rules qua giao diện hoặc Terraform:

**Ví dụ cấu hình cho Web Server (HTTP/HTTPS):**
- Protocol: TCP, Port: 80, Source: 0.0.0.0/0 (Cho phép mọi nơi)
- Protocol: TCP, Port: 443, Source: 0.0.0.0/0

**Ví dụ cấu hình cho Database (chỉ nhận traffic từ Web SG):**
- Protocol: TCP, Port: 5432, Source: sg-xxxxxxxx (ID của Web Security Group)

```hcl
# Ví dụ Terraform cho Security Group Inbound Rule
resource "aws_security_group_rule" "allow_http" {
  type              = "inbound"
  from_port         = 80
  to_port           = 80
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.web_sg.id
}
```

## 10. Production concerns
### Least Privilege
Chỉ mở đúng những cổng cần thiết. Ví dụ: Không mở toàn bộ dải port 0-65535 nếu chỉ dùng port 80.

### Source Group Referencing
Thay vì dùng IP tĩnh của server, hãy dùng ID của Security Group nguồn làm Source. Điều này giúp hệ thống tự động thích ứng khi các server trong group nguồn thay đổi IP (Auto Scaling).

### Security
Tuyệt đối tránh mở cổng 22 (SSH) cho `0.0.0.0/0`. Chỉ nên cho phép IP tĩnh của văn phòng hoặc dải IP của VPN.

## 11. Common mistakes
- Mistake: Mở cổng quản trị (22, 3389) cho toàn bộ internet (`0.0.0.0/0`).
  Fix: Giới hạn Source theo IP cụ thể của người quản trị.

- Mistake: Quên rằng Security Group là stateful.
  Fix: Không cần mở cổng ở Outbound Rules cho traffic phản hồi của một Inbound request.

## 12. Sample project
Thiết lập một kiến trúc 2 lớp (2-tier):
1. Security Group cho Web: Inbound port 80/443 từ Internet.
2. Security Group cho DB: Inbound port 3306 chỉ từ Security Group của Web.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt lớn nhất giữa Inbound Rules của Security Group và Network ACL là gì?
   A: Security Group là stateful (nhớ trạng thái kết nối) và hoạt động ở tầng instance. Network ACL là stateless và hoạt động ở tầng subnet.

2. Q: Nếu tôi thêm một Inbound Rule cho phép port 80, tôi có cần thêm một Outbound Rule cho phép port 80 để server trả lời khách hàng không?
   A: Không cần, vì Security Group có tính stateful, traffic phản hồi sẽ tự động được cho phép.

### Scenario
"Khách hàng không thể kết nối tới Database dù bạn đã mở port 3306 cho IP của họ. Bạn sẽ kiểm tra gì?"
-> Trả lời:
1. Kiểm tra Inbound Rules của Database Security Group.
2. Kiểm tra Outbound Rules của máy khách (nếu có).
3. Kiểm tra Network ACL của cả subnet chứa Database và subnet chứa máy khách.
4. Kiểm tra Route Table xem traffic có đường đi tới Database không.

## 14. References
- AWS Docs: [Security Group Rules](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/security-group-rules.html)
- AWS Whitepaper: [Security Best Practices](https://aws.amazon.com/whitepapers/best-practices-for-security-group-management/)

## 15. Real-world Code
Tìm kiếm các module Terraform `terraform-aws-modules/security-group/aws` trên GitHub để xem cách các chuyên gia tổ chức hàng trăm quy tắc Inbound một cách khoa học.

## 16. Community
- Reddit: r/aws
- Stack Overflow: Tag [amazon-web-services] [security-group]
- AWS Blog: Security section.
