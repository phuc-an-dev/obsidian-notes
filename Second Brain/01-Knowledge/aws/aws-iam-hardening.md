---
created: 2026-05-07
tags:
  - "#type/pattern"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/security"
related:
  - "[[iam]]"
  - "[[aws-mfa]]"
  - "[[aws-deny-explicit-dangerous-services]]"
  - "[[aws-power-user-access]]"
---

## 1. What
AWS IAM Hardening là quá trình áp dụng các biện pháp bảo mật nâng cao và các phương pháp hay nhất (best practices) để thắt chặt quyền truy cập vào tài khoản AWS. Mục tiêu là giảm thiểu bề mặt tấn công và đảm bảo rằng chỉ những người/thành phần hợp pháp mới có quyền thực hiện các hành động cụ thể.

## 2. Why
IAM là "chìa khóa vạn năng" vào hạ tầng đám mây của bạn. Nếu IAM bị cấu hình lỏng lẻo:
- Hacker có thể chiếm quyền Root và xóa sạch dữ liệu.
- Rò rỉ Access Keys dẫn đến việc tài nguyên bị lợi dụng để đào tiền ảo (cryptojacking).
- Nhân viên cũ vẫn còn quyền truy cập vào hệ thống nhạy cảm.
Hardening giúp tuân thủ các tiêu chuẩn bảo mật như CIS AWS Foundations Benchmark.

## 3. Mental Model
Hãy tưởng tượng AWS là một **ngân hàng**.
- IAM cơ bản là việc cấp thẻ cho nhân viên.
- IAM Hardening là:
    - Bắt buộc nhân viên phải quét vân tay + nhập mã PIN (MFA).
    - Nhân viên quầy chỉ có chìa khóa ngăn kéo, không có chìa khóa kho tiền (Least Privilege).
    - Camera giám sát mọi lúc nhân viên mở két (CloudTrail).
    - Thay ổ khóa định kỳ mỗi 90 ngày (Credential Rotation).

## 4. Where it fits
IAM Hardening nằm ở tầng **Security** của AWS Well-Architected Framework. Nó là lớp phòng thủ đầu tiên và quan trọng nhất trước khi traffic chạm đến EC2, S3 hay RDS.

## 5. When to use
- Ngay khi khởi tạo một tài khoản AWS mới.
- Định kỳ hằng quý (Security Audit).
- Trước khi chuẩn bị cho các đợt kiểm tra tuân thủ (Compliance) như SOC2, PCI-DSS.
- Khi có thay đổi nhân sự quan trọng trong team DevOps.

## 6. When NOT to use
Không có trường hợp nào không nên dùng. Tuy nhiên, trong môi trường Sandbox/Personal Lab, bạn có thể nới lỏng một chút để thử nghiệm, nhưng tuyệt đối không được nới lỏng cho tài khoản Root.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Giảm thiểu tối đa rủi ro bị hack tài khoản. | Quy trình làm việc của Developer có thể chậm lại (phải dùng MFA thường xuyên). |
| Dễ dàng truy vết khi có sự cố. | Tăng khối lượng công việc quản trị cho team Security/DevOps. |
| Đáp ứng các yêu cầu khắt khe về tuân thủ. | Có thể gây lỗi "Access Denied" cho các ứng dụng cũ nếu phân quyền quá chặt. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| AWS IAM Identity Center | Cách tiếp cận hiện đại hơn cho môi trường Multi-account, thay thế cho việc tạo IAM User thủ công. |
| External IDP (Okta/Auth0) | Quản lý định danh tập trung bên ngoài AWS, chuyên nghiệp hơn cho doanh nghiệp lớn. |

## 9. How
Các bước thiết yếu để Hardening IAM:
1. **Khóa tài khoản Root**: Không tạo Access Key cho Root, bật MFA và cất thông tin đăng nhập vào két sắt.
2. **Bật MFA**: Bắt buộc cho tất cả người dùng có quyền truy cập vào Console.
3. **Nguyên tắc Quyền hạn tối thiểu (Least Privilege)**: Sử dụng Managed Policies hoặc Customer Managed Policies thay vì Full Access.
4. **Cấu hình Password Policy**: Độ dài tối thiểu 14 ký tự, yêu cầu ký tự đặc biệt, hết hạn sau 90 ngày.
5. **Xóa các Credentials cũ**: Tự động vô hiệu hóa Access Keys không sử dụng sau 90 ngày.
6. **Sử dụng IAM Roles**: Tuyệt đối không lưu Access Keys trên EC2/Lambda, sử dụng Instance Profiles thay thế.

## 10. Production concerns
### Scaling
Khi số lượng User tăng lên, hãy sử dụng **IAM Groups** để quản lý quyền thay vì gán cho từng cá nhân. Đối với môi trường lớn (Multi-account), sử dụng **Service Control Policies (SCPs)** ở cấp Organization để áp đặt các rào cản bảo mật (Guardrails).

### Failure
Nếu cấu hình sai SCP hoặc IAM Policy, bạn có thể bị khóa khỏi chính tài khoản của mình. Luôn giữ ít nhất một "Break-glass" account (tài khoản khẩn cấp) được cấu hình cực kỳ bảo mật nhưng có quyền quản trị cao.

### Monitoring
Sử dụng **IAM Access Analyzer** để tìm các tài nguyên bị chia sẻ ra bên ngoài trái phép. Sử dụng **AWS Config** để giám sát việc tuân thủ Password Policy.

## 11. Common mistakes
- Mistake: Chia sẻ Access Keys giữa các thành viên trong team hoặc giữa các ứng dụng.
  Fix: Mỗi người dùng/ứng dụng phải có định danh riêng biệt.

- Mistake: Để Access Keys trong file `.env` hoặc commit lên GitHub.
  Fix: Sử dụng IAM Roles cho tài nguyên AWS và Secrets Manager cho ứng dụng bên ngoài.

## 12. Sample project
Viết một Lambda function định kỳ chạy hàng ngày để quét danh sách IAM Users:
- Gửi thông báo Slack nếu phát hiện User chưa bật MFA.
- Tự động vô hiệu hóa Access Keys đã cũ hơn 180 ngày.

## 13. Interview
### Core Q&A
1. Q: Làm thế nào để đảm bảo một IAM User chỉ có thể thực hiện hành động khi họ đã xác thực MFA?
   A: Sử dụng phần `Condition` trong IAM Policy với key `aws:MultiFactorAuthPresent: "true"`.

2. Q: Sự khác biệt giữa Service-Linked Role và IAM Role thông thường là gì?
   A: Service-Linked Role được AWS tự động tạo cho các dịch vụ cụ thể (như Auto Scaling) để thay mặt bạn gọi các dịch vụ khác, bạn không thể chỉnh sửa quyền của nó trừ khi AWS cho phép.

### Scenario
Hacker chiếm được Access Key của một nhân viên. Tuy nhiên, chúng không thể xóa được S3 Bucket dù nhân viên đó có quyền Admin. Tại sao?
Trả lời: Có thể do tài khoản đang nằm trong một AWS Organization và có một **Service Control Policy (SCP)** cấm hành động `s3:DeleteBucket`, hoặc bucket đó có chính sách **Object Lock** đang được bật.

## 14. References
- CIS AWS Foundations Benchmark: https://www.cisecurity.org/benchmark/amazon_web_services
- AWS IAM Best Practices Docs: https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html

## 15. Real-world Code
Terraform để thiết lập Password Policy nghiêm ngặt:
```hcl
resource "aws_iam_account_password_policy" "strict" {
  minimum_password_length        = 14
  require_lowercase_characters   = true
  require_numbers                = true
  require_uppercase_characters   = true
  require_symbols                = true
  allow_users_to_change_password = true
  password_reuse_prevention      = 24
  max_password_age               = 90
}
```

## 16. Community
- YouTube: "AWS IAM Deep Dive" - AWS re:Invent.
- Blog: "The 10 commandments of AWS Security" - Cloudonaut.
- Tool: `prowler` - Công cụ mã nguồn mở phổ biến để audit bảo mật AWS theo tiêu chuẩn CIS.
