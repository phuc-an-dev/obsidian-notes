---
created: 2026-05-05
tags:
  - "#type/tutorial"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/error-handling"
related:
  - "[[aws-budget-alert]]"
  - "[[aws-guardduty]]"
---

## 1. What
Đây là checklist và quy trình rà soát định kỳ nhằm phát hiện các tài nguyên lạ (unauthorized resources) trong tài khoản AWS. Mục tiêu chính là phát hiện sớm các dấu hiệu bị hack tài khoản để đào coin (crypto mining) hoặc đánh cắp dữ liệu.

## 2. Why
Kẻ tấn công thường tạo ra các instance cấu hình cao (GPU) hoặc hàng loạt instance nhỏ ở các Region mà người dùng ít khi sử dụng (ví dụ: Tokyo, Middle East) để tránh bị phát hiện. Nếu không rà soát, hóa đơn AWS có thể tăng vọt lên hàng chục ngàn USD chỉ sau vài ngày.

## 3. Mental Model
Hãy tưởng tượng tài khoản AWS của bạn như một tòa nhà chung cư nhiều phòng (Regions). Bạn thường chỉ ở phòng 1 và 2. Kẻ trộm lẻn vào và mở tiệc ở phòng 10, 11 (những vùng bạn không bao giờ ngó tới). Checklist này giống như việc bảo vệ đi tuần tra tất cả các phòng để đảm bảo không có khách lạ đang sử dụng điện nước của bạn.

## 4. Where it fits
AWS Console / CLI -> Global View -> Regional Resources -> Billing Dashboard.
Quy trình này nên được thực hiện hàng tuần hoặc tự động hóa qua AWS Config.

## 5. When to use
- Khi hóa đơn AWS (Billing) tăng đột biến không rõ nguyên nhân.
- Sau khi phát hiện lộ lọt IAM Access Key hoặc Password.
- Định kỳ hàng tháng (Monthly Health Check) để đảm bảo vệ sinh tài nguyên.
- Ngay khi nhận được cảnh báo từ AWS GuardDuty hoặc AWS Budget.

## 6. When NOT to use
- Không dùng thay thế cho các dịch vụ bảo mật tự động như GuardDuty hay Security Hub (đây là bước kiểm tra thủ công/bổ trợ).
- Không dùng nếu bạn đang thực hiện các đợt Load Test lớn (vì tài nguyên tăng là bình thường).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Phát hiện được các tài nguyên mà tool tự động bỏ sót | Tốn thời gian nếu làm thủ công trên nhiều Region |
| Giúp hiểu rõ cấu trúc hạ tầng hiện tại | Dễ gây nhầm lẫn nếu không nắm rõ các resource hệ thống tự tạo |
| Tiết kiệm chi phí đáng kể nếu phát hiện sớm | Cần quyền Admin để rà soát toàn diện |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| AWS Config | Tự động theo dõi thay đổi cấu hình, nhưng có phí |
| Prowler / ScoutSuite | Tool mã nguồn mở quét bảo mật toàn diện, rất mạnh mẽ |
| AWS Trusted Advisor | Chỉ ra các tài nguyên dư thừa nhưng không chuyên sâu về bảo mật |

## 9. How
```bash
# Sử dụng AWS CLI để kiểm tra instance trên tất cả các regions (script mẫu)
for region in $(aws ec2 describe-regions --query 'Regions[].RegionName' --output text); do
  echo "Checking region $region..."
  aws ec2 describe-instances --region $region \
    --query 'Reservations[].Instances[].{ID:InstanceId,Type:InstanceType,Status:State.Name}' \
    --output table
done

# Kiểm tra các tài nguyên tốn tiền nhất trong tháng
aws ce get-cost-and-usage \
    --time-period Start=2026-04-01,End=2026-05-01 \
    --granularity MONTHLY \
    --metrics "BlendedCost" \
    --group-by Type=DIMENSION,Key=SERVICE
```

## 10. Production concerns
### Scaling
Với hàng trăm tài khoản AWS (Multi-account), sử dụng AWS Organizations và CloudFormation StackSets để triển khai các rule rà soát tự động trên toàn bộ tổ chức.

### Failure
Nếu phát hiện resource lạ, tuyệt đối không xóa ngay lập tức nếu chưa sao lưu log hoặc snapshot để phục vụ điều tra (Forensics). Hãy stop và cô lập chúng trước.

### Monitoring
Thiết lập CloudWatch Alarms cho Billing. Ví dụ: Nếu tổng chi phí vượt quá 10 USD so với dự kiến, gửi thông báo ngay về Slack.

## 11. Common mistakes
- Mistake: Chỉ kiểm tra Region mình hay dùng (như us-east-1).
  Fix: Kẻ tấn công luôn ưu tiên các Region "vắng vẻ" (ap-northeast-3, sa-east-1) để đặt botnet/miner.

- Mistake: Quên kiểm tra các tài nguyên không phải EC2 như SageMaker Notebooks, Lambda functions, hay CloudFront distributions.
  Fix: Kiểm tra mục "Resources by Region" trong AWS Billing Dashboard để thấy bức tranh toàn cảnh.

## 12. Sample project
Viết một Lambda function chạy hàng ngày (EventBridge trigger). Function này sẽ quét tất cả Regions, tìm các EC2 instance không có tag `Project` hoặc `Owner` và gửi danh sách đó về email cho Admin.

## 13. Interview
### Core Q&A
1. Q: Làm thế nào để nhanh chóng biết có tài nguyên lạ đang chạy trong tài khoản AWS mà không cần vào từng Region?
   A: Cách nhanh nhất là kiểm tra AWS Billing Dashboard -> "Bills" -> "Track costs by region". Hoặc sử dụng AWS Resource Explorer để search global.
2. Q: Bạn thấy một Instance loại "p3.16xlarge" (rất đắt) đang chạy ở Region lạ, bạn sẽ làm gì?
   A: 1. Stop instance ngay lập tức. 2. Thu hồi toàn bộ Access Keys của tài khoản liên quan. 3. Đổi mật khẩu root/admin. 4. Kiểm tra CloudTrail để tìm xem ai/key nào đã tạo instance đó.

### Scenario
Tài khoản AWS của bạn nhận được cảnh báo "High CPU Usage" từ một Region bạn không bao giờ sử dụng. Bạn kiểm tra và thấy 50 instance đang đào Bitcoin. Hãy nêu quy trình xử lý?
Bước 1: Chụp ảnh màn hình làm bằng chứng. Bước 2: Dùng script CLI để terminate toàn bộ instance đó đồng loạt. Bước 3: Deactivate toàn bộ IAM Users bị nghi ngờ. Bước 4: Kiểm tra IAM Roles và Policies xem có bị sửa đổi không. Bước 5: Liên hệ AWS Support để yêu cầu refund (thường họ sẽ hỗ trợ nếu là lần đầu bị hack).

## 14. References
- Official Docs: https://docs.aws.amazon.com/whitepapers/latest/aws-security-incident-response-guide/welcome.html
- GitHub Repo: https://github.com/toniblyx/prowler
- Spec / RFC: N/A
- Changelog: AWS Resource Explorer launch.

## 15. Real-world Code
https://github.com/aws-samples/aws-security-audit-scripts (Các script audit bảo mật từ chính AWS)

## 16. Community
- Reddit: r/aws
- Stack Overflow: Tag #aws-security
- Blog: AWS Security Blog
- Talk: "How to respond to security incidents" (AWS re:Inforce).
