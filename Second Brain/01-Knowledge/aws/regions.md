---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/performance"
related:
  - "[[iam]]"
---

## 1. What
AWS Regions là các vị trí địa lý vật lý trên toàn thế giới nơi AWS đặt các cụm trung tâm dữ liệu. Mỗi Region bao gồm nhiều cụm trung tâm dữ liệu biệt lập và tách biệt về mặt vật lý được gọi là Availability Zones (AZs).

## 2. Why
Việc chia nhỏ hạ tầng toàn cầu thành các Regions và AZs giúp khách hàng:
1. **Giảm độ trễ**: Đặt ứng dụng gần người dùng nhất.
2. **Tuân thủ pháp lý**: Lưu trữ dữ liệu tại quốc gia cụ thể.
3. **Tính sẵn sàng cao**: Nếu một trung tâm dữ liệu gặp sự cố (thiên tai, mất điện), ứng dụng vẫn chạy ở trung tâm khác.

## 3. Mental Model
Hãy tưởng tượng AWS là một chuỗi siêu thị toàn cầu.
- **Region** là một Thành phố (ví dụ: Hà Nội, TP.HCM). Bạn chọn thành phố gần nhà bạn nhất để đi chợ nhanh hơn.
- **Availability Zone (AZ)** là các chi nhánh siêu thị khác nhau trong cùng thành phố đó. Nếu chi nhánh ở Quận 1 bị mất điện, bạn vẫn có thể sang chi nhánh Quận 3 để mua hàng vì chúng độc lập về nguồn điện và hạ tầng, nhưng vẫn thuộc cùng một hệ thống quản lý.

## 4. Where it fits
AWS Global Infrastructure -> **Regions** -> **Availability Zones** -> Data Centers.

## 5. When to use
- Luôn luôn phải chọn một Region khi bắt đầu sử dụng bất kỳ dịch vụ AWS nào (trừ các dịch vụ Global như IAM, Route 53).
- Chọn nhiều AZs (Multi-AZ) để triển khai các ứng dụng quan trọng cần độ tin cậy cao.

## 6. When NOT to use
- Không nên chọn Region chỉ dựa trên giá rẻ mà bỏ qua độ trễ đối với người dùng cuối.
- Không nên triển khai ứng dụng chỉ trên 1 AZ duy nhất cho môi trường Production.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Khả năng chịu lỗi cực cao (Multi-AZ) | Chi phí truyền tải dữ liệu giữa các AZs (Data Transfer) |
| Tối ưu hóa độ trễ toàn cầu | Phức tạp hơn trong việc đồng bộ hóa dữ liệu |
| Tuân thủ chủ quyền dữ liệu | Một số dịch vụ mới có thể chưa xuất hiện ở tất cả Regions |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| On-premise Data Center | Tự quản lý hoàn toàn nhưng khó mở rộng toàn cầu. |
| Multi-Cloud | Dùng cả AWS, Azure, GCP để tránh rủi ro sập cả một nhà cung cấp. |

## 9. How
Kiểm tra danh sách các AZs trong một Region bằng AWS CLI:
```bash
aws ec2 describe-availability-zones --region us-east-1
```

## 10. Production concerns
### Scaling
Các Region khác nhau có giới hạn tài nguyên (Quotas) khác nhau. Cần kiểm tra kỹ trước khi scale lớn.

### Failure
Sử dụng chiến lược **Multi-Region** cho các hệ thống "Mission Critical". Nếu toàn bộ một vùng địa lý gặp thảm họa, bạn có thể chuyển sang Region khác.

### Monitoring
Theo dõi **AWS Health Dashboard** để biết tình trạng hoạt động của từng Region và AZ.

## 11. Common mistakes
- Mistake: Giả định rằng mọi dịch vụ đều có mặt ở tất cả các Region.
  Fix: Luôn kiểm tra "AWS Region Table" trước khi thiết kế kiến trúc.

- Mistake: Không tính toán chi phí Data Transfer giữa các Region (rất đắt).
  Fix: Thiết kế luồng dữ liệu tối ưu, hạn chế truyền dữ liệu xuyên Region nếu không cần thiết.

## 12. Sample project
Thiết kế hệ thống lưu trữ ảnh: Ảnh gốc lưu tại S3 Region Singapore (gần VN), nhưng được sao lưu tự động (Replication) sang Region Tokyo để dự phòng thảm họa.

## 13. Interview
### Core Q&A
1. Q: "Region" và "Availability Zone" khác nhau như thế nào?
   A: Region là một khu vực địa lý lớn (ví dụ: Singapore). Availability Zone là một hoặc nhiều trung tâm dữ liệu biệt lập bên trong Region đó (ví dụ: ap-southeast-1a, ap-southeast-1b).

### Scenario
1. Q: Khách hàng ở Việt Nam nên chọn Region nào?
   A: Thông thường là Singapore (`ap-southeast-1`) hoặc Hong Kong (`ap-east-1`) vì có độ trễ thấp nhất. Tuy nhiên cần xem xét cả yếu tố giá cả và các dịch vụ cụ thể có hỗ trợ hay không.

## 14. References
- Global Infrastructure: https://aws.amazon.com/about-aws/global-infrastructure/
- Regions and Zones: https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-regions-availability-zones.html

## 15. Real-world Code
Sử dụng biến `region` trong các công cụ IaC (Terraform/CDK) để dễ dàng triển khai cùng một hạ tầng lên nhiều Region khác nhau.

## 16. Community
- YouTube: "AWS Global Infrastructure Explained".
- Twitter: #AWS #CloudComputing.
