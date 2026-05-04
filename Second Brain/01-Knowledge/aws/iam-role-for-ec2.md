---
created: 2026-05-04
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/iam"
related:
  - "[[iam-policy]]"
  - "[[ec2]]"
---

## 1. What
**IAM Role cho EC2** là một thực thể (identity) trong AWS dùng để cấp quyền hạn cho các ứng dụng chạy trên EC2 Instance mà không cần sử dụng Access Key/Secret Key. Nó cho phép Instance tự động lấy các credentials tạm thời (temporary credentials) thông qua **Instance Profile**.

## 2. Why
Việc lưu trữ thủ công Access Key/Secret Key bên trong code hoặc tệp cấu hình trên server là cực kỳ nguy hiểm. Nếu server bị tấn công, kẻ xấu sẽ có toàn quyền truy cập AWS của bạn vĩnh viễn. IAM Role giải quyết vấn đề này bằng cách sử dụng các key có thời hạn ngắn (thường là vài giờ) và tự động xoay vòng (auto-rotate).

## 3. Mental Model
Hãy tưởng tượng IAM Role như một chiếc **"Thẻ chìa khóa khách sạn (Guest Keycard)"**. Thay vì đưa cho nhân viên một chiếc chìa khóa vạn năng bằng sắt (Permanent Key), bạn đưa cho họ một chiếc thẻ từ. Chiếc thẻ này chỉ có tác dụng mở những phòng nhất định và sẽ tự động hết hạn sau một khoảng thời gian. Nếu nhân viên làm mất thẻ, rủi ro sẽ thấp hơn nhiều vì thẻ sẽ sớm vô hiệu.

## 4. Where it fits
Nó đóng vai trò là cầu nối bảo mật giữa hạ tầng và tài nguyên:
`EC2 Instance -> Instance Profile -> IAM Role -> STS (Security Token Service) -> AWS Resources`

## 5. When to use
- Khi ứng dụng chạy trên EC2 cần truy cập S3 để lưu file.
- Khi cần gửi log từ EC2 lên CloudWatch Logs.
- Khi ứng dụng cần tương tác với DynamoDB hoặc các AWS services khác.

## 6. When NOT to use
- Khi ứng dụng chạy bên ngoài AWS (như local machine hoặc On-premise) - trường hợp này nên dùng IAM User với Access Key hoặc IAM Roles Anywhere.
- Khi các tác vụ không yêu cầu quyền truy cập vào AWS Resources.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Bảo mật cực cao, không lo lộ Permanent Keys. | Cần hiểu về cơ chế Instance Profile. |
| Tự động quản lý vòng đời của credential. | Khó debug hơn một chút khi quyền hạn bị từ chối. |
| Quản lý tập trung tại một nơi (IAM Console). | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| **IAM User Keys** | Dễ cấu hình nhưng cực kỳ kém an toàn, khó quản lý việc xoay vòng key. |
| **IAM Roles Anywhere** | Dùng chứng chỉ X.509 để cấp role cho server ngoài AWS. |
| **Hardcoded Credentials** | Tuyệt đối không sử dụng trong môi trường sản xuất. |

## 9. How
Quy trình tạo Role cho EC2:
1. Vào **IAM Console** -> **Roles** -> **Create role**.
2. Chọn **AWS service** làm Trusted entity và chọn **EC2**.
3. Gắn các **IAM Policies** cần thiết (ví dụ: `AmazonS3ReadOnlyAccess`).
4. Đặt tên cho Role (ví dụ: `EC2-S3-ReadOnly-Role`) và nhấn **Create**.

## 10. Production concerns
### Scaling
Một Role có thể được gắn cho hàng ngàn EC2 Instance cùng lúc. Điều này giúp quản lý quyền hạn ở quy mô lớn trở nên vô cùng đơn giản.

### Failure
Nếu Role không có đủ quyền, ứng dụng sẽ nhận lỗi `AccessDenied`. Cần kiểm tra lại **Trust Relationship** của Role để đảm bảo dịch vụ `ec2.amazonaws.com` có quyền assume role đó.

### Monitoring
Sử dụng **CloudTrail** để theo dõi các hành động mà EC2 Instance thực hiện thông qua Role.

## 11. Common mistakes
- **Mistake**: Tạo Role nhưng quên không cấu hình **Trust Relationship** cho EC2.
  **Fix**: Đảm bảo chính sách tin cậy (Trust Policy) cho phép hành động `sts:AssumeRole` từ `ec2.amazonaws.com`.

- **Mistake**: Gắn quá nhiều quyền (AdministratorAccess) cho EC2 Role.
  **Fix**: Áp dụng nguyên tắc Least Privilege, chỉ cấp những quyền thực sự cần thiết.

## 12. Sample project
Xây dựng một hệ thống sao lưu database tự động chạy trên EC2. Script sao lưu sẽ đẩy tệp tin lên S3. Thay vì cấu hình key, hãy tạo một Role có quyền `s3:PutObject` và gắn vào Instance đó.

## 13. Interview
### Core Q&A
1. **Q**: Tại sao dùng IAM Role cho EC2 lại an toàn hơn IAM User?
   **A**: Vì Role sử dụng temporary credentials (key tạm thời) và tự động xoay vòng bởi AWS, loại bỏ hoàn toàn rủi ro lộ Permanent Keys.
2. **Q**: Instance Profile là gì?
   **A**: Là một "thùng chứa" (container) cho IAM Role để EC2 có thể hiểu và sử dụng Role đó tại thời điểm khởi tạo.
3. **Q**: Làm thế nào để ứng dụng lấy được credential từ Role?
   **A**: AWS SDK tự động tìm kiếm trong **Instance Metadata Service (IMDS)** tại địa chỉ `169.254.169.254`.

### Scenario
**Tình huống**: Bạn đã gắn Role vào EC2 nhưng ứng dụng vẫn báo không có quyền truy cập S3. Bạn kiểm tra gì?
**Trả lời**: Tôi sẽ kiểm tra: 1. Role đã được gắn đúng Policy chưa? 2. Instance Profile đã được gắn vào EC2 chưa? 3. Ứng dụng có đang dùng SDK của AWS không (vì SDK tự động xử lý việc lấy token)? 4. Có proxy nào đang chặn truy cập tới địa chỉ metadata `169.254.169.254` không?

## 14. References
- Official Docs: [IAM Roles for EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html)

## 15. Real-world Code
- Các Framework như Spring Boot (AWS Cloud) hoặc AWS SDK cho Java/NodeJS đều tự động hỗ trợ lấy credential từ EC2 Role.

## 16. Community
- Reddit: r/aws
- Stack Overflow: Tag [amazon-ec2], [aws-iam]
