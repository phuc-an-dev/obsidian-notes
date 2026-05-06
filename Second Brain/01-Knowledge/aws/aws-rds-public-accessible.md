---
created: 2026-05-05
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/security"
related:
  - "[[aws-check-strange-resources]]"
---

## 1. What
RDS Publicly Accessible là một thiết lập trong Amazon RDS cho phép các database instance có địa chỉ IP công cộng và có thể được truy cập trực tiếp từ internet. Khi bật tính năng này, bất kỳ ai có endpoint và thông tin đăng nhập đều có thể thử kết nối tới database.

## 2. Why
Tính năng này được thiết kế để hỗ trợ việc phát triển và thử nghiệm nhanh chóng, cho phép lập trình viên kết nối từ máy local mà không cần thiết lập VPN hay SSH Tunnel. Tuy nhiên, nó thường bị lạm dụng hoặc quên tắt khi đưa lên production, dẫn đến rủi ro bảo mật cực lớn.

## 3. Mental Model
Hãy tưởng tượng database của bạn là một két sắt chứa đầy vàng. Việc để RDS Publicly Accessible giống như việc bạn đặt cái két sắt đó ngay ngoài vỉa hè thay vì để sâu trong hầm ngầm của ngân hàng. Dù có khóa (password), nhưng mọi kẻ trộm đều có thể tiếp cận và tìm cách phá khóa 24/7.

## 4. Where it fits
Internet -> IGW (Internet Gateway) -> RDS Instance (Public IP).
Mô hình an toàn hơn: Internet -> Bastion Host/VPN -> RDS Instance (Private IP).

## 5. When to use
Chỉ nên dùng trong môi trường Sandbox hoặc Development cực ngắn hạn để test kết nối nhanh từ bên ngoài mà không có hạ tầng mạng phức tạp. Tuyệt đối không dùng cho dữ liệu nhạy cảm.

## 6. When NOT to use
Không dùng cho môi trường Production, Staging hoặc bất kỳ database nào chứa thông tin người dùng, bí mật kinh doanh. Việc để database public là vi phạm các tiêu chuẩn tuân thủ bảo mật như SOC2 hoặc PCI-DSS.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Dễ dàng kết nối từ máy local mà không cần cấu hình phức tạp. | Nguy cơ bị tấn công Brute-force mật khẩu liên tục. |
| Tiết kiệm chi phí so với việc dựng VPN Gateway hoặc NAT Gateway. | Dễ bị lộ thông tin qua các lỗ hổng zero-day của engine database. |
| Tiện lợi cho việc demo nhanh. | Tăng bề mặt tấn công (Attack Surface) cho hạ tầng AWS. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| SSH Tunneling (Bastion Host) | Bảo mật hơn, chỉ cho phép truy cập qua một server trung gian được bảo vệ chặt chẽ. |
| AWS Client VPN | Kết nối an toàn như đang trong mạng nội bộ, quản lý được user truy cập. |
| Site-to-Site VPN | Kết nối trực tiếp văn phòng công ty với VPC AWS. |

## 9. How
```bash
# Sử dụng AWS CLI để sửa đổi RDS instance thành không công khai
aws rds modify-db-instance \
    --db-instance-identifier my-db-instance \
    --no-publicly-accessible \
    --apply-immediately
```

## 10. Production concerns
### Scaling
Việc tắt Public Access không ảnh hưởng đến khả năng scaling của RDS nhưng yêu cầu các ứng dụng client phải nằm trong VPC hoặc có kết nối VPN.

### Failure
Nếu quên cấu hình Security Group phù hợp sau khi tắt Public Access, ứng dụng có thể mất kết nối tới database.

### Monitoring
Sử dụng AWS Config để tự động phát hiện và ngăn chặn các RDS instance được tạo với chế độ Publicly Accessible.

## 11. Common mistakes
- Mistake: Chỉ dựa vào mật khẩu mạnh mà vẫn để database public.
  Fix: Luôn tắt Publicly Accessible và dùng Security Group để giới hạn IP truy cập.

- Mistake: Nghĩ rằng thay đổi port mặc định (ví dụ 3306 sang 3307) là đủ an toàn.
  Fix: Kẻ tấn công dùng port scanner sẽ tìm ra port mới trong vài giây. Cách duy nhất là chặn truy cập từ internet.

## 12. Sample project
Thiết lập một Terraform module để khởi tạo RDS luôn có thuộc tính `publicly_accessible = false` và đi kèm với một cụm EC2 Bastion Host để quản trị.

## 13. Interview
### Core Q&A
1. Q: Tại sao việc để RDS Publicly Accessible lại nguy hiểm dù đã có mật khẩu?
   A: Vì nó mở cửa cho các cuộc tấn công Brute-force, từ chối dịch vụ (DoS) và khai thác lỗ hổng trực tiếp vào database engine mà không cần qua lớp bảo vệ ứng dụng. Ngoài ra, lỗi cấu hình ở mức database có thể dẫn đến rò rỉ toàn bộ dữ liệu.

### Scenario
Một khách hàng báo cáo database của họ bị ransomeware mã hóa dữ liệu và yêu cầu tiền chuộc. Qua kiểm tra, bạn thấy RDS đang để Publicly Accessible. Bạn sẽ xử lý thế nào để ngăn chặn tái diễn?
Trả lời: Đầu tiên là snapshot database để cứu vãn, sau đó ngay lập tức tắt Publicly Accessible, đổi toàn bộ mật khẩu, và thiết lập SSH Tunnel qua Bastion Host để truy cập quản trị. Đồng thời rà soát lại CloudTrail để tìm dấu vết truy cập trái phép.

## 14. References
- Official Docs: https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/UsingWithRDS.ConfiguringConnectivity.html
- GitHub Repo: N/A
- Spec / RFC: N/A
- Changelog: N/A

## 15. Real-world Code
Kiểm tra cấu hình trong CloudFormation hoặc Terraform để đảm bảo database nằm trong Subnet riêng tư (Private Subnet).

## 16. Community
- Reddit: r/aws - thảo luận về các vụ hack RDS do để public.
- Stack Overflow: Cách kết nối RDS từ local khi tắt public access.
- Blog: AWS Security Blog về database protection.
- Talk: AWS re:Invent sessions về VPC Security.
