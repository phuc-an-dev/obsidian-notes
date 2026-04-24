---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/http"
related:
  - "[[cloudfront]]"
  - "[[ec2]]"
---

## 1. What
AWS Route 53 là một dịch vụ hệ thống phân giải tên miền (DNS) có độ sẵn sàng cao và khả năng mở rộng cực tốt. Nó giúp chuyển đổi các tên miền thân thiện (ví dụ: `www.example.com`) thành các địa chỉ IP số (ví dụ: `192.0.2.1`) mà máy tính sử dụng để kết nối với nhau.

## 2. Why
Nếu không có DNS, người dùng phải nhớ những dãy số IP khô khan để truy cập web. Route 53 ra đời không chỉ để giải quyết việc phân giải tên miền mà còn cung cấp các tính năng nâng cao như kiểm tra sức khỏe máy chủ (Health Checks) và các chính sách điều phối lưu lượng (Routing Policies) mà các dịch vụ DNS truyền thống không có.

## 3. Mental Model
Hãy tưởng tượng Route 53 như một **Tổng đài viên thông minh**.
- Khi khách hàng gọi đến tên một công ty (Tên miền), tổng đài viên sẽ tra cứu danh bạ và nối máy đến đúng số điện thoại (IP).
- Nếu văn phòng chính đang bận hoặc bị cháy (Server hỏng), tổng đài viên đủ thông minh để nối máy sang văn phòng dự phòng ở thành phố khác (Failover).
- Thậm chí, tổng đài viên có thể nối máy đến văn phòng gần nhất với vị trí của khách hàng để cuộc gọi rõ nét hơn (Latency routing).

## 4. Where it fits
User -> Browser -> **AWS Route 53** -> IP Address/Endpoint -> CloudFront/ALB/S3.

## 5. When to use
- Đăng ký và quản lý tên miền (Domain Registration).
- Điều phối traffic giữa các Region khác nhau trên toàn cầu.
- Thiết lập hệ thống dự phòng (Disaster Recovery) với chính sách Failover.
- Kiểm tra trạng thái hoạt động của các server (Health Checks).

## 6. When NOT to use
- Khi bạn chỉ muốn quản lý DNS cục bộ bên trong một mạng nội bộ không kết nối internet (mặc dù vẫn dùng được Private Hosted Zones nhưng có thể quá phức tạp so với nhu cầu).
- Khi bạn đã hài lòng với trình quản lý DNS miễn phí của nhà đăng ký tên miền và không cần các tính năng định tuyến nâng cao.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Độ tin cậy 100% SLA | Chi phí cao hơn so với DNS miễn phí |
| Tích hợp mượt mà với AWS (Alias records) | Cấu hình Routing Policies phức tạp |
| Hỗ trợ Health Checks và Failover tự động | Đôi khi bị giới hạn bởi thời gian cache (TTL) của ISP |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Cloudflare DNS | Rất nhanh, miễn phí cơ bản, bảo mật tốt. |
| Google Cloud DNS | Tương đương, mạnh về hiệu năng. |
| GoDaddy/Namecheap DNS | Đơn giản, đi kèm khi mua tên miền nhưng ít tính năng. |

## 9. How
Tạo một Record đơn giản bằng AWS CLI:
```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890 \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.example.com",
        "Type": "A",
        "TTL": 300,
        "ResourceRecords": [{"Value": "1.2.3.4"}]
      }
    }]
  }'
```

## 10. Production concerns
### Scaling
Route 53 tự động scale để xử lý hàng tỷ truy vấn. Tuy nhiên, cần lưu ý cấu hình **TTL (Time To Live)** hợp lý: TTL thấp giúp cập nhật nhanh nhưng tăng số truy vấn (tốn tiền), TTL cao thì ngược lại.

### Failure
Sử dụng **Failover Routing Policy** kết hợp với Health Checks. Nếu server chính fail, Route 53 sẽ tự động trỏ về server dự phòng hoặc trang thông báo bảo trì.

### Monitoring
Theo dõi số lượng truy vấn và trạng thái Health Checks trong CloudWatch.

## 11. Common mistakes
- Mistake: Dùng record `CNAME` cho root domain (ví dụ: `example.com`).
  Fix: Luôn dùng record **Alias (A)** của Route 53 để trỏ root domain về tài nguyên AWS (S3, ALB, CloudFront).

- Mistake: Đặt TTL quá cao cho các record thường xuyên thay đổi IP.
  Fix: Giảm TTL xuống còn 60-300 giây khi đang trong quá trình chuyển đổi hệ thống.

## 12. Sample project
Thiết lập hệ thống Global: Người dùng ở Châu Á được trỏ về Tokyo Region, người dùng ở Châu Âu được trỏ về Frankfurt Region dựa trên **Geolocation Routing Policy**.

## 13. Interview
### Core Q&A
1. Q: "Alias Record" khác gì với "CNAME Record"?
   A: Alias Record là tính năng riêng của AWS, cho phép trỏ tên miền về tài nguyên AWS mà không tốn phí truy vấn và có thể dùng cho root domain. CNAME thì tốn phí truy vấn thêm và không được dùng cho root domain.

### Scenario
1. Q: Làm thế nào để triển khai Blue/Green deployment ở tầng DNS?
   A: Tôi sẽ dùng **Weighted Routing Policy**. Ban đầu cho 100% traffic vào bản Blue, sau đó chuyển dần 10%, 20%... sang bản Green cho đến khi hoàn tất.

## 14. References
- Official Docs: https://aws.amazon.com/route53/
- Routing Policies: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html

## 15. Real-world Code
Sử dụng Terraform để quản lý Hosted Zones và Records nhằm đảm bảo tính đồng nhất giữa các môi trường.

## 16. Community
- YouTube: "Route 53 Deep Dive" - AWS re:Invent.
- Twitter: #Route53 #AWS.
