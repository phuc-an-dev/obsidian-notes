---
created: 2026-05-05
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/compute"
related:
  - "[[aws-ec2-instance]]"
  - "[[aws-ec2-key-pairs]]"
---

## 1. What
AWS Security Group (SG) là một firewall ảo hoạt động ở cấp độ Instance (thực tế là ở cấp độ Network Interface - ENI). Nó kiểm soát các luồng dữ liệu đi vào (Inbound) và đi ra (Outbound) của một hoặc nhiều EC2 instances dựa trên các quy tắc (Rules) về Port và IP Protocol.

## 2. Why
Để đảm bảo an toàn, các server không nên mở toàn bộ các cổng kết nối ra internet. Security Group cung cấp một lớp bảo mật "Allow-only" (chỉ cho phép những gì được định nghĩa), giúp ngăn chặn các truy cập trái phép và giảm thiểu bề mặt tấn công (Attack Surface) cho hệ thống.

## 3. Mental Model
Hãy coi Security Group như một nhân viên bảo vệ đứng ngay trước cửa phòng của bạn (EC2 Instance). Nhân viên này chỉ có một "Danh sách trắng" (Allow-list). Nếu tên bạn có trong danh sách, bạn được vào. Đặc biệt, nhân viên này có trí nhớ rất tốt (Stateful): nếu ông ấy đã cho bạn vào cửa, ông ấy sẽ mặc định cho bạn ra mà không cần kiểm tra lại danh sách. Ngược lại, nếu ông ấy cho phép bạn ra ngoài mua đồ, ông ấy cũng sẽ tự động mở cửa cho bạn quay lại phòng.

## 4. Where it fits
Internet -> Internet Gateway -> [NACL] -> [Security Group] -> EC2 Instance.

## 5. When to use
- Mọi EC2 instance khi khởi tạo đều phải được gán vào ít nhất một Security Group.
- Mở các port dịch vụ cụ thể: Port 80/443 cho Web Server, Port 22 cho SSH, Port 3306 cho MySQL.
- Giới hạn truy cập giữa các tầng kiến trúc (ví dụ: chỉ cho phép Web Server truy cập vào Database Server).

## 6. When NOT to use
- Khi cần chặn một địa chỉ IP cụ thể (Deny rule) - SG không có tính năng Deny, bạn phải dùng Network ACL (NACL).
- Khi cần bảo mật ở tầng ứng dụng sâu hơn (Layer 7) như chống tấn công SQL Injection, Cross-site Scripting (nên dùng AWS WAF).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Stateful: Tự động cho phép traffic phản hồi (Return traffic). | Giới hạn số lượng quy tắc trên mỗi Security Group (mặc định 60 rules). |
| Có thể tham chiếu Security Group khác làm Source (SG-referencing). | Không hỗ trợ quy tắc "Chặn" (Explicit Deny). |
| Thay đổi rule có hiệu lực ngay lập tức. | Hoạt động ở Layer 4, không hiểu được nội dung gói tin ở Layer 7. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Network ACL (NACL) | Cấp độ Subnet, Stateless, hỗ trợ cả Allow và Deny rules. |
| AWS WAF | Firewall tầng ứng dụng (L7), bảo vệ chống lại các lỗ hổng web. |
| AWS Network Firewall | Dịch vụ firewall có quản lý, kiểm soát toàn bộ VPC traffic chuyên sâu. |

## 9. How
Tạo Security Group và thêm rule cho phép SSH từ IP của bạn:
```bash
# Tạo SG
aws ec2 create-security-group \
    --group-name MyWebSG \
    --description "Security group for web server" \
    --vpc-id vpc-1a2b3c4d

# Thêm rule Inbound cho SSH (Port 22)
aws ec2 authorize-security-group-ingress \
    --group-id sg-0123456789abcdef0 \
    --protocol tcp \
    --port 22 \
    --cidr 203.0.113.0/32
```

## 10. Production concerns
### Scaling
Sử dụng "Security Group Referencing" thay vì điền IP tĩnh. Ví dụ: Database SG chỉ cần để Source là Web SG ID. Khi Web instances tăng lên hoặc thay đổi IP, Database SG vẫn tự động nhận diện và cho phép.

### Failure
Cấu hình sai Security Group là nguyên nhân hàng đầu dẫn đến lỗi "Connection Timeout". Luôn kiểm tra kỹ cả Inbound và Outbound rules.

### Monitoring
Sử dụng VPC Flow Logs để theo dõi các gói tin bị REJECT bởi Security Group, giúp phát hiện các hành vi dò tìm port trái phép.

## 11. Common mistakes
- Mistake: Mở port 22 (SSH) hoặc 3306 (DB) cho toàn bộ internet (0.0.0.0/0).
  Fix: Chỉ mở cho dãy IP cụ thể của công ty hoặc dùng Bastion Host/VPN.

- Mistake: Quên rằng SG là Stateful nên cấu hình cả Inbound và Outbound cho cùng một luồng traffic.
  Fix: Chỉ cần cấu hình chiều khởi tạo (Initiator), chiều phản hồi sẽ tự động được cho phép.

## 12. Sample project
Thiết lập "Security Group Chaining":
1. `Web-SG`: Inbound 80/443 từ 0.0.0.0/0.
2. `App-SG`: Inbound 8080 từ `Web-SG`.
3. `DB-SG`: Inbound 5432 từ `App-SG`.
Cách làm này đảm bảo Database không bao giờ có thể bị truy cập trực tiếp từ Internet, ngay cả khi hacker biết IP của DB.

## 13. Interview
### Core Q&A
1. Q: "Stateful" trong Security Group có nghĩa là gì?
   A: Nghĩa là nếu một request đi vào (Inbound) được cho phép, thì traffic phản hồi tương ứng (Outbound) sẽ tự động được cho phép đi ra, bất kể quy tắc Outbound là gì, và ngược lại.
2. Q: Sự khác biệt chính giữa Security Group và Network ACL là gì?
   A: SG ở cấp độ Instance, Stateful, chỉ có Allow rules. NACL ở cấp độ Subnet, Stateless, có cả Allow và Deny rules, được xử lý trước SG khi traffic đi vào.

### Scenario
Bạn có một ứng dụng web chạy trên EC2 không thể kết nối tới RDS Database mặc dù bạn đã gán đúng Username/Password. Bạn sẽ kiểm tra Security Group như thế nào?
Trả lời: Tôi sẽ kiểm tra 2 nơi: 1. Inbound rule của Database SG phải cho phép port tương ứng (vd 3306) với Source là Security Group của Web EC2. 2. Outbound rule của Web SG phải cho phép gửi traffic đến Port và IP/SG của Database (mặc định Outbound là Allow All nhưng cần check lại nếu đã bị sửa).

## 14. References
- Official Docs: https://docs.aws.amazon.com/vpc/latest/userguide/VPC_SecurityGroups.html
- GitHub Repo: N/A
- Spec / RFC: N/A
- Changelog: N/A

## 15. Real-world Code
- Terraform SG example: https://registry.terraform.io/modules/terraform-aws-modules/security-group/aws/latest

## 16. Community
- Reddit: r/aws - Security Group best practices.
- Stack Overflow: "Difference between Security Group and NACL".
- Blog: AWS Security Blog - "Control traffic to your AWS resources using security groups".
