---
created: 2026-05-05
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/security"
related:
  - "[[aws-cloudtrail]]"
---

## 1. What
AWS GuardDuty là một dịch vụ phát hiện đe dọa (Threat Detection) thông minh, hoạt động liên tục để bảo vệ tài khoản AWS, khối lượng công việc và dữ liệu trong S3. Nó sử dụng Machine Learning, phát hiện bất thường và tích hợp Threat Intelligence để nhận diện các hành vi nguy hiểm.

## 2. Why
Các hệ thống bảo mật dựa trên quy tắc (Rule-based) truyền thống thường không theo kịp các kỹ thuật tấn công hiện đại. GuardDuty ra đời để tự động hóa việc phân tích hàng tỷ sự kiện từ nhiều nguồn log khác nhau mà không làm ảnh hưởng đến hiệu năng hệ thống.

## 3. Mental Model
Hãy coi GuardDuty như một thám tử tư thông minh và im lặng. Thám tử này không đứng gác ở cửa mà liên tục đọc các báo cáo (logs) và quan sát hành vi của mọi người trong tòa nhà. Nếu thấy ai đó đi lại đáng ngờ hoặc cố tình thử chìa khóa lạ, thám tử sẽ lập tức báo động.

## 4. Where it fits
Logs Sources (VPC Flow Logs, CloudTrail, DNS Logs) -> GuardDuty Analysis -> Findings (Alerts).

## 5. When to use
GuardDuty nên được kích hoạt cho mọi tài khoản AWS Production. Nó đặc biệt hữu ích cho các hệ thống có nhiều instance công khai hoặc lưu trữ dữ liệu nhạy cảm trên S3.

## 6. When NOT to use
Đối với các tài khoản học tập cá nhân với ngân sách hạn hẹp, GuardDuty có thể tốn một khoản phí nhỏ sau thời gian dùng thử 30 ngày. Tuy nhiên, rủi ro bị hack đào coin tốn nhiều tiền hơn rất nhiều so với phí GuardDuty.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Kích hoạt chỉ bằng một cú click, không cần cài đặt agent. | Chi phí dựa trên khối lượng dữ liệu log được phân tích. |
| Không làm giảm hiệu năng hệ thống (Zero performance impact). | Có thể có kết quả dương tính giả (False Positives) cần lọc thủ công. |
| Tự động cập nhật các mẫu tấn công mới nhất từ AWS Security. | Chỉ phát hiện, không tự động ngăn chặn (cần kết hợp với Lambda). |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Amazon Macie | Tập trung sâu vào việc phát hiện dữ liệu nhạy cảm (PII) trong S3. |
| IDS/IPS (Third-party) | Thường yêu cầu cài đặt agent và cấu hình mạng phức tạp hơn. |

## 9. How
```bash
# Kích hoạt GuardDuty cho một vùng cụ thể
aws guardduty create-detector --enable
```

## 10. Production concerns
### Scaling
GuardDuty tự động scale theo lượng log phát sinh trong account của bạn, xử lý hàng triệu sự kiện mỗi giây.

### Failure
GuardDuty là dịch vụ regional. Nếu một region bị down, việc bảo vệ ở region đó có thể bị gián đoạn. Nên cấu hình gửi Findings về một region trung tâm.

### Monitoring
Tích hợp GuardDuty Findings với Amazon EventBridge để gửi thông báo tự động tới Slack, PagerDuty hoặc gọi Lambda để tự động cách ly các instance bị nhiễm mã độc.

## 11. Common mistakes
- Mistake: Bật GuardDuty nhưng không bao giờ kiểm tra "Findings".
  Fix: Thiết lập alert tự động qua SNS/Email cho các Findings có độ ưu tiên cao (High Severity).

- Mistake: Nghĩ rằng GuardDuty là tường lửa.
  Fix: GuardDuty là hệ thống phát hiện (Detection), không phải ngăn chặn (Prevention). Bạn vẫn cần WAF và Security Groups.

## 12. Sample project
Thiết lập quy trình tự động: GuardDuty phát hiện một EC2 instance đang giao tiếp với một địa chỉ IP thuộc mạng botnet -> EventBridge kích hoạt Lambda -> Lambda gán một Security Group "Quarantine" cho instance đó để chặn mọi lưu lượng ra/vào.

## 13. Interview
### Core Q&A
1. Q: GuardDuty lấy dữ liệu từ đâu để phân tích?
   A: GuardDuty phân tích 3 nguồn log chính: VPC Flow Logs, AWS CloudTrail Management Events, và DNS Query Logs. Ngoài ra nó còn có thể bảo vệ S3 Data Events và EKS Audit Logs.

### Scenario
GuardDuty báo cáo một Finding loại "UnauthorizedAccess:IAMUser/ConsoleLoginFromUnknownIP". Bạn sẽ xử lý thế nào?
Trả lời: Đây là dấu hiệu tài khoản IAM bị lộ mật khẩu. Tôi sẽ ngay lập tức: 1. Vô hiệu hóa password của user đó. 2. Thu hồi các session đang hoạt động. 3. Kiểm tra CloudTrail để xem user đó đã làm gì từ IP lạ kia. 4. Yêu cầu user đổi password và bật MFA nếu chưa có.

## 14. References
- Official Docs: https://docs.aws.amazon.com/guardduty/latest/ug/what-is-guardduty.html
- GitHub Repo: N/A
- Spec / RFC: N/A
- Changelog: N/A

## 15. Real-world Code
Sử dụng AWS Security Hub để tổng hợp kết quả từ GuardDuty cùng với các dịch vụ bảo mật khác của AWS.

## 16. Community
- Reddit: r/aws - thảo luận về các mẫu Finding phổ biến.
- Stack Overflow: Cách tắt bớt các Finding không cần thiết để giảm nhiễu.
- Blog: AWS Security Blog về Threat Detection.
- Talk: AWS re:Invent sessions về GuardDuty Deep Dive.
