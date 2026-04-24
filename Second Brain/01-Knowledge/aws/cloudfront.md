---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/performance"
related:
  - "[[s3]]"
  - "[[ec2]]"
---

## 1. What
AWS CloudFront là một dịch vụ mạng phân phối nội dung (Content Delivery Network - CDN) tốc độ cao, giúp phân phối dữ liệu, video, ứng dụng và API cho người dùng trên toàn cầu với độ trễ thấp và tốc độ truyền cao. Nó hoạt động thông qua một mạng lưới các điểm hiện diện (Edge Locations) trên toàn thế giới.

## 2. Why
Nếu bạn đặt máy chủ tại Mỹ nhưng người dùng ở Việt Nam truy cập, dữ liệu phải đi qua quãng đường địa lý xa xôi, gây ra độ trễ (latency) lớn. CloudFront ra đời để lưu trữ bản sao của dữ liệu tại các vị trí gần người dùng nhất, giúp giảm thời gian tải trang và giảm tải cho máy chủ gốc (Origin).

## 3. Mental Model
Hãy tưởng tượng máy chủ gốc của bạn là một nhà máy sản xuất bánh kẹo lớn ở một thành phố duy nhất. CloudFront giống như một hệ thống các cửa hàng tiện lợi nhượng quyền đặt ở mọi ngõ ngách. Thay vì mọi khách hàng phải lái xe hàng trăm cây số đến nhà máy để mua kẹo, họ chỉ cần đi bộ ra cửa hàng tiện lợi gần nhất. Nếu cửa hàng hết hàng, họ mới gọi về nhà máy để nhập thêm (Cache Miss).

## 4. Where it fits
User -> **Edge Location (CloudFront)** -> Regional Edge Cache -> Origin Server (S3 / ALB / EC2 / External).

## 5. When to use
- Phân phối nội dung tĩnh như hình ảnh, CSS, JavaScript.
- Phân phối video trực tuyến (Streaming) qua giao thức HLS/DASH.
- Bảo mật ứng dụng web bằng cách tích hợp với AWS WAF và Shield.
- Tăng tốc độ truy cập API động.
- Chạy mã logic nhẹ tại vùng biên (CloudFront Functions / Lambda@Edge).

## 6. When NOT to use
- Ứng dụng nội bộ với số lượng người dùng rất ít và tập trung ở một vùng địa lý duy nhất.
- Khi dữ liệu thay đổi quá nhanh và yêu cầu tính nhất quán tuyệt đối (mặc dù vẫn dùng được nhưng cấu hình cache sẽ rất phức tạp).
- Khi chi phí truyền tải dữ liệu (Data Transfer Out) vượt quá ngân sách cho phép.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Cải thiện đáng kể trải nghiệm người dùng (độ trễ thấp) | Phức tạp trong việc quản lý bộ nhớ đệm (Cache Invalidation) |
| Bảo mật mạnh mẽ (DDoS protection, SSL/TLS) | Chi phí tăng thêm dựa trên lượng dữ liệu truyền tải |
| Giảm tải cực lớn cho máy chủ gốc | Khó debug các vấn đề liên quan đến Header và Cookies |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Cloudflare | Đối thủ lớn nhất, mạnh về bảo mật và gói miễn phí tốt. |
| Akamai | Giải pháp cho doanh nghiệp cực lớn, mạng lưới rộng nhất. |
| Fastly | Cho phép tùy chỉnh logic tại vùng biên cực mạnh (VCL). |

## 9. How
Cấu hình cơ bản CloudFront với S3 Origin:
1. Tạo một S3 Bucket và upload ảnh.
2. Tạo CloudFront Distribution, chọn S3 Bucket làm Origin.
3. Cấu hình **Origin Access Control (OAC)** để đảm bảo người dùng chỉ có thể truy cập ảnh qua CloudFront, không được truy cập trực tiếp URL của S3.
4. Sử dụng Domain được cấp (ví dụ: `d123.cloudfront.net/logo.png`).

## 10. Production concerns
### Scaling
CloudFront tự động scale để xử lý hàng triệu request mỗi giây mà không cần can thiệp thủ công.

### Failure
Nếu máy chủ gốc (Origin) gặp sự cố, CloudFront có thể phục vụ nội dung cũ từ cache (Stale content) hoặc chuyển hướng sang một máy chủ dự phòng (Origin Failover).

### Monitoring
Theo dõi tỉ lệ Cache Hit Rate, tỷ lệ lỗi 4xx/5xx thông qua CloudFront console và CloudWatch Logs.

## 11. Common mistakes
- Mistake: Không xóa cache (Invalidation) sau khi cập nhật file trên máy chủ gốc.
  Fix: Luôn thực hiện `Create Invalidation` cho path `/index.html` hoặc dùng versioning cho file (ví dụ: `style.v2.css`).

- Mistake: Để S3 Bucket ở chế độ Public để CloudFront truy cập được.
  Fix: Sử dụng OAC (Origin Access Control) để giữ S3 ở chế độ Private, tăng tính bảo mật.

## 12. Sample project
Triển khai một trang web React tĩnh (SPA) lên S3, sử dụng CloudFront làm CDN, cấu hình SSL từ ACM (Certificate Manager) để có HTTPS và dùng CloudFront Functions để xử lý chuyển hướng URL (URL Rewrite).

## 13. Interview
### Core Q&A
1. Q: "Edge Location" và "Regional Edge Cache" khác nhau như thế nào?
   A: Edge Location là các điểm nhỏ, gần người dùng nhất để phục vụ cache nhanh. Regional Edge Cache là các điểm lớn hơn, nằm giữa Edge Location và Origin, giúp giữ cache lâu hơn và giảm tải cho Origin khi nhiều Edge Location cùng yêu cầu một tệp.

### Scenario
1. Q: Làm thế nào để phân phối nội dung riêng tư (chỉ dành cho người dùng đã trả phí) qua CloudFront?
   A: Tôi sẽ sử dụng **CloudFront Signed URLs** hoặc **Signed Cookies**. Ứng dụng của tôi sẽ sinh ra một URL có chứa thông tin xác thực và thời gian hết hạn cho người dùng. CloudFront sẽ kiểm tra chữ ký này trước khi phục vụ nội dung.

## 14. References
- Official Docs: https://aws.amazon.com/cloudfront/
- Security with OAC: https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html

## 15. Real-world Code
Sử dụng AWS CDK (TypeScript) để định nghĩa CloudFront Distribution kèm theo WAF Web ACL để bảo vệ ứng dụng khỏi các cuộc tấn công SQL Injection và XSS.

## 16. Community
- YouTube: "CloudFront Deep Dive" - AWS Online Tech Talks.
- Stack Overflow: [amazon-cloudfront] tag.
- Blog: AWS Networking & Content Delivery Blog.
