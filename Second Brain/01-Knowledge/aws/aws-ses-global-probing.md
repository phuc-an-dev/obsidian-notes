---
created: 2026-05-07
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/security"
related:
  - "[[aws-guardduty.md]]"
  - "[[aws-iam-hardening.md]]"
  - "[[aws-deny-explicit-dangerous-services.md]]"
---

## 1. What
Global Probing (Quét dịch vụ trên toàn cầu) đối với Amazon SES (Simple Email Service) là hành vi của kẻ tấn công hoặc các công cụ tự động thực hiện việc thăm dò, quét các tài khoản AWS để tìm kiếm các SES identities (email hoặc domain đã được xác thực) có cấu hình lỏng lẻo hoặc các IAM credentials có quyền sử dụng SES nhằm mục đích phát tán thư rác (spam) hoặc lừa đảo (phishing).

## 2. Why
SES là một dịch vụ cực kỳ "có giá trị" đối với tội phạm mạng vì:
- Tỷ lệ vào inbox cao nhờ danh tiếng (reputation) của hạ tầng AWS.
- Chi phí gửi cực rẻ.
- Nếu chiếm được một danh tính đã xác thực, kẻ tấn công có thể mượn danh thương hiệu uy tín để lừa đảo quy mô lớn.
AWS thường xuyên quét và giám sát các hành vi probing này để bảo vệ danh tiếng cho toàn bộ dải IP của họ.

## 3. Mental Model
Hãy tưởng tượng AWS SES giống như một **"Tổng đài bưu điện uy tín"**.
- Kẻ xấu không có thẻ hội viên (Danh tính xác thực) sẽ đi quanh tòa nhà, thử vặn mọi tay nắm cửa (Probing) để xem có ai quên khóa cửa hoặc có ai đánh rơi thẻ hội viên (Access Keys) không.
- Một khi chúng lọt vào được và dùng danh nghĩa của bạn để gửi hàng ngàn lá thư rác, bưu điện sẽ bị mang tiếng, và thậm chí cả tòa nhà có thể bị cảnh sát (Các tổ chức chống spam) phong tỏa.

## 4. Where it fits
Attacker -> **AWS API (ListIdentities / SendEmail)** -> CloudTrail Logs -> **Amazon GuardDuty (Detection)** -> Security Alert.

## 5. When to use
N/A (Đây là hành vi cần phòng chống). Tuy nhiên, bạn cần hiểu về nó khi cấu hình giám sát bảo mật cho tài khoản AWS.

## 6. When NOT to use
N/A.

## 7. Trade-offs
| Pros (Của việc giám sát) | Cons (Của việc giám sát) |
|------|------|
| Phát hiện sớm nguy cơ bị hack tài khoản. | Có thể gây ra False Positive nếu team DevOps thực hiện audit định kỳ. |
| Bảo vệ danh tiếng gửi mail của domain. | Đòi hỏi chi phí cho các dịch vụ giám sát như GuardDuty. |
| Ngăn chặn việc phát sinh chi phí gửi mail đột biến. | N/A |

## 8. Alternatives
- **DMARC/SPF/DKIM**: Các bản ghi DNS giúp xác thực mail chính chủ, hỗ trợ các bưu điện khác nhận diện mail giả mạo dù probing có thành công.
- **AWS SES Sandbox**: Mặc định SES nằm trong Sandbox, chỉ cho phép gửi mail tới các địa chỉ đã xác thực, hạn chế thiệt hại nếu bị probing.

## 9. How
Làm thế nào để phát hiện Probing qua Amazon GuardDuty:
GuardDuty sẽ báo động nếu thấy các hành vi bất thường như:
- `Discovery:IAMUser/AnomalousBehavior.SES`: User thực hiện hàng loạt lệnh liệt kê identities ở các Region mà họ chưa từng sử dụng.
- `UnauthorizedAccess:IAMUser/ConsoleLoginSuccess.UnexpectedMeasurement`: Đăng nhập Console từ IP lạ và truy cập ngay vào SES.

Cách cấu hình IAM Policy an toàn để chống lạm dụng SES:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "ses:SendEmail",
            "Resource": "arn:aws:ses:ap-southeast-1:123456789012:identity/marketing@example.com",
            "Condition": {
                "StringEquals": {
                    "ses:FromAddress": "marketing@example.com"
                }
            }
        }
    ]
}
```
*Lưu ý: Luôn chỉ định cụ thể Resource ARN thay vì dùng `*`.*

## 10. Production concerns
### Reputation Management
Nếu bị probing thành công và phát tán spam, điểm uy tín (Reputation Score) của bạn sẽ giảm. Nếu giảm xuống dưới một mức nhất định, AWS sẽ tự động tạm dừng (suspend) quyền gửi mail của bạn.

### Global Presence
Kẻ tấn công thường quét SES trên **tất cả 20+ Regions** của AWS vì họ hy vọng bạn quên cấu hình bảo mật ở những Region xa xôi.

## 11. Common mistakes
- Mistake: Xác thực domain trong SES nhưng gán quyền `ses:*` cho tất cả mọi người trong team.
  Fix: Chỉ cấp quyền `ses:SendEmail` cho các ứng dụng cần thiết và giới hạn địa chỉ gửi.
- Mistake: Để Access Keys của SES trong mã nguồn Frontend.
  Fix: Luôn gửi mail thông qua Backend và sử dụng IAM Roles.

## 12. Sample project
Thiết lập hệ thống phản ứng tự động:
1. GuardDuty phát hiện hành vi SES Probing.
2. EventBridge bắt sự kiện và kích hoạt Lambda.
3. Lambda tự động thu hồi (Quarantine) IAM User/Role đang bị nghi ngờ và gửi tin nhắn khẩn cấp qua Telegram/Slack.

## 13. Interview
### Core Q&A
1. Q: Tại sao kẻ tấn công lại quét SES ở những Region mà tôi không dùng?
   A: Vì chúng hy vọng bạn không đặt giám sát hoặc báo động ở đó, giúp chúng có thêm thời gian phát tán spam trước khi bị phát hiện.

2. Q: Làm thế nào để ngăn chặn việc gửi mail hàng loạt ngay cả khi Access Key bị lộ?
   A: Sử dụng **Sending Quotas** và **Sending Rates** trong SES để giới hạn số lượng mail tối đa có thể gửi trong một khoảng thời gian.

### Scenario
"Hệ thống báo động của bạn phát hiện một loạt lệnh `ListIdentities` được thực thi từ một IP ở Nga, trong khi team bạn ở Việt Nam. Bạn làm gì?"
-> Trả lời: Đây là dấu hiệu điển hình của Global Probing. Tôi sẽ ngay lập tức: 1. Vô hiệu hóa Access Key của User liên quan. 2. Kiểm tra `SendEmail` logs trong CloudTrail xem đã có mail nào bị phát tán chưa. 3. Đổi password và bật MFA cho User đó nếu họ dùng Console.

## 14. References
- AWS Security Blog: [Detecting and Mitigating SES Abuse](https://aws.amazon.com/blogs/messaging/monitoring-your-usage-and-maintaining-your-sending-reputation/)
- GuardDuty Documentation: [SES Finding Types](https://docs.aws.amazon.com/guardduty/latest/userguide/guardduty_finding-types-iam.html)

## 15. Real-world Code
Nghiên cứu các bộ lọc spam của AWS SES và cách cấu hình `Configuration Sets` để theo dõi sự kiện gửi mail.

## 16. Community
- Reddit: r/aws - các bài đăng về việc tài khoản bị "treo" SES do spam.
- Stack Overflow: Cách cấu hình `ses:FromAddress` trong IAM.
