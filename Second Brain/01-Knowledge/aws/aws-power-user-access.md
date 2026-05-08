---
created: 2026-05-07
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/security"
related:
  - "[[iam.md]]"
  - "[[aws-iam-hardening.md]]"
  - "[[aws-deny-explicit-dangerous-services.md]]"
---

## 1. What
PowerUserAccess là một AWS Managed Policy (chính sách do AWS quản lý) cung cấp toàn quyền truy cập (full access) vào hầu hết các dịch vụ và tài nguyên của AWS, nhưng ngoại trừ việc quản lý người dùng (Users), nhóm (Groups) và các chính sách định danh (IAM). 

## 2. Why
Trong môi trường phát triển (Development), lập trình viên cần sự linh hoạt tối đa để thử nghiệm và cấu hình các dịch vụ như EC2, S3, RDS mà không muốn phải xin phép cho từng hành động nhỏ. PowerUserAccess ra đời để:
- Trao quyền tự chủ cho Developer để xây dựng hạ tầng.
- Đảm bảo an toàn bằng cách ngăn chặn họ tự cấp thêm quyền cho mình hoặc tạo ra các "lỗ hổng" định danh (IAM privilege escalation).

## 3. Mental Model
Hãy tưởng tượng AWS là một **"Xưởng cơ khí khổng lồ"**:
- **AdministratorAccess** là chủ xưởng: Ông ấy có chìa khóa của tất cả máy móc và cả chìa khóa của tủ hồ sơ nhân sự (có quyền thuê/đuổi thợ).
- **PowerUserAccess** là thợ chính (Senior Mechanic): Bạn có toàn quyền sử dụng mọi loại máy hàn, máy cắt, máy tiện trong xưởng để làm ra sản phẩm. Nhưng bạn không có chìa khóa tủ hồ sơ (IAM). Bạn không thể tuyển thêm thợ mới hoặc thay đổi lương (quyền hạn) của thợ khác.

## 4. Where it fits
IAM Console -> Policies -> Search "PowerUserAccess" -> Attach to Group/User/Role.

## 5. When to use
- Cấp cho các Senior Developers trong tài khoản Sandbox hoặc Development.
- Dùng cho các ứng dụng tự động hóa hạ tầng (như Terraform runner) trong môi trường thử nghiệm nơi bạn tin tưởng hoàn toàn vào mã nguồn.

## 6. When NOT to use
- **Môi trường Production**: Luôn áp dụng nguyên tắc "Least Privilege" (Quyền hạn tối thiểu) - chỉ cấp đúng những gì cần thiết.
- Cấp cho nhân viên mới hoặc người chưa am hiểu sâu về chi phí AWS (vì họ có thể vô tình bật các dịch vụ cực đắt tiền).
- Cấp cho các ứng dụng thực thi (Application Roles) - ứng dụng chỉ nên có quyền truy cập vào các tài nguyên cụ thể mà nó sử dụng.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Tăng tốc độ làm việc (không tốn thời gian xin quyền). | Rủi ro về chi phí (vô tình tạo tài nguyên đắt tiền). |
| Giảm gánh nặng cho đội ngũ Cloud Admin. | Rủi ro xóa nhầm tài nguyên quan trọng của người khác. |
| Ngăn chặn leo thang đặc quyền (Privilege Escalation). | Không kiểm soát được việc tuân thủ quy trình chuẩn. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| AdministratorAccess | Có thêm quyền quản lý IAM, rủi ro bảo mật cao nhất. |
| ReadOnlyAccess | Chỉ được xem, không được tạo mới hay sửa đổi. |
| Customer Managed Policies | Tốt nhất cho bảo mật, chỉ cấp quyền cho các dịch vụ cụ thể dự án đang dùng. |

## 9. How
Để kiểm tra nội dung của chính sách này, bạn có thể xem trong IAM Console. Phần quan trọng nhất là:
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "NotAction": [
                "iam:*",
                "organizations:*",
                "account:*"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "iam:CreateServiceLinkedRole",
                "iam:DeleteServiceLinkedRole",
                "iam:ListRoles",
                "organizations:DescribeOrganization",
                "account:ListRegions"
            ],
            "Resource": "*"
        }
    ]
}
```
Lưu ý: Nó sử dụng `NotAction` để cấm các hành động liên quan đến IAM và Organizations.

## 10. Production concerns
### Blast Radius
Với PowerUserAccess, một sai sót nhỏ (như chạy nhầm script xóa sạch RDS) có thể gây thảm họa cho toàn bộ hệ thống trong tài khoản đó. Luôn sử dụng **Service Control Policies (SCPs)** để giới hạn các hành động nguy hiểm nhất ở cấp Organization.

### Cost
PowerUser có thể khởi chạy các instance đắt tiền (như p4d.24xlarge). Cần thiết lập **AWS Budgets** để cảnh báo sớm.

## 11. Common mistakes
- Mistake: Nghĩ rằng PowerUser không thể xóa tài nguyên.
  Fix: PowerUser có "Full Access" để xóa tài nguyên (S3, EC2, v.v.), họ chỉ không thể xóa User/Role.
- Mistake: Cấp PowerUserAccess cho tài khoản Root.
  Fix: Tài khoản Root luôn có toàn quyền, không cần gán policy này. Tuyệt đối không dùng tài khoản Root hàng ngày.

## 12. Sample project
Thiết lập một "Developer Group" trong AWS IAM:
1. Tạo Group tên `Senior-Developers`.
2. Gán chính sách `PowerUserAccess`.
3. Gán thêm chính sách `IAMUserChangePassword` (để họ tự đổi pass).
4. Thêm các User Senior vào Group này.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt lớn nhất giữa AdministratorAccess và PowerUserAccess là gì?
   A: PowerUserAccess không có quyền quản lý IAM (người dùng, nhóm, vai trò, chính sách) và một số quyền quản trị cấp tài khoản/tổ chức.

2. Q: Tại sao PowerUserAccess vẫn cho phép `iam:CreateServiceLinkedRole`?
   A: Vì nhiều dịch vụ (như Auto Scaling) cần tạo Role riêng để hoạt động. Nếu cấm hoàn toàn `iam:*`, PowerUser sẽ không thể sử dụng đầy đủ các dịch vụ đó.

### Scenario
"Một Developer có quyền PowerUserAccess báo rằng họ không thể tạo một EKS Cluster mới. Tại sao?"
-> Trả lời: EKS thường yêu cầu tạo các IAM Roles cho Worker Nodes hoặc Fargate. Mặc dù PowerUser có quyền tạo Cluster, nhưng họ có thể bị vướng ở bước tạo Role đi kèm nếu quy trình đó yêu cầu quyền `iam:CreateRole` (thứ mà PowerUser không có mặc định).

## 14. References
- AWS Managed Policies: [PowerUserAccess](https://docs.aws.amazon.com/aws-managed-policy/latest/reference/PowerUserAccess.html)

## 15. Real-world Code
Terraform để gán PowerUserAccess cho một Role:
```hcl
resource "aws_iam_role_policy_attachment" "power_user_attach" {
  role       = aws_iam_role.dev_role.name
  policy_arn = "arn:aws:iam::aws:policy/PowerUserAccess"
}
```

## 16. Community
- Reddit: r/aws - thảo luận về việc phân quyền cho Dev team.
- Blog: AWS Security Blog - "How to use managed policies safely".
