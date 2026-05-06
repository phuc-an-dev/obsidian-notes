---
created: 2026-05-05
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/security"
related:
  - "[[aws-guardduty]]"
---

## 1. What
AWS CloudTrail là một dịch vụ quản trị, tuân thủ và kiểm tra hoạt động vận hành của tài khoản AWS. Nó ghi lại mọi hành động được thực hiện bởi người dùng, vai trò (role) hoặc các dịch vụ AWS khác dưới dạng các sự kiện (events).

## 2. Why
Trong một hệ thống cloud, việc biết "Ai đã làm gì, vào lúc nào, từ đâu" là cực kỳ quan trọng. Nếu không có CloudTrail, bạn sẽ không thể điều tra được ai đã xóa database, ai đã thay đổi Security Group, hoặc tài khoản nào đang bị chiếm quyền điều khiển.

## 3. Mental Model
Hãy tưởng tượng CloudTrail như một hệ thống Camera an ninh ghi hình 24/7 trong một tòa nhà. Mỗi khi có người mở cửa, bật đèn hay di chuyển đồ đạc, camera đều ghi lại thời gian và khuôn mặt của người đó. CloudTrail chính là "hộp đen" của toàn bộ hạ tầng AWS.

## 4. Where it fits
API Call -> AWS CloudTrail -> CloudTrail Logs (S3/CloudWatch Logs).

## 5. When to use
CloudTrail nên được bật mặc định trên tất cả các vùng (Regions) của tài khoản AWS. Đây là yêu cầu bắt buộc cho mọi dự án nghiêm túc để đảm bảo tính minh bạch và khả năng điều tra (Forensics).

## 6. When NOT to use
Không có lý do gì để tắt CloudTrail. Tuy nhiên, cần cân nhắc việc ghi lại "Data events" (ví dụ: các thao tác đọc/ghi file trên S3) vì nó có thể tạo ra khối lượng log khổng lồ và tốn kém chi phí.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Cung cấp lịch sử hoạt động chi tiết cho việc audit. | Chi phí lưu trữ log trên S3 có thể tăng cao nếu không có chính sách lifecycle. |
| Hỗ trợ phát hiện các hành vi bất thường. | Việc phân tích log thủ công rất khó khăn do khối lượng dữ liệu lớn. |
| Tích hợp tốt với CloudWatch Alarms để cảnh báo thời gian thực. | Có độ trễ nhất định (vài phút) từ lúc sự kiện xảy ra đến khi xuất hiện trong log. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| AWS Config | Tập trung vào thay đổi trạng thái tài nguyên (Resource State) hơn là ai gọi API. |
| VPC Flow Logs | Ghi lại lưu lượng mạng, không ghi lại các hành động API. |

## 9. How
```bash
# Kiểm tra trạng thái của CloudTrail trail
aws cloudtrail describe-trails

# Xem các sự kiện gần đây nhất (Management Events)
aws cloudtrail lookup-events --max-items 5
```

## 10. Production concerns
### Scaling
CloudTrail tự động scale để xử lý mọi lượng API call trong account của bạn.

### Failure
Nếu S3 bucket dùng để chứa log bị xóa hoặc bị chặn quyền ghi, CloudTrail sẽ không thể lưu trữ log. Cần thiết lập chính sách bảo vệ bucket này.

### Monitoring
Tích hợp CloudTrail với Amazon CloudWatch Logs để tạo các Metric Filter và Alert cho các hành động nguy hiểm (ví dụ: `DeleteBucket`, `TerminateInstances`).

## 11. Common mistakes
- Mistake: Chỉ bật CloudTrail ở một vùng (Single-region trail).
  Fix: Luôn bật CloudTrail cho tất cả các vùng (Multi-region trail) để bắt được các hành động ở những vùng bạn không thường xuyên sử dụng.

- Mistake: Để log CloudTrail trong cùng một account bị hack.
  Fix: Sử dụng "Log Archive" account riêng biệt và đẩy log về đó để đảm bảo kẻ tấn công không thể xóa dấu vết sau khi xâm nhập.

## 12. Sample project
Thiết lập một CloudTrail trail gửi log về S3 bucket có bật tính năng MFA Delete và Object Lock để ngăn chặn việc xóa log trái phép. Cấu hình CloudWatch Alarm để báo tin nhắn Slack mỗi khi có ai đó dùng Root Account.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa Management Events và Data Events trong CloudTrail là gì?
   A: Management Events ghi lại các thao tác quản trị như tạo/xóa tài nguyên (ví dụ: `RunInstances`). Data Events ghi lại các thao tác dữ liệu bên trong tài nguyên (ví dụ: `GetObject` trong S3, `PutItem` trong DynamoDB). Data events mặc định không được ghi để tiết kiệm chi phí.

### Scenario
Bạn phát hiện một EC2 instance quan trọng bị xóa vào đêm qua nhưng không ai thừa nhận. Bạn sẽ làm gì?
Trả lời: Tôi sẽ vào CloudTrail Console, sử dụng tính năng Lookup events và lọc theo `Event name: TerminateInstances`. Tôi sẽ xác định được `User name`, `Source IP address` và thời gian chính xác của hành động đó để tìm ra nguyên nhân và người thực hiện.

## 14. References
- Official Docs: https://docs.aws.amazon.com/awscloudtrail/latest/userguide/cloudtrail-user-guide.html
- GitHub Repo: N/A
- Spec / RFC: N/A
- Changelog: N/A

## 15. Real-world Code
Kiểm soát tính toàn vẹn của log bằng cách bật `EnableLogFileValidation: true` trong Terraform hoặc CloudFormation.

## 16. Community
- Reddit: r/aws - các case study dùng CloudTrail để điều tra xâm nhập.
- Stack Overflow: Cách truy vấn CloudTrail log bằng Amazon Athena.
- Blog: AWS Security Blog về log analysis.
- Talk: AWS re:Invent - "Deep Dive into AWS CloudTrail".
