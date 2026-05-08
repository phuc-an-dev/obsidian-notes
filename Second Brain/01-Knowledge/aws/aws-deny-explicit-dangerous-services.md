---
created: 2026-05-05
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/security"
related:
  - "[[aws-budget-alert]]"
  - "[[aws-iam-hardening]]"
  - "[[aws-power-user-access]]"
---

## 1. What
AWS Deny Explicit Dangerous Services là việc sử dụng cơ chế "Explicit Deny" trong IAM Policy hoặc Service Control Policies (SCP) để chủ động ngăn chặn việc sử dụng các dịch vụ AWS có rủi ro cao, tốn kém hoặc không được phép trong tổ chức. Trong AWS, một lệnh Deny luôn có ưu tiên cao hơn bất kỳ lệnh Allow nào.

## 2. Why
Mặc định, các tài khoản AWS có quyền truy cập vào hàng trăm dịch vụ ở hàng chục vùng (regions). Một lỗi cấu hình hoặc một tài khoản bị hack có thể dẫn đến việc kẻ tấn công bật các dịch vụ cực đắt tiền (như SageMaker, Redshift) hoặc khởi tạo tài nguyên ở các vùng xa xôi để đào coin nhằm trốn tránh sự giám sát.

## 3. Mental Model
Hãy tưởng tượng AWS là một siêu thị khổng lồ nơi bạn cho nhân viên thẻ tín dụng để đi mua đồ. "Allow" là danh sách những món họ ĐƯỢC mua. "Explicit Deny" giống như việc bạn dán một thông báo lớn ở quầy thu ngân: "Tuyệt đối không bán thuốc lá và rượu cho người cầm thẻ này", dù họ có bất kỳ giấy phép nào khác đi chăng nữa.

## 4. Where it fits
IAM Policy / SCP -> Evaluation Engine (Deny beats Allow) -> AWS Service Access.

## 5. When to use
Nên áp dụng ngay từ đầu cho các tài khoản Production và Sandbox. Các mục tiêu phổ biến bao gồm: chặn các vùng không sử dụng (Region Restriction), chặn các dịch vụ tốn kém không cần thiết, và chặn việc xóa log (CloudTrail).

## 6. When NOT to use
Không nên áp dụng các chính sách Deny quá khắt khe trong môi trường R&D (Nghiên cứu) nơi các kỹ sư cần thử nghiệm các dịch vụ mới, trừ khi các dịch vụ đó vi phạm chính sách tuân thủ của công ty.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Bảo vệ tuyệt đối trước các sai lầm vô tình hoặc cố ý. | Có thể gây khó khăn cho việc triển khai các ứng dụng hợp lệ nếu chính sách quá rộng. |
| Giảm thiểu rủi ro tài chính (Bill Shock). | SCP chỉ áp dụng được khi dùng AWS Organizations. |
| Đơn giản hóa việc quản lý bảo mật ở quy mô lớn. | Việc gỡ lỗi (debug) tại sao một request bị Deny có thể phức tạp. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| IAM Permissions Boundary | Giới hạn quyền tối đa mà một user có thể được cấp, nhưng không mạnh bằng SCP. |
| AWS Budgets | Cảnh báo chi phí nhưng không ngăn chặn được hành động khởi tạo tài nguyên. |

## 9. How
```json
// SCP ví dụ: Chặn mọi hành động ở các vùng không được phép (trừ các dịch vụ toàn cầu)
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "DenyAllOutsideRequestedRegions",
            "Effect": "Deny",
            "NotAction": [
                "iam:*",
                "organizations:*",
                "route53:*",
                "budgets:*",
                "cloudfront:*",
                "support:*"
            ],
            "Resource": "*",
            "Condition": {
                "StringNotEquals": {
                    "aws:RequestedRegion": [
                        "us-east-1",
                        "ap-southeast-1"
                    ]
                }
            }
        }
    ]
}
```

## 10. Production concerns
### Scaling
Sử dụng SCP trong AWS Organizations là cách hiệu quả nhất để áp dụng chính sách Deny cho hàng nghìn account cùng lúc.

### Failure
Một lỗi sai trong câu lệnh Deny có thể làm tê liệt toàn bộ hệ thống (ví dụ: vô tình Deny quyền truy cập của chính các dịch vụ hệ thống).

### Monitoring
Sử dụng CloudTrail để theo dõi các sự kiện có `errorCode: AccessDenied` để điều chỉnh chính sách kịp thời.

## 11. Common mistakes
- Mistake: Chặn vùng nhưng quên loại trừ (whitelist) các dịch vụ Global như IAM, CloudFront.
  Fix: Luôn sử dụng `NotAction` cho các dịch vụ không phụ thuộc vùng.

- Mistake: Dùng IAM Policy thay vì SCP cho các tài khoản con.
  Fix: User có quyền Admin trong tài khoản con vẫn có thể tự gỡ bỏ IAM Policy của chính mình. Chỉ có SCP từ tài khoản Master mới không thể bị gỡ bỏ.

## 12. Sample project
Thiết lập một AWS Organization, tạo một OU (Organizational Unit) tên là "Sandbox" và áp dụng SCP chặn hoàn toàn dịch vụ `aws-portal` (Billing) và các loại instance EC2 cực lớn (ví dụ: `p3.*`, `x1.*`) để tránh rủi ro chi phí.

## 13. Interview
### Core Q&A
1. Q: Điều gì xảy ra nếu một user có IAM Policy "Allow All" nhưng bị áp dụng một SCP "Deny EC2"?
   A: User đó sẽ KHÔNG thể sử dụng EC2. Trong AWS, một lệnh Deny rõ ràng luôn thắng bất kỳ lệnh Allow nào, bất kể nó được đặt ở đâu (Policy, Boundary, hay SCP).

### Scenario
Hệ thống của bạn chỉ chạy ở Singapore (ap-southeast-1). Bạn phát hiện kẻ xấu hack được một tài khoản và tạo hàng loạt máy đào coin ở vùng US-West-2. Bạn sẽ làm gì để ngăn chặn vĩnh viễn?
Trả lời: Tôi sẽ tạo một Service Control Policy (SCP) áp dụng cho toàn bộ account, thực hiện `Deny` tất cả các hành động API nếu `aws:RequestedRegion` không phải là `ap-southeast-1`. Điều này đảm bảo dù kẻ tấn công có quyền Admin, chúng cũng không thể tạo tài nguyên ở bất kỳ vùng nào khác ngoài Singapore.

## 14. References
- Official Docs: https://docs.aws.amazon.com/organizations/latest/userguide/orgs_manage_policies_scps.html
- GitHub Repo: N/A
- Spec / RFC: N/A
- Changelog: N/A

## 15. Real-world Code
Sử dụng Terraform `aws_organizations_policy` để định nghĩa và gắn SCP vào các Root hoặc OU.

## 16. Community
- Reddit: r/aws - thảo luận về các "Blacklist" dịch vụ nên chặn.
- Stack Overflow: Cách gỡ lỗi "Implicit Deny" vs "Explicit Deny".
- Blog: AWS Security Blog về Region Restriction.
- Talk: AWS re:Invent - "Governance at Scale".
