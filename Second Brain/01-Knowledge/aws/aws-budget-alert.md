---
created: 2026-05-05
tags:
  - "#type/tutorial"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/performance"
related:
  - "[[aws-check-strange-resources]]"
---

## 1. What
AWS Budget Alert là một tính năng trong AWS Budgets cho phép người dùng thiết lập các ngưỡng chi phí hoặc mức sử dụng tài nguyên. Khi chi phí thực tế hoặc chi phí dự báo vượt quá ngưỡng này, AWS sẽ gửi thông báo qua Email hoặc SNS.

## 2. Why
Rất nhiều người dùng AWS đã phải đối mặt với hóa đơn hàng nghìn USD (Bill Shock) do vô tình bật các dịch vụ đắt tiền hoặc bị hack tài khoản để đào coin. AWS Budget Alert ra đời để giải quyết vấn đề này bằng cách cảnh báo sớm trước khi số tiền nợ trở nên quá lớn.

## 3. Mental Model
Hãy coi AWS Budget Alert giống như một chiếc ví thông minh. Bạn cài đặt rằng nếu tiêu quá 50 USD một tháng, chiếc ví sẽ rung lên và gửi tin nhắn cho bạn. Nó không ngăn bạn tiêu tiền, nhưng nó đảm bảo bạn luôn biết mình đang tiêu bao nhiêu.

## 4. Where it fits
Billing Dashboard -> AWS Budgets -> Create Budget -> Set Threshold -> Alert (Email/SNS).

## 5. When to use
Mọi tài khoản AWS (kể cả tài khoản Free Tier) đều nên có ít nhất một Budget Alert. Đây là lớp bảo vệ đầu tiên chống lại việc chi tiêu quá đà.

## 6. When NOT to use
Không có trường hợp nào không nên dùng Budget Alert. Tuy nhiên, nếu bạn có hàng nghìn account, việc quản lý budget thủ công sẽ không hiệu quả bằng việc dùng AWS Organizations với các chính sách tổng thể.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Miễn phí cho 2 budget đầu tiên. | Có thể gây nhiễu nếu thiết lập ngưỡng quá thấp hoặc không thực tế. |
| Cung cấp dự báo (Forecasting) để cảnh báo sớm. | Không tự động dừng tài nguyên (trừ khi kết hợp với Lambda). |
| Dễ dàng thiết lập qua Console hoặc CLI. | Độ trễ cập nhật dữ liệu billing có thể lên tới 8-24 giờ. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| AWS Cost Anomaly Detection | Dùng AI để phát hiện các thay đổi chi phí bất thường, linh hoạt hơn budget cố định. |
| CloudWatch Billing Alarms | Đơn giản hơn nhưng ít tùy chọn chi tiết hơn so với AWS Budgets. |

## 9. How
```bash
# Ví dụ JSON cấu hình Budget qua CLI
{
  "BudgetName": "Monthly_Budget_Limit",
  "BudgetLimit": {
    "Amount": "100",
    "Unit": "USD"
  },
  "TimeUnit": "MONTHLY",
  "BudgetType": "COST"
}
# Lệnh thực thi
aws budgets create-budget --account-id 123456789012 --budget file://budget.json --notifications-with-subscribers file://notifications.json
```

## 10. Production concerns
### Scaling
Khi hệ thống scale lớn, chi phí sẽ tăng theo. Cần cập nhật budget định kỳ để phản ánh đúng quy mô kinh doanh.

### Failure
Dữ liệu Billing không phải thời gian thực. Một vụ hack đào coin có thể tiêu tốn hàng trăm USD trước khi alert kịp gửi đi.

### Monitoring
Theo dõi các notification log để đảm bảo email alert không bị rơi vào thư rác.

## 11. Common mistakes
- Mistake: Chỉ đặt alert dựa trên chi phí thực tế (Actual).
  Fix: Nên đặt thêm alert dựa trên chi phí dự báo (Forecasting) để biết sớm nguy cơ vượt hạn mức.

- Mistake: Dùng email cá nhân ít kiểm tra để nhận alert.
  Fix: Dùng Email nhóm (Distribution List) hoặc tích hợp với Slack/Teams qua SNS và Chatbot.

## 12. Sample project
Thiết lập một Budget Alert 10 USD cho tài khoản Free Tier, cấu hình SNS để gửi tin nhắn về Discord/Slack khi chi phí vượt quá 5 USD.

## 13. Interview
### Core Q&A
1. Q: AWS Budget Alert có tự động tắt các EC2 instance khi vượt ngưỡng không?
   A: Mặc định là không. Budget Alert chỉ gửi thông báo. Để tự động tắt tài nguyên, bạn cần thiết lập "Budget Actions" để gọi AWS Lambda hoặc áp dụng IAM policy giới hạn.

### Scenario
Sếp của bạn yêu cầu đảm bảo tài khoản AWS của team dev không bao giờ tiêu quá 500 USD/tháng. Bạn sẽ triển khai những gì?
Trả lời: Tôi sẽ tạo một AWS Budget với ngưỡng 500 USD. Tôi sẽ thiết lập 3 mức cảnh báo: 50% (để theo dõi), 80% (để rà soát tài nguyên) và 100% (gửi thông báo khẩn cấp). Ngoài ra, tôi sẽ cấu hình Budget Action để ngăn chặn việc tạo thêm tài nguyên mới nếu đạt ngưỡng 100%.

## 14. References
- Official Docs: https://docs.aws.amazon.com/cost-management/latest/userguide/budgets-managing-costs.html
- GitHub Repo: N/A
- Spec / RFC: N/A
- Changelog: N/A

## 15. Real-world Code
Sử dụng Terraform Resource `aws_budgets_budget` để quản lý hạ tầng billing dưới dạng code.

## 16. Community
- Reddit: r/aws - các bài chia sẻ về việc xin hoàn tiền từ AWS sau khi nhận budget alert.
- Stack Overflow: Cách parse SNS notification từ AWS Budget.
- Blog: AWS Cloud Financial Management.
- Talk: AWS re:Invent Cost Optimization.
