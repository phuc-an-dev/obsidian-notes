---
created: 2026-05-04
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/iam"
related:
  - "[[iam]]"
  - "[[s3]]"
---

## 1. What
**IAM Policy** là một tài liệu JSON trong AWS dùng để định nghĩa quyền hạn (permissions). Nó quy định cụ thể ai (Principal) có thể làm gì (Action) trên tài nguyên nào (Resource) và trong điều kiện nào (Condition). Chính sách này có thể được gắn vào User, Group, hoặc Role.

## 2. Why
Trong một hệ thống đám mây phức tạp, việc cấp quyền quá rộng (như quyền Admin cho tất cả mọi người) là một rủi ro bảo mật cực lớn. IAM Policy cho phép thực hiện nguyên tắc **"Quyền hạn tối thiểu" (Least Privilege)**, đảm bảo mỗi thực thể chỉ có đúng những quyền cần thiết để hoàn thành công việc, giúp giảm thiểu thiệt hại nếu tài khoản bị xâm nhập.

## 3. Mental Model
Hãy tưởng tượng IAM Policy như một chiếc **"Thẻ ra vào (Access Badge)"** của một tòa nhà văn phòng. 
- Trên thẻ có ghi rõ: "Bạn được phép vào phòng họp (Action: s3:ListBucket)".
- "Nhưng bạn không được phép vào phòng server (Effect: Deny)".
- "Và thẻ này chỉ có tác dụng trong giờ hành chính (Condition)".
Khi bạn quẹt thẻ vào một cánh cửa (Resource), hệ thống sẽ kiểm tra nội dung trên thẻ để quyết định cho bạn vào hay không.

## 4. Where it fits
Nó là lớp bảo mật trung tâm điều khiển mọi tương tác trong AWS:
`Identity (User/Role) -> Attached Policy -> Policy Evaluation Logic -> AWS Resource`

## 5. When to use
- Khi cần cấp quyền cho một lập trình viên truy cập vào S3.
- Khi cần cấu hình một Lambda function để nó có thể ghi log vào CloudWatch.
- Khi cần thiết lập rào cản bảo mật (Permission Boundary) để giới hạn quyền tối đa của một User.

## 6. When NOT to use
- Khi muốn quản lý quyền truy cập bên trong logic của ứng dụng (ví dụ: phân quyền admin/user của trang web - hãy dùng database riêng).
- Khi muốn quản lý quyền truy cập ở mức độ tổ chức lớn (Organization), nên kết hợp thêm với **Service Control Policies (SCPs)**.

## 7. Trade-offs
| Loại Policy | Pros | Cons |
|-------------|------|------|
| **AWS Managed** | Do AWS quản lý, tự động cập nhật, dễ dùng. | Không thể chỉnh sửa, đôi khi cấp quyền quá rộng. |
| **Customer Managed** | Linh hoạt, có thể tái sử dụng, hỗ trợ versioning. | Phải tự quản lý, bảo trì. |
| **Inline Policy** | Gắn chặt vào 1 identity, không sợ bị xóa nhầm. | Không thể tái sử dụng, khó quản lý ở quy mô lớn. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| **Resource-based Policy** | Gắn trực tiếp vào tài nguyên (như S3 Bucket Policy), dùng cho truy cập xuyên tài khoản (Cross-account). |
| **ACL (Access Control List)** | Cách quản lý cũ, hiện nay AWS khuyến nghị dùng Policy hơn. |
| **SCPs** | Dùng để đặt giới hạn tối đa cho toàn bộ tài khoản trong AWS Organizations. |

## 9. How
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowS3ReadAccess",
            "Effect": "Allow",
            "Action": [
                "s3:Get*",
                "s3:List*"
            ],
            "Resource": "arn:aws:s3:::my-company-bucket/*",
            "Condition": {
                "IpAddress": {
                    "aws:SourceIp": "1.2.3.4/32"
                }
            }
        }
    ]
}
```

## 10. Production concerns
### Scaling
Sử dụng **Group-based permissions** thay vì gắn policy trực tiếp vào từng User. Khi có thêm 100 nhân viên mới, bạn chỉ cần thêm họ vào Group tương ứng.

### Failure
Luôn ghi nhớ: **Explicit Deny luôn thắng Explicit Allow**. Nếu có bất kỳ policy nào (kể cả SCP hay Resource-based) từ chối quyền, thì hành động đó sẽ bị chặn hoàn toàn dù các policy khác có cho phép.

### Monitoring
Sử dụng **IAM Access Analyzer** để kiểm tra các policy có cấp quyền truy cập công khai (public) hoặc xuyên tài khoản không mong muốn hay không.

## 11. Common mistakes
- **Mistake**: Sử dụng wildcard quá đà: `"Action": "*"` trên `"Resource": "*"`.
  **Fix**: Luôn liệt kê cụ thể các action và ARN của tài nguyên.

- **Mistake**: Quên rằng IAM Policy có giới hạn kích thước (thường là 6KB - 10KB tùy loại).
  **Fix**: Chia nhỏ thành nhiều Managed Policies nếu chính sách quá dài.

## 12. Sample project
Tạo một Customer Managed Policy cho team Frontend, chỉ cho phép họ upload và xóa file trong thư mục `static/` của một S3 Bucket cụ thể, và bắt buộc phải sử dụng MFA (Multi-Factor Authentication).

## 13. Interview
### Core Q&A
1. **Q**: Sự khác biệt giữa Managed Policy và Inline Policy là gì?
   **A**: Managed Policy là một đối tượng độc lập, có thể gắn vào nhiều identity và có phiên bản. Inline Policy là một phần không thể tách rời của identity, không thể tái sử dụng.
2. **Q**: Các thành phần chính của một Statement trong Policy là gì?
   **A**: Effect (Allow/Deny), Action (hành động), Resource (tài nguyên), và Condition (điều kiện tùy chọn).
3. **Q**: Thứ tự ưu tiên khi đánh giá Policy của AWS là gì?
   **A**: Mặc định là Deny -> Kiểm tra Explicit Deny -> Kiểm tra Explicit Allow -> Nếu không có Allow thì mặc định là Deny.

### Scenario
**Tình huống**: Bạn đã gắn Policy "Allow S3 Read" cho một User, nhưng User đó vẫn báo lỗi "Access Denied" khi đọc file. Bạn sẽ kiểm tra những gì?
**Trả lời**: Tôi sẽ kiểm tra: 1. Có Policy nào khác đang Deny không? 2. S3 Bucket đó có Bucket Policy (Resource-based) chặn truy cập không? 3. Có SCP nào từ Organizations đang chặn không? 4. Condition trong Policy có đang không thỏa mãn (ví dụ sai IP) không?

## 14. References
- Official Docs: [IAM JSON Policy Reference](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies.html)
- AWS Workshop: [IAM Policy Simulator](https://policysim.aws.amazon.com/)

## 15. Real-world Code
- Repository `aws-iam-policy-templates` trên GitHub chứa hàng nghìn mẫu policy cho các service thông dụng.

## 16. Community
- Reddit: r/aws
- Stack Overflow: Tag [aws-iam]
- Blog: AWS Security Blog.
