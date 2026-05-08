---
created: 2026-05-07
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/security"
related:
  - "[[s3]]"
  - "[[aws-bucket-policy]]"
---

## 1. What
AWS S3 Block Public Access là một tính năng bảo mật cấp cao, cung cấp khả năng kiểm soát tập trung để ngăn chặn việc dữ liệu trong S3 bị truy cập công khai. Nó hoạt động như một lớp bảo vệ bổ sung, ghi đè lên các thiết lập của Bucket Policy và Access Control Lists (ACLs) nếu chúng cố tình hoặc vô ý cho phép truy cập public.

## 2. Why
Rất nhiều vụ rò rỉ dữ liệu lớn xảy ra do quản trị viên cấu hình sai Bucket Policy hoặc ACL, vô tình mở cửa cho toàn bộ Internet. Block Public Access ra đời để giải quyết triệt để vấn đề này bằng cách cho phép bạn "khóa cửa" ở cấp độ toàn bộ tài khoản (Account-level) hoặc từng Bucket cụ thể, đảm bảo rằng ngay cả khi có sai sót trong phân quyền chi tiết, dữ liệu vẫn an toàn.

## 3. Mental Model
Hãy tưởng tượng S3 Bucket là một căn phòng trong một tòa nhà.
- ACL và Bucket Policy là các ổ khóa riêng biệt trên cửa phòng.
- Block Public Access giống như một cái **chốt tổng** (Master Lock) được gắn bên ngoài tòa nhà. Nếu chốt tổng này đang đóng, thì dù bạn có chìa khóa của phòng (Policy cho phép), bạn vẫn không thể đi từ bên ngoài vào trong tòa nhà để đến phòng đó được.

## 4. Where it fits
Request -> S3 Endpoint -> **Block Public Access Check** -> Bucket Policy/ACL Evaluation -> Access Granted/Denied.
Nếu Block Public Access được bật và request là public, AWS sẽ từ chối ngay lập tức mà không cần kiểm tra các policy khác.

## 5. When to use
- Luôn luôn bật ở cấp độ Account cho tất cả các tài khoản AWS của doanh nghiệp để đảm bảo an toàn tối đa.
- Bật cho tất cả các bucket chứa dữ liệu nhạy cảm, mã nguồn, hoặc logs.
- Kể từ tháng 4/2023, AWS mặc định bật tính năng này cho tất cả các bucket mới tạo.

## 6. When NOT to use
- Khi bucket được sử dụng để host một Static Website cần cho phép người dùng truy cập công khai qua Internet.
- Khi bucket dùng để phân phối tài nguyên công khai (ví dụ: kho tải phần mềm, tài liệu hướng dẫn mở).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Ngăn chặn rò rỉ dữ liệu do lỗi cấu hình thủ công. | Có thể làm hỏng (break) các ứng dụng cần truy cập public nếu bật nhầm. |
| Quản lý tập trung ở cấp Account, không cần check từng bucket. | Gây nhầm lẫn cho người mới khi policy đúng nhưng vẫn bị Access Denied. |
| Bảo vệ cả các bucket hiện tại và các bucket sẽ tạo trong tương lai. | N/A |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Bucket Policy "Deny" | Linh hoạt hơn nhưng dễ viết sai logic và khó quản lý quy mô lớn. |
| AWS Organizations SCPs | Có thể dùng để cấm hành động tắt Block Public Access, nhưng không trực tiếp chặn traffic. |

## 9. How
Bật Block Public Access cho một bucket cụ thể bằng AWS CLI:
```bash
aws s3api put-public-access-block \
    --bucket my-secret-data-bucket \
    --public-access-block-configuration "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"
```

## 10. Production concerns
### Scaling
Không ảnh hưởng đến hiệu năng hay khả năng mở rộng của S3.

### Failure
Nếu bạn cần mở public cho một bucket trong một tài khoản đang bật Block Public Access ở Account-level, bạn phải tắt thiết lập ở Account-level trước (hoặc loại trừ bucket đó), điều này có thể gây rủi ro cho các bucket khác nếu không cẩn thận.

### Monitoring
AWS Trusted Advisor và Amazon GuardDuty sẽ cảnh báo nếu tính năng này bị tắt trên các bucket nhạy cảm.

## 11. Common mistakes
- Mistake: Chỉ bật 1 trong 4 tùy chọn mà tưởng rằng đã an toàn tuyệt đối.
  Fix: Hiểu rõ 4 tùy chọn (BlockPublicAcls, IgnorePublicAcls, BlockPublicPolicy, RestrictPublicBuckets) và thường là bật cả 4.

- Mistake: Quên rằng thiết lập ở cấp Account sẽ ghi đè thiết lập ở cấp Bucket.
  Fix: Luôn kiểm tra cả hai nơi khi gặp lỗi Access Denied không rõ nguyên nhân.

## 12. Sample project
Thiết lập một "Security Baseline" cho tài khoản AWS mới: Viết một script Terraform/CloudFormation tự động bật Block Public Access ở cấp Account ngay sau khi tài khoản được khởi tạo.

## 13. Interview
### Core Q&A
1. Q: 4 tùy chọn của Block Public Access khác nhau như thế nào?
   A:
   - `BlockPublicAcls`: Không cho phép gán ACL mới là public.
   - `IgnorePublicAcls`: Bỏ qua tất cả ACL public hiện có.
   - `BlockPublicPolicy`: Không cho phép gán Bucket Policy mới là public.
   - `RestrictPublicBuckets`: Chặn truy cập public và cross-account vào các bucket có policy public.

2. Q: Điều gì xảy ra nếu tôi bật "Ignore Public ACLs" trên một bucket đang có các object public qua ACL?
   A: Toàn bộ các object đó sẽ ngay lập tức không còn truy cập public được nữa, dù ACL vẫn còn đó nhưng bị AWS lờ đi.

### Scenario
Một lập trình viên báo rằng họ đã copy một Bucket Policy chuẩn từ tài liệu AWS để host website nhưng vẫn bị 403 Forbidden. Bạn sẽ kiểm tra gì?
Trả lời: Tôi sẽ kiểm tra xem tính năng "Block Public Access" có đang được bật ở cấp Bucket hoặc cấp Account hay không. Nếu có, nó sẽ chặn Bucket Policy đó.

## 14. References
- Official Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/access-control-block-public-access.html
- AWS Security Blog: https://aws.amazon.com/blogs/aws/amazon-s3-block-public-access-another-layer-of-protection-for-your-accounts-and-buckets/

## 15. Real-world Code
Sử dụng Terraform để bật Block Public Access cho toàn bộ account:
```hcl
resource "aws_s3_account_public_access_block" "example" {
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

## 16. Community
- YouTube: "AWS S3 Security Best Practices"
- Reddit: r/aws - thảo luận về việc AWS mặc định bật tính năng này.
