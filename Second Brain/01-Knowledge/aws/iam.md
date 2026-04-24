---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/error-handling"
related:
  - "[[ec2]]"
  - "[[s3]]"
---

## 1. What
AWS IAM (Identity and Access Management) là một dịch vụ web giúp bạn kiểm soát quyền truy cập vào các tài nguyên AWS một cách an toàn. Nó cho phép bạn quản lý ai (xác thực - authentication) và họ có những quyền gì (ủy quyền - authorization) đối với các tài nguyên trong tài khoản AWS của bạn.

## 2. Why
Nếu không có IAM, mọi người sẽ phải dùng chung tài khoản Root (tài khoản chủ), điều này cực kỳ nguy hiểm vì ai cũng có quyền xóa sạch mọi thứ. IAM ra đời để cung cấp khả năng phân quyền chi tiết (granular permissions), giúp áp dụng nguyên tắc "Quyền hạn tối thiểu" (Least Privilege) để bảo vệ hệ thống khỏi các sai sót vô ý hoặc tấn công ác ý.

## 3. Mental Model
Hãy tưởng tượng AWS là một tòa nhà văn phòng cao cấp.
- **IAM User**: Là một nhân viên có thẻ ID riêng để vào tòa nhà.
- **IAM Group**: Là các phòng ban (ví dụ: Phòng Kế toán, Phòng Kỹ thuật). Bạn cấp quyền cho cả phòng thì mọi nhân viên trong đó đều có quyền tương tự.
- **IAM Role**: Là một "chiếc mũ" chức vụ. Bất kỳ nhân viên nào (hoặc máy móc) đội chiếc mũ đó lên sẽ có quyền hạn của chức vụ đó trong một khoảng thời gian nhất định.
- **IAM Policy**: Là một bản nội quy ghi rõ: "Người giữ thẻ này được vào phòng A nhưng không được mở tủ B".

## 4. Where it fits
User/Service -> **IAM Request Evaluation** -> Policy Check -> Allow/Deny -> AWS Resource (S3, EC2, etc.).

## 5. When to use
- Luôn luôn sử dụng để quản lý quyền truy cập cho nhân viên, đối tác.
- Cấp quyền cho các ứng dụng chạy trên EC2 hoặc Lambda để chúng truy cập các dịch vụ khác (S3, DynamoDB).
- Thiết lập xác thực đa yếu tố (MFA) cho các tài khoản nhạy cảm.
- Liên kết (Federate) với các hệ thống định danh bên ngoài như Google, Active Directory.

## 6. When NOT to use
- Đừng dùng IAM User cho các ứng dụng chạy bên ngoài AWS nếu có thể dùng IAM Roles với OIDC.
- Đừng dùng IAM để quản lý người dùng cuối (End-users) của ứng dụng web (nên dùng **Amazon Cognito**).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Dịch vụ miễn phí, tích hợp toàn bộ AWS | Cấu trúc Policy JSON có thể rất phức tạp |
| Quản lý tập trung quyền hạn | Dễ gây ra lỗi "Access Denied" nếu phân quyền quá chặt |
| Hỗ trợ bảo mật mạnh mẽ (MFA, Roles) | Khó kiểm soát khi có hàng ngàn User/Policy |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Amazon Cognito | Dành cho người dùng cuối của ứng dụng (B2C). |
| AWS IAM Identity Center | (Trước là AWS SSO) Tốt hơn để quản lý nhiều tài khoản AWS tập trung. |
| Okta / Auth0 | Giải pháp định danh bên thứ ba, thường liên kết ngược lại IAM. |

## 9. How
Một ví dụ về IAM Policy cho phép chỉ đọc (Read-only) một bucket S3 cụ thể:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:Get*",
                "s3:List*"
            ],
            "Resource": [
                "arn:aws:s3:::my-secure-bucket",
                "arn:aws:s3:::my-secure-bucket/*"
            ]
        }
    ]
}
```

## 10. Production concerns
### Scaling
IAM là một dịch vụ Global, không phụ thuộc vào Region, tự động mở rộng để xử lý hàng tỷ yêu cầu kiểm tra quyền mỗi giây.

### Failure
IAM có độ sẵn sàng cực cao. Tuy nhiên, nếu bạn xóa nhầm một Role quan trọng, các dịch vụ phụ thuộc (như EC2/Lambda) sẽ bị dừng hoạt động ngay lập tức. Luôn có phương án backup/versioning cho các Policy.

### Monitoring
Sử dụng **AWS CloudTrail** để ghi lại mọi hành động liên quan đến IAM (ai đã đổi password, ai đã tạo user mới). Sử dụng **IAM Access Analyzer** để tìm các tài nguyên bị lộ ra ngoài.

## 11. Common mistakes
- Mistake: Sử dụng tài khoản Root cho các công việc hàng ngày.
  Fix: Khóa tài khoản Root bằng MFA và chỉ dùng IAM User/Role.

- Mistake: Sử dụng Access Key/Secret Key cứng trong mã nguồn.
  Fix: Luôn sử dụng **IAM Roles** cho EC2, Lambda hoặc các dịch vụ AWS khác.

## 12. Sample project
Thiết lập một quy trình CI/CD: Tạo một IAM Role cho GitHub Actions để nó có quyền upload file lên S3 thông qua OIDC mà không cần lưu trữ bất kỳ Secret Key nào trên GitHub.

## 13. Interview
### Core Q&A
1. Q: IAM Role và IAM User khác nhau như thế nào?
   A: User gắn liền với một người/máy cụ thể và có thông tin đăng nhập lâu dài. Role không có thông tin đăng nhập mà được "đảm nhận" (assume) bởi bất kỳ ai có quyền, cung cấp thông tin xác thực tạm thời và an toàn hơn.

### Scenario
1. Q: Bạn làm gì nếu một lập trình viên báo rằng họ bị lỗi "Access Denied" khi gọi API S3?
   A: Tôi sẽ kiểm tra IAM Policy của họ, xem Resource ARN có đúng không, Action có bị chặn bởi SCP (Service Control Policies) ở tầng Organization không, và kiểm tra CloudTrail để xem lý do chính xác dẫn đến việc bị Deny.

## 14. References
- Official Docs: https://docs.aws.amazon.com/iam/
- Security Best Practices: https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html
- IAM Policy Simulator: https://policysim.aws.amazon.com/

## 15. Real-world Code
Sử dụng các công cụ như `iam-policy-json-to-terraform` để chuyển đổi các chính sách sang mã nguồn quản lý hạ tầng (IaC).

## 16. Community
- YouTube: "IAM Best Practices" - AWS re:Invent sessions.
- Stack Overflow: [aws-iam] tag.
- Blog: AWS Security Blog.
