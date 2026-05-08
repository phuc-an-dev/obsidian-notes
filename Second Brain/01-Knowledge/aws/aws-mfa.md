---
created: 2026-05-05
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/security"
related:
  - "[[iam.md]]"
  - "[[ListAccessKeys in AWS.md]]"
  - "[[aws-cloudtrail.md]]"
  - "[[aws-iam-hardening]]"
---

## 1. What
AWS MFA (Multi-Factor Authentication) là một lớp bảo mật bổ sung yêu cầu người dùng cung cấp một yếu tố thứ hai (như mã số từ điện thoại hoặc thiết bị phần cứng) ngoài mật khẩu khi đăng nhập vào AWS Management Console hoặc thực hiện các thao tác quan trọng qua CLI.

## 2. Why
Mật khẩu dù mạnh đến đâu cũng có thể bị lộ qua tấn công Phishing, Keylogger hoặc dùng chung mật khẩu. Trong AWS, nếu tài khoản Root hoặc IAM user có quyền Admin bị lộ mật khẩu, toàn bộ hạ tầng và dữ liệu của công ty có thể bị xóa sạch hoặc bị đánh cắp trong vài phút.

## 3. Mental Model
Hãy tưởng tượng việc đăng nhập AWS giống như mở một cánh cửa bảo mật cao. Mật khẩu là chiếc chìa khóa vật lý. MFA giống như một thiết bị quét vân tay hoặc một mật mã thay đổi liên tục trên điện thoại. Để vào nhà, bạn phải có cả chìa khóa VÀ vân tay.

## 4. Where it fits
User Login -> Password Verification -> MFA Challenge -> Access Granted.

## 5. When to use
MFA là bắt buộc (Mandatory) cho: 1. Tài khoản Root (Root Account). 2. Tất cả các IAM user có quyền truy cập vào Console. 3. Các user có quyền xóa tài nguyên quan trọng (như S3 buckets hoặc database).

## 6. When NOT to use
Không có trường hợp nào không nên dùng MFA cho người dùng thật. Tuy nhiên, đối với các "Service Accounts" dùng cho máy móc (như CI/CD runner), chúng ta dùng IAM Roles và Temporary Credentials thay vì user có password và MFA.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Chống lại 99.9% các cuộc tấn công chiếm đoạt tài khoản. | Thêm một bước phiền phức mỗi khi đăng nhập. |
| Đáp ứng các yêu cầu tuân thủ (Compliance) như HIPAA, PCI-DSS. | Nguy cơ bị khóa tài khoản nếu mất thiết bị MFA (điện thoại bị hỏng/mất). |
| Miễn phí (nếu dùng Virtual MFA app). | Cần quy trình recovery phức tạp cho các tài khoản quan trọng. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Hardware MFA Device | An toàn nhất, chống được các cuộc tấn công qua mạng nhưng tốn phí mua thiết bị. |
| Virtual MFA (Google Authenticator/Authy) | Tiện lợi, miễn phí, phổ biến nhất. |
| FIDO Security Keys (Yubikey) | Tốc độ nhanh, bảo mật cực cao, hỗ trợ nhiều nền tảng. |

## 9. How
```bash
# Ví dụ IAM Policy bắt buộc user phải có MFA mới được thực hiện thao tác
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "BlockMostAccessUnlessSignedWithMFA",
            "Effect": "Deny",
            "NotAction": [
                "iam:CreateVirtualMFADevice",
                "iam:EnableMFADevice",
                "iam:ResyncMFADevice",
                "iam:ListAccountAliases",
                "iam:ListUsers"
            ],
            "Resource": "*",
            "Condition": {
                "BoolIfExists": {
                    "aws:MultiFactorAuthPresent": "false"
                }
            }
        }
    ]
}
```

## 10. Production concerns
### Scaling
Khi số lượng nhân viên lớn, hãy dùng AWS IAM Identity Center (SSO) để quản lý MFA tập trung thay vì cấu hình từng IAM user đơn lẻ.

### Failure
Nếu mất thiết bị MFA của Root account, bạn phải liên hệ với AWS Support qua một quy trình xác minh danh tính cực kỳ nghiêm ngặt.

### Monitoring
Sử dụng IAM Credential Report định kỳ để kiểm tra những user nào chưa bật MFA.

## 11. Common mistakes
- Mistake: Chỉ bật MFA cho user mà quên bật cho Root account.
  Fix: Root account là mục tiêu nguy hiểm nhất, phải bật MFA đầu tiên và cất thiết bị MFA ở nơi an toàn.

- Mistake: Không có phương án dự phòng khi mất điện thoại chứa Virtual MFA.
  Fix: Lưu trữ mã Recovery code (nếu có) hoặc dùng thiết bị MFA phần cứng lưu trong két sắt công ty.

## 12. Sample project
Thiết lập một IAM Policy áp dụng cho toàn bộ group "Developers", yêu cầu họ phải xác thực MFA trước khi được phép xem nội dung các S3 bucket chứa dữ liệu khách hàng.

## 13. Interview
### Core Q&A
1. Q: Nếu tôi dùng AWS CLI, làm thế nào để sử dụng MFA?
   A: Bạn sử dụng lệnh `aws sts get-session-token` kèm theo mã MFA để lấy temporary credentials (AccessKey, SecretKey, SessionToken). Sau đó dùng bộ key này để thực hiện các lệnh CLI tiếp theo.

### Scenario
Một lập trình viên báo rằng họ làm mất điện thoại có chứa mã MFA và không thể vào làm việc. Bạn sẽ xử lý thế nào?
Trả lời: Là Admin, tôi sẽ: 1. Xác minh danh tính của lập trình viên đó qua các kênh khác. 2. Vào IAM Console, tạm thời deactivate thiết bị MFA cũ của họ. 3. Yêu cầu họ thiết lập thiết bị MFA mới ngay lập tức khi đăng nhập lại. 4. Nhắc nhở họ về việc bảo quản thiết bị hoặc dùng các app có backup như Authy.

## 14. References
- Official Docs: https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_mfa.html
- GitHub Repo: N/A
- Spec / RFC: N/A
- Changelog: N/A

## 15. Real-world Code
Sử dụng AWS Config Rule `iam-user-mfa-enabled` để tự động kiểm tra và đánh dấu các user vi phạm chính sách bảo mật.

## 16. Community
- Reddit: r/aws - các câu chuyện "kinh dị" khi bị hack do không bật MFA.
- Stack Overflow: Cách script hóa việc kiểm tra MFA trạng thái của user.
- Blog: AWS Security Blog - "Ten security golden rules".
- Talk: AWS re:Invent - "IAM Best Practices".
