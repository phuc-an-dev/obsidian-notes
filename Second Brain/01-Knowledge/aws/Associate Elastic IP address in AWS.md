---
created: 2026-05-07
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/networking"
related:
  - "[[ec2]]"
  - "[[aws-ec2-instance]]"
  - "[[aws-ec2-network-interfaces]]"
---

## 1. What
Elastic IP (EIP) là một địa chỉ IP công cộng tĩnh (static IPv4 address) được thiết kế cho điện toán đám mây động. Khi bạn gán (associate) một EIP cho một EC2 instance hoặc một Network Interface (ENI), địa chỉ này sẽ không thay đổi ngay cả khi instance bị dừng (stop) và khởi động lại (start).

## 2. Why
Mặc định, khi bạn khởi động một EC2 instance, AWS cấp cho nó một địa chỉ Public IP tạm thời. Tuy nhiên, địa chỉ này sẽ bị AWS thu hồi và thay đổi mỗi khi bạn Stop/Start instance. Điều này gây khó khăn nếu bạn cần:
- Trỏ bản ghi DNS (như A record) cố định về server.
- Cấu hình whitelist IP trên các dịch vụ bên thứ ba (như cổng thanh toán, API đối tác).
- Duy trì kết nối ổn định cho các ứng dụng client không hỗ trợ cập nhật IP linh hoạt.

## 3. Mental Model
Hãy tưởng tượng địa chỉ IP giống như **số điện thoại**:
- **Public IP mặc định** giống như một chiếc SIM rác: Mỗi khi bạn tắt máy và bật lại, nhà mạng lại cấp cho bạn một số mới. Bạn không thể in số này lên danh thiếp vì nó luôn thay đổi.
- **Elastic IP** giống như một **số điện thoại cố định (hotline)**: Bạn mua số này (Allocate) và sở hữu nó. Bạn có thể cắm nó vào bất kỳ chiếc điện thoại nào (Associate). Dù bạn đổi điện thoại mới, khách hàng vẫn gọi cho bạn qua số hotline đó.

## 4. Where it fits
Internet -> **Elastic IP** -> Internet Gateway -> **EC2 Instance / ENI**.

## 5. When to use
- Khi host một web server hoặc ứng dụng cần truy cập trực tiếp từ Internet qua một IP cố định.
- Khi cần khả năng chuyển đổi nhanh (failover) giữa các instance: Bạn có thể nhanh chóng gán EIP từ instance bị lỗi sang một instance dự phòng.
- Khi làm việc với các hệ thống yêu cầu bảo mật dựa trên IP (IP-based filtering).

## 6. When NOT to use
- Đối với các instance nằm trong Private Subnet (không có kết nối trực tiếp ra Internet).
- Khi bạn sử dụng Load Balancer (ALB/NLB): Người dùng nên truy cập qua DNS của Load Balancer thay vì IP trực tiếp của EC2.
- Khi bạn cần hàng ngàn IP tĩnh (EIP có giới hạn mặc định là 5 địa chỉ trên mỗi Region cho mỗi tài khoản).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| IP cố định, không đổi khi Stop/Start. | Chi phí: AWS tính phí nếu EIP không được gán cho instance đang chạy. |
| Dễ dàng failover giữa các instance. | Giới hạn số lượng (Quotas) trên mỗi tài khoản. |
| Hỗ trợ Reverse DNS (với yêu cầu gửi cho AWS). | Chỉ hỗ trợ IPv4 (không có EIP cho IPv6, thay vào đó dùng GUA). |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Application Load Balancer (ALB) | Cung cấp DNS cố định, hỗ trợ scale nhiều instance phía sau, an toàn hơn. |
| Global Accelerator | Cung cấp IP tĩnh toàn cầu, tối ưu hóa đường truyền qua mạng AWS. |
| Dynamic DNS (DDNS) | Script tự động cập nhật bản ghi DNS khi IP thay đổi, nhưng có độ trễ TTL. |

## 9. How
Quy trình gồm 2 bước:
1. **Allocate**: Yêu cầu AWS cấp cho bạn một IP từ pool của họ.
2. **Associate**: Gắn IP đó vào một EC2 Instance ID hoặc Network Interface ID cụ thể.

Lệnh AWS CLI:
```bash
# 1. Cấp phát EIP
aws ec2 allocate-address --domain vpc

# 2. Gán EIP cho instance (sử dụng AllocationId từ bước 1)
aws ec2 associate-address \
    --instance-id i-0123456789abcdef0 \
    --allocation-id eipalloc-0123456789abcdef0

# 3. Gỡ gán (Disassociate)
aws ec2 disassociate-address --association-id eipassoc-0123456789abcdef0

# 4. Giải phóng (Release) - Xóa hẳn để không bị tính phí
aws ec2 release-address --allocation-id eipalloc-0123456789abcdef0
```

## 10. Production concerns
### Cost Optimization
AWS tính phí cho các EIP được cấp phát nhưng **không sử dụng** (không gán vào instance đang chạy). Điều này để tránh lãng phí tài nguyên IP của AWS. Luôn **Release** EIP nếu bạn không còn cần đến nó.

### Limits
Nếu bạn cần nhiều hơn 5 EIP, bạn phải gửi yêu cầu "Service Limit Increase" cho AWS.

### Region Specific
EIP được cấp phát cho một Region cụ thể và không thể di chuyển sang Region khác.

## 11. Common mistakes
- Mistake: Quên release EIP sau khi xóa EC2 instance, dẫn đến hóa đơn hàng tháng tăng thêm vài đô la vô ích.
  Fix: Thiết lập quy trình dọn dẹp (Cleanup) hoặc dùng AWS Config để phát hiện EIP mồ côi.

- Mistake: Gán EIP trực tiếp vào EC2 trong Public Subnet nhưng Security Group không mở đúng port.
  Fix: Luôn kiểm tra Security Group và Network ACL khi gặp lỗi Connection Timeout.

## 12. Sample project
Thiết lập một "Hệ thống phục hồi nhanh":
1. Chạy 2 instance Web-A và Web-B.
2. Gán EIP cho Web-A.
3. Viết một script giám sát (Health check): Nếu Web-A không phản hồi, script tự động thực hiện lệnh `associate-address` để chuyển EIP sang Web-B.

## 13. Interview
### Core Q&A
1. Q: Chi phí của Elastic IP được tính như thế nào?
   A: Miễn phí nếu nó được gắn vào một instance đang chạy (running) và instance đó chỉ có duy nhất 1 EIP. Bạn bị tính phí nếu: EIP không được gắn vào đâu, gắn vào instance đang bị stop, hoặc gắn vào một network interface không sử dụng.

2. Q: Có thể gán EIP cho một instance trong Private Subnet không?
   A: Có thể về mặt thao tác kỹ thuật, nhưng instance đó vẫn không thể ra Internet được vì thiếu Internet Gateway và Route phù hợp. EIP cần một "đường dẫn" ra ngoài.

### Scenario
"Khách hàng yêu cầu website của họ phải có IP tĩnh để đối tác cấu hình Firewall. Website hiện đang chạy 1 EC2 duy nhất. Bạn sẽ làm gì?"
-> Trả lời: Tôi sẽ Allocate một Elastic IP và Associate nó vào EC2 đó. Sau đó cung cấp IP này cho đối tác. Tôi cũng sẽ dặn khách hàng tuyệt đối không được Release IP này nếu không sẽ mất liên kết với đối tác.

## 14. References
- AWS Docs: [Elastic IP addresses](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html)
- Pricing: [Amazon EC2 Pricing - IP Address section](https://aws.amazon.com/ec2/pricing/)

## 15. Real-world Code
Sử dụng Terraform để tạo và gán EIP:
```hcl
resource "aws_eip" "lb" {
  instance = aws_instance.web.id
  domain   = "vpc"
}
```

## 16. Community
- YouTube: "AWS Elastic IP Tutorial" - AWS Training.
- Stack Overflow: [amazon-elastic-ip] tag.
- Reddit: Thảo luận về việc AWS tăng phí EIP từ tháng 2/2024.
