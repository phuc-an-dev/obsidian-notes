---
created: 2026-05-07
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/security"
related:
  - "[[s3]]"
  - "[[s3-presigned-url]]"
  - "[[iam]]"
  - "[[aws-ec2-security-groups]]"
  - "[[aws-block-public-access]]"
---

## 1. What
AWS Bucket Policy là một chính sách dựa trên tài nguyên (resource-based policy) được viết bằng JSON, cho phép bạn cấp quyền truy cập vào các bucket S3 và các đối tượng (objects) bên trong chúng. Nó được gắn trực tiếp vào bucket thay vì gắn vào người dùng (IAM User) hay vai trò (IAM Role).

## 2. Why
Trước khi có Bucket Policy, việc quản lý quyền truy cập S3 chủ yếu dựa trên IAM Policies. Tuy nhiên, IAM Policies có giới hạn về kích thước và chỉ có thể quản lý những gì người dùng trong cùng tài khoản được phép làm. Bucket Policy giải quyết vấn đề bằng cách cho phép quản lý tập trung quyền truy cập vào dữ liệu, bao gồm cả việc cấp quyền cho các tài khoản AWS khác (cross-account) hoặc cho phép truy cập công khai (public access).

## 3. Mental Model
Hãy tưởng tượng S3 Bucket là một kho lưu trữ dữ liệu.
- IAM Policy giống như một tấm thẻ nhân viên: Trên thẻ ghi rõ nhân viên đó được phép vào kho nào.
- Bucket Policy giống như một bản hướng dẫn dán ngay cửa kho: Nó quy định rõ ai (dù có thẻ hay không, dù là người của công ty hay khách) được phép vào và làm gì trong kho đó.

## 4. Where it fits
Request -> S3 Endpoint -> IAM Evaluation + Bucket Policy Evaluation -> Access Granted/Denied.
Tất cả các quyền truy cập được đánh giá theo nguyên tắc: Nếu có bất kỳ lệnh Deny tường minh nào, yêu cầu sẽ bị từ chối ngay lập tức.

## 5. When to use
- Khi cần cấp quyền truy cập cho người dùng từ các tài khoản AWS khác (Cross-account access).
- Khi cần cấp quyền truy cập công khai cho các object (ví dụ: host website tĩnh).
- Khi muốn áp đặt các điều kiện bảo mật nghiêm ngặt trên toàn bộ bucket (ví dụ: chỉ cho phép truy cập qua HTTPS).
- Khi IAM Policy của người dùng đã quá lớn và chạm giới hạn kích thước file.

## 6. When NOT to use
- Khi bạn chỉ cần quản lý quyền cho một vài cá nhân trong cùng một tài khoản AWS (nên dùng IAM Policy để dễ quản lý tập trung).
- Khi quyền truy cập cần thay đổi quá thường xuyên và phức tạp dựa trên logic ứng dụng (có thể cân nhắc S3 Access Points).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Quản lý quyền truy cập tập trung tại mức tài nguyên. | Dễ gây nhầm lẫn khi kết hợp với IAM Policy (Conflict). |
| Hỗ trợ Cross-account và Public access dễ dàng. | Nếu cấu hình sai có thể vô tình làm lộ dữ liệu ra Internet. |
| Cho phép kiểm soát dựa trên địa chỉ IP hoặc VPC. | Giới hạn kích thước policy JSON (20 KB). |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| IAM Policy | Gắn vào User/Role, không hỗ trợ cấp quyền cho anonymous users hoặc cross-account mà không có trust relationship. |
| S3 ACLs | Phương pháp cũ, ít linh hoạt hơn, không hỗ trợ các điều kiện (Conditions) phức tạp. |
| Pre-signed URLs | Dùng cho quyền truy cập tạm thời (temporary) cho một object cụ thể. |

## 9. How
Dưới đây là một ví dụ về Bucket Policy chỉ cho phép truy cập qua kết nối bảo mật (HTTPS) và từ chối mọi truy cập HTTP thường.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSSLRequestsOnly",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::my-secure-bucket",
        "arn:aws:s3:::my-secure-bucket/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

## 10. Production concerns
### Scaling
Bucket Policy được đánh giá cực nhanh bởi AWS, không gây ảnh hưởng đến latency của yêu cầu S3.

### Failure
Nếu policy bị xóa hoặc cấu hình sai (Deny nhầm chính mình), root user của account vẫn có thể vào để reset policy.

### Monitoring
Sử dụng AWS CloudTrail để ghi lại các thay đổi đối với Bucket Policy (PutBucketPolicy) và các yêu cầu bị từ chối do policy (Access Denied).

## 11. Common mistakes
- Mistake: Sử dụng "Principal": "*" trong một Statement có Effect: "Allow" mà không có Condition, dẫn đến việc công khai toàn bộ dữ liệu ra Internet.
  Fix: Luôn sử dụng Condition (ví dụ giới hạn IP hoặc VPC) hoặc chỉ định Principal cụ thể.

- Mistake: Quên không cấp quyền cho cả Bucket (arn:aws:s3:::bucket) và Object (arn:aws:s3:::bucket/*) khi cần các thao tác như ListBucket và GetObject.
  Fix: Khai báo cả hai ARN trong phần Resource của policy.

## 12. Sample project
Tạo một Bucket S3 dùng để chứa log từ nhiều tài khoản AWS khác nhau. Yêu cầu:
- Mỗi tài khoản chỉ được phép ghi log vào thư mục mang ID của tài khoản đó.
- Không tài khoản nào được phép xóa log sau khi đã ghi.
- Bắt buộc dùng HTTPS.

## 13. Interview
### Core Q&A
1. Q: Điều gì xảy ra nếu IAM Policy cho phép (Allow) nhưng Bucket Policy từ chối (Deny)?
   A: Kết quả cuối cùng sẽ là Deny. Trong AWS, một lệnh Deny tường minh luôn có ưu tiên cao nhất so với bất kỳ lệnh Allow nào.

2. Q: Làm thế nào để cấp quyền cho một tài khoản AWS khác truy cập vào Bucket của bạn?
   A: Bạn cần thêm một Statement trong Bucket Policy với Principal là ARN của tài khoản đó (hoặc IAM User/Role của họ) và hành động (Action) mong muốn.

3. Q: Sự khác biệt giữa Resource-based policy và Identity-based policy là gì?
   A: Resource-based (như Bucket Policy) được gắn vào tài nguyên và xác định ai có thể làm gì với nó. Identity-based (IAM Policy) được gắn vào người dùng và xác định họ có thể làm gì với các tài nguyên khác.

### Scenario
Một công ty phát hiện dữ liệu nhạy cảm trong S3 bị rò rỉ. Khi kiểm tra, IAM Policy của nhân viên vận hành rất chặt chẽ. Bạn sẽ kiểm tra gì tiếp theo?
Trả lời: Cần kiểm tra ngay Bucket Policy để xem có statement nào dùng Principal: * hoặc có cấu hình sai cho phép truy cập từ bên ngoài không. Đồng thời kiểm tra thiết lập "Block Public Access" của Bucket đó.

## 14. References
- Official Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucket-policies.html
- GitHub Repo: N/A
- Spec / RFC: N/A
- Changelog: N/A

## 15. Real-world Code
Tìm kiếm các mẫu Bucket Policy phổ biến cho bảo mật tại:
https://github.com/aws-samples/amazon-s3-safe-bucket-policy-generator

## 16. Community
- Reddit: r/aws
- Stack Overflow: Search with tag [amazon-s3] and [amazon-bucket-policy]
- Blog: AWS Security Blog
- Talk: AWS re:Invent sessions on S3 Security Deep Dive
