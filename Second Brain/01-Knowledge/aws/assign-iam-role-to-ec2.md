---
created: 2026-05-04
tags:
  - "#type/tutorial"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/iam"
related:
  - "[[iam-role-for-ec2]]"
  - "[[ec2]]"
---

## 1. What
**Gán Role vào EC2 Instance** là hành động liên kết một **IAM Role** (thông qua **Instance Profile**) với một máy ảo EC2 cụ thể. Thao tác này cho phép các tiến trình bên trong máy ảo đó có thể thực hiện các yêu cầu API đến AWS bằng danh tính và quyền hạn của Role đã gán.

## 2. Why
Một IAM Role khi được tạo ra chỉ là một tập hợp các quy tắc nằm trong hệ thống IAM. Để Role đó có hiệu lực trên một máy chủ cụ thể, bạn phải thực hiện việc "gán" (attach). Nếu không gán, EC2 Instance sẽ không có danh tính AWS và mọi yêu cầu truy cập tài nguyên (như S3, DynamoDB) sẽ bị từ chối.

## 3. Mental Model
Hãy tưởng tượng IAM Role như một cái **"Chứng minh thư (ID Card)"**. Việc tạo Role giống như việc in xong chiếc thẻ. Còn việc gán Role vào EC2 giống như hành động **"Đeo thẻ vào cổ nhân viên"**. Chỉ khi người nhân viên (EC2) đeo chiếc thẻ đó lên người, họ mới có thể đi qua các cửa bảo mật của công ty.

## 4. Where it fits
Nó là bước cuối cùng trong cấu hình bảo mật EC2:
`Create Policy -> Create Role -> Attach Policy to Role -> Attach Role to EC2 (Instance Profile)`

## 5. When to use
- Ngay khi khởi tạo một EC2 Instance mới (trong bước Configure Instance).
- Khi một Instance đang chạy cần bổ sung quyền hạn để tương tác với các dịch vụ AWS khác.
- Khi muốn thay đổi quyền hạn của một nhóm server bằng cách thay đổi Role chung của chúng.

## 6. When NOT to use
- Khi bạn muốn kiểm soát quyền ở mức độ người dùng đăng nhập (User level) thay vì mức độ máy chủ (Server level).
- Khi Instance không cần truy cập vào bất kỳ tài nguyên AWS nào.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Có thể thay đổi Role cho Instance đang chạy mà không cần khởi động lại. | Một Instance chỉ có thể gắn duy nhất 1 Role tại một thời điểm. |
| Quản lý quyền hạn cực kỳ linh hoạt. | Cần chú ý đến phiên bản Metadata Service (IMDSv1 vs IMDSv2). |
| Áp dụng được cho cả Auto Scaling Group. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| **Instance Metadata Service** | Đây là cơ chế bên dưới, không phải là lựa chọn thay thế. |
| **Credentials File** | Lưu `~/.aws/credentials` trên disk (Kém an toàn). |
| **Environment Variables** | Truyền `AWS_ACCESS_KEY_ID` (Kém an toàn). |

## 9. How
Cách gán Role vào EC2 qua Console:
1. Mở **EC2 Console** -> Chọn **Instances**.
2. Chọn Instance cần gán -> Nhấn nút **Actions**.
3. Chọn **Security** -> **Modify IAM role**.
4. Chọn Role mong muốn từ danh sách thả xuống.
5. Nhấn **Update IAM role**.

Cách kiểm tra trên server:
```bash
# Kiểm tra xem Role đã được nhận diện chưa
curl http://169.254.169.254/latest/meta-data/iam/info
```

## 10. Production concerns
### Scaling
Trong **Auto Scaling Group (ASG)**, bạn không gán Role cho từng Instance mà gán vào **Launch Template** hoặc **Launch Configuration**. Khi ASG tạo máy mới, Role sẽ tự động được gán.

### Failure
Nếu gán nhầm Role, Instance có thể bị mất quyền truy cập hoặc có quá nhiều quyền. Thao tác gán thường có hiệu lực gần như ngay lập tức (vài giây).

### Monitoring
Kiểm tra log của ứng dụng để xem các lỗi `403 Forbidden`. Sử dụng **AWS Access Analyzer** để xem Role nào đang được sử dụng thực tế.

## 11. Common mistakes
- **Mistake**: Cố gắng gán nhiều Role cho 1 Instance.
  **Fix**: Kết hợp các chính sách (Policies) vào trong một Role duy nhất.

- **Mistake**: Gán Role nhưng ứng dụng vẫn dùng credentials cũ trong file `.aws/credentials`.
  **Fix**: Xóa bỏ các file credentials thủ công trên server để SDK tự động tìm đến Metadata Service.

## 12. Sample project
Sử dụng AWS CLI trên EC2 để liệt kê các bucket S3:
1. Gán Role có quyền `S3ReadOnly` vào EC2.
2. SSH vào EC2 và chạy `aws s3 ls`. 
**Kết quả**: Bạn có thể xem danh sách mà không cần chạy `aws configure`.

## 13. Interview
### Core Q&A
1. **Q**: Có thể gán Role cho EC2 đang chạy (Running) không?
   **A**: Có, từ năm 2017 AWS đã cho phép gán hoặc thay đổi Role cho Instance đang chạy mà không cần reboot.
2. **Q**: Điều gì xảy ra bên dưới khi ta gán Role?
   **A**: AWS sẽ tạo một **Instance Profile** liên kết với Role đó và đưa thông tin credential tạm thời vào dịch vụ Metadata (IMDS) của máy ảo.
3. **Q**: Làm sao để gỡ Role khỏi EC2?
   **A**: Vào **Modify IAM role** và chọn **No IAM role**.

### Scenario
**Tình huống**: Bạn vừa gán Role mới cho EC2 nhưng lệnh `aws s3 ls` vẫn báo lỗi "Expired Token". Bạn làm gì?
**Trả lời**: Token của Role có thời gian tồn tại. Thông thường SDK sẽ tự động làm mới. Tôi sẽ kiểm tra xem thời gian trên máy chủ có bị lệch không (NTP sync) hoặc thử khởi động lại ứng dụng để nó xóa cache token cũ.

## 14. References
- Official Docs: [Modify IAM role for EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html#replace-iam-role)

## 15. Real-world Code
- Lệnh CLI để gán role: `aws ec2 associate-iam-instance-profile --instance-id i-1234567890abcdef0 --iam-instance-profile Name=MyRoleName`

## 16. Community
- Reddit: r/aws
- Stack Overflow: Tag [aws-iam-role], [ec2-instance-profile]
