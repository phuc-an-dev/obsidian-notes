---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/http"
related:
  - "[[s3]]"
  - "[[iam]]"
---

## 1. What
Pre-signed URL là một URL được tạo ra bởi chủ sở hữu tài nguyên S3 bằng cách sử dụng thông tin xác thực bảo mật của họ (IAM User/Role) để cấp quyền truy cập tạm thời cho người khác. URL này chứa các tham số truy vấn (query params) mang thông tin chữ ký xác thực.

## 2. Why
Thông thường, các đối tượng trong S3 được để ở chế độ riêng tư (Private) để bảo mật. Tuy nhiên, đôi khi bạn cần cho phép người dùng cuối (người không có tài khoản AWS) tải xuống một file hoặc upload một file lên một bucket riêng tư mà không cần làm cho cả bucket đó trở thành Public. Pre-signed URL giải quyết vấn đề này bằng cách cấp quyền có thời hạn.

## 3. Mental Model
Hãy tưởng tượng Pre-signed URL như một chiếc **vé xem phim**.
- Bạn (App Server) là người bán vé có thẩm quyền.
- Khán giả (User) không có chìa khóa rạp phim.
- Bạn đưa cho khán giả một chiếc vé (URL). Chiếc vé này chỉ có giá trị cho một bộ phim cụ thể (Object), tại một rạp cụ thể (Bucket) và sẽ hết hạn sau một khoảng thời gian nhất định (Expiration). Khán giả chỉ cần cầm vé đó là có thể vào rạp mà không cần thẻ ID nhân viên.

## 4. Where it fits
User -> Request URL -> **App Server (Signer)** -> AWS SDK -> **Pre-signed URL** -> User -> S3.

## 5. When to use
- Cho phép người dùng tải xuống tài liệu cá nhân (hóa đơn, hồ sơ bệnh án) từ bucket riêng tư.
- Cho phép người dùng upload ảnh trực tiếp lên S3 từ trình duyệt (giúp giảm tải cho server của bạn).
- Chia sẻ dữ liệu tạm thời cho bên thứ ba qua email hoặc tin nhắn.

## 6. When NOT to use
- Đối với các tài nguyên tĩnh công khai (như logo website, CSS) -> Nên dùng S3 Public hoặc CloudFront.
- Khi cần chia sẻ dữ liệu cho hàng triệu người dùng cùng lúc trong thời gian dài -> Nên dùng **CloudFront Signed URLs**.
- Khi quyền truy cập là vĩnh viễn.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Cực kỳ an toàn (không lộ Secret Key) | Khó thu hồi (revoke) trước khi hết hạn |
| Giảm tải băng thông cho server | URL có thể khá dài và trông "xấu" |
| Dễ triển khai với AWS SDK | Phụ thuộc vào tính chính xác của đồng hồ server (clock skew) |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| CloudFront Signed URLs | Tốt hơn cho quy mô lớn, hỗ trợ giới hạn IP. |
| IAM User Keys | Quá nguy hiểm khi giao key cho người dùng cuối. |
| S3 Public | Không có tính bảo mật, ai cũng có thể truy cập. |

## 9. How
Tạo Pre-signed URL để tải xuống (GET) bằng AWS SDK v3 (Node.js):
```javascript
import { S3Client, GetObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

const client = new S3Client({ region: "us-east-1" });

async function getUrl() {
  const command = new GetObjectCommand({
    Bucket: "my-private-bucket",
    Key: "report.pdf",
  });
  
  // URL hết hạn sau 15 phút (900 giây)
  const url = await getSignedUrl(client, command, { expiresIn: 900 });
  console.log("Presigned URL:", url);
}
```

## 10. Production concerns
### Scaling
Việc sinh URL tốn rất ít tài nguyên và không yêu cầu gọi API lên AWS (nó được tính toán cục bộ dựa trên mã hóa). Do đó, nó scale rất tốt.

### Failure
Nếu thông tin xác thực của người tạo URL (IAM Role) bị thay đổi hoặc bị xóa, URL đó sẽ lập tức trở nên vô hiệu ngay cả khi chưa hết hạn.

### Monitoring
Theo dõi các yêu cầu truy cập S3 thông qua S3 Server Access Logs hoặc CloudTrail để phát hiện các hành vi truy cập bất thường từ các URL đã ký.

## 11. Common mistakes
- Mistake: Thiết lập thời gian hết hạn quá dài (ví dụ: 1 tuần).
  Fix: Chỉ nên để thời gian đủ dùng (ví dụ: 15-30 phút) để giảm thiểu rủi ro nếu URL bị lộ.

- Mistake: Người tạo URL (Signer) có quá nhiều quyền hạn.
  Fix: Sử dụng một IAM Role với quyền hạn tối thiểu (chỉ `s3:GetObject` trên đúng file đó).

## 12. Sample project
Xây dựng chức năng "Tải hóa đơn PDF": Khi người dùng nhấn nút "Tải về", Backend kiểm tra session, nếu hợp lệ thì sinh một Pre-signed URL và redirect người dùng đến URL đó để tải trực tiếp từ S3.

## 13. Interview
### Core Q&A
1. Q: Pre-signed URL có an toàn không nếu nó bị rò rỉ?
   A: Nó chỉ an toàn trong phạm vi thời gian hết hạn. Nếu bị lộ, bất kỳ ai có URL đều có thể truy cập. Vì vậy, thời gian hết hạn ngắn và sử dụng HTTPS là bắt buộc.

### Scenario
1. Q: Bạn làm thế nào để người dùng upload file 1GB lên S3 mà không đi qua Backend của bạn?
   A: Tôi sẽ sinh một Pre-signed URL với phương thức **PUT**. Trình duyệt của người dùng sẽ gửi file trực tiếp lên S3 bằng URL đó. Backend của tôi chỉ đóng vai trò "người ký vé".

## 14. References
- AWS Docs: https://docs.aws.amazon.com/AmazonS3/latest/userguide/ShareObjectPreSignedURL.html
- SDK v3 Guide: https://aws.amazon.com/blogs/developer/generate-presigned-urls-using-aws-sdk-for-javascript-v3/

## 15. Real-world Code
Học cách tích hợp Pre-signed URL với các thư viện frontend như `Uppy` hoặc `Dropzone` để thực hiện upload file mượt mà.

## 16. Community
- Blog: "S3 Presigned URLs: Security Best Practices" - Cloudonaut.
- Discussion: So sánh Pre-signed URL vs CloudFront Signed URL trên Reddit.
