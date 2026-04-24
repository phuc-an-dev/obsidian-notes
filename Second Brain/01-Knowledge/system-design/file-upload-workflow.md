---
created: 2026-04-24
tags:
  - "#type/pattern"
  - "#status/draft"
  - "#lang/system-design"
  - "#topic/http"
related:
  - "[[s3-presigned-url]]"
  - "[[s3]]"
  - "[[cloudfront]]"
---

## 1. What
Đây là quy trình tải tệp tin (upload) từ Client trực tiếp lên S3 thông qua cơ chế Pre-signed URL, sau đó sử dụng CloudFront để phục vụ các tệp tin đó cho người dùng khác. Quy trình gồm: Client -> Backend -> Pre-signed URL -> S3 -> CloudFront.

## 2. Why
Việc tải tệp tin trực tiếp qua Backend sẽ gây tốn tài nguyên server (CPU, RAM, Băng thông), làm chậm hệ thống khi có nhiều người upload file lớn cùng lúc. Kỹ thuật này đẩy việc truyền tải tệp tin sang cho hạ tầng của AWS xử lý, Backend chỉ đóng vai trò kiểm soát quyền hạn.

## 3. Mental Model
Hãy tưởng tượng quy trình như việc bạn gửi đồ tại một tủ đồ công cộng:
1. Bạn đến quầy quản lý (Backend) xin phép gửi đồ.
2. Quản lý kiểm tra bạn là ai và đưa cho bạn một cái chìa khóa tạm thời (Pre-signed URL) có giá trị trong 15 phút.
3. Bạn tự mang đồ đến đúng ngăn tủ ghi trên chìa khóa (S3) để cất.
4. Khi bạn bè muốn xem món đồ đó, họ chỉ cần nhìn qua một tấm kính trưng bày được đặt ở ngay đầu phố (CloudFront) thay vì phải đi vào tận kho.

## 4. Where it fits
1. **Request**: Client yêu cầu upload -> Backend.
2. **Authorize**: Backend xác thực, tạo Pre-signed URL -> Client.
3. **Upload**: Client gửi file trực tiếp -> S3 bằng URL đã nhận.
4. **Serve**: User khác truy cập file -> CloudFront -> S3.

## 5. When to use
- Các ứng dụng cho phép người dùng upload ảnh, video, tài liệu.
- Khi cần tối ưu hóa hiệu năng cho Backend server.
- Khi cần phục vụ file tĩnh cho hàng triệu người dùng toàn cầu với độ trễ thấp.

## 6. When NOT to use
- File cực nhỏ (vài KB) mà logic xử lý sau khi upload phức tạp (có thể upload qua backend để xử lý đồng bộ).
- Các hệ thống nội bộ đơn giản, traffic thấp.
- Khi tệp tin cần được xử lý/chỉnh sửa ngay lập tức tại Backend trước khi lưu.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Giảm tải cực lớn cho Backend | Phức tạp hơn ở phía Frontend (xử lý PUT request) |
| Tận dụng băng thông cực lớn của AWS S3 | Khó track trạng thái upload trực tiếp từ Backend |
| Bảo mật cao hơn nhờ URL có thời hạn | Cần cấu hình CORS trên S3 Bucket |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Multipart Upload qua Backend | Đơn giản cho FE, nhưng BE rất dễ bị treo khi file lớn. |
| AWS Amplify Storage | Dễ dùng nhưng bị bó buộc vào framework Amplify. |

## 9. How
Quy trình thực hiện:
1. **Backend (Node/Spring)**: Dùng AWS SDK tạo Pre-signed URL (Method: PUT).
2. **Frontend (Client)**: 
   ```javascript
   fetch(presignedUrl, { method: 'PUT', body: fileFile });
   ```
3. **S3**: Cấu hình CORS cho phép domain của Frontend thực hiện PUT request.
4. **CloudFront**: Cấu hình OAC trỏ đến S3 bucket để serve file qua HTTPS.

## 10. Production concerns
### Scaling
Hệ thống này scale gần như vô hạn vì S3 và CloudFront là các dịch vụ serverless được AWS tối ưu hóa cực tốt.

### Failure
Nếu upload thất bại ở giữa chừng, Client nên có cơ chế Retry hoặc sử dụng **Multipart Upload** cho các file > 100MB để không phải upload lại từ đầu.

### Monitoring
Sử dụng S3 Event Notifications để gọi Lambda function lưu thông tin file vào Database ngay sau khi upload thành công (ObjectCreated event).

## 11. Common mistakes
- Mistake: Quên cấu hình CORS trên S3.
  Fix: Cấu hình CORS cho phép `AllowedHeaders`, `AllowedMethods` (PUT, POST) từ Origin của Frontend.

- Mistake: Dùng Pre-signed URL để serve file (tải xuống).
  Fix: Nên dùng CloudFront URL để serve file nhằm tiết kiệm chi phí băng thông và tăng tốc độ.

## 12. Sample project
Ứng dụng chia sẻ video ngắn: Người dùng chọn video, ứng dụng lấy Pre-signed URL, tải video lên S3. S3 gọi Lambda để nén video, sau đó video nén được phục vụ qua CloudFront cho người xem.

## 13. Interview
### Core Q&A
1. Q: Tại sao phải dùng CloudFront để serve file thay vì dùng trực tiếp S3 URL?
   A: S3 URL không tối ưu về vị trí địa lý (latency cao) và chi phí Data Transfer Out của S3 đắt hơn so với CloudFront. Ngoài ra CloudFront hỗ trợ HTTPS với domain tùy chỉnh dễ dàng hơn.

### Scenario
1. Q: Làm thế nào để Backend biết được người dùng đã upload file thành công để cập nhật vào Database?
   A: Cách tốt nhất là dùng **S3 Event Notifications**. Khi file được lưu vào S3, AWS sẽ gửi một message đến SQS/Lambda. Lambda sẽ thực hiện logic cập nhật database. Điều này đảm bảo tính nhất quán ngay cả khi Client bị mất mạng sau khi upload.

## 14. References
- S3 Presigned URL Guide: https://docs.aws.amazon.com/AmazonS3/latest/userguide/PresignedUrlUploadObject.html
- CloudFront with S3: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistS3AndCustomOrigins.html

## 15. Real-world Code
Tìm kiếm các pattern "Serverless File Upload" trên GitHub để xem cách kết hợp Lambda và S3 Event.

## 16. Community
- YouTube: "AWS S3 Upload with Pre-signed URLs".
- Blog: "Scaling File Uploads on AWS".
