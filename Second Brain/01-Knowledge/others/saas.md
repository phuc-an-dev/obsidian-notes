---
created: 2026-04-21
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/system-design"
  - "#topic/performance"
related:
  - "[[RESTful API]]"
---

# Software as a Service (SaaS)

## 1. What
SaaS (Software as a Service) là mô hình phân phối phần mềm trong đó ứng dụng được lưu trữ bởi nhà cung cấp dịch vụ bên thứ ba và được người dùng truy cập qua internet, thường thông qua trình duyệt web. Thay vì mua và cài đặt phần mềm cục bộ, người dùng trả phí đăng ký để sử dụng ứng dụng.

## 2. Why
Trước đây, doanh nghiệp phải mua bản quyền phần mềm (on-premise), tự cài đặt, quản lý hạ tầng và cập nhật thủ công. SaaS ra đời để loại bỏ gánh nặng vận hành hạ tầng (IT overhead), giảm chi phí đầu tư ban đầu (CAPEX sang OPEX), và cho phép khả năng mở rộng nhanh chóng.

## 3. Mental Model
Hãy tưởng tượng sự khác biệt giữa việc mua máy phát điện cá nhân và sử dụng điện lưới:
- On-premise: Bạn mua máy phát điện, tự mua nhiên liệu, tự sửa chữa khi hỏng.
- SaaS: Bạn sử dụng điện lưới. Bạn chỉ cần bật công tắc, trả phí hàng tháng dựa trên lượng điện tiêu thụ. Nhà cung cấp lo phần hạ tầng nhà máy điện, bảo trì và phân phối.

## 4. Where it fits
```
[User Browser/App]
       |
[HTTPS / API Gateway]
       |
[SaaS Application Cluster]
       |
[Multi-tenant Database / Infrastructure]
```

## 5. When to use
- Khi cần triển khai giải pháp nhanh chóng mà không muốn đầu tư vào hạ tầng kỹ thuật.
- Khi cần làm việc từ xa, yêu cầu truy cập ứng dụng mọi lúc mọi nơi từ bất kỳ thiết bị nào.
- Khi quy mô người dùng biến động và cần khả năng mở rộng linh hoạt.

## 6. When NOT to use
- Các ứng dụng yêu cầu quyền kiểm soát dữ liệu tuyệt đối hoặc tuân thủ các quy định bảo mật khắt khe (ví dụ: một số ngành ngân hàng, quân sự).
- Khi kết nối internet không ổn định, gây ảnh hưởng đến khả năng làm việc liên tục.
- Khi chi phí đăng ký dài hạn (subscription) trở nên đắt đỏ hơn chi phí sở hữu phần mềm trọn đời.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Dễ tiếp cận, truy cập mọi nơi | Phụ thuộc vào kết nối Internet |
| Không cần cài đặt/bảo trì hạ tầng | Ít quyền kiểm soát dữ liệu/hệ thống |
| Khả năng mở rộng (scalability) cao | Chi phí đăng ký dài hạn có thể cao |
| Luôn cập nhật phiên bản mới nhất | Rủi ro về bảo mật dữ liệu phía nhà cung cấp |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| On-premise | Cài đặt cục bộ, toàn quyền kiểm soát nhưng tốn kém vận hành |
| PaaS | Cung cấp nền tảng phát triển, lập trình viên vẫn phải tự viết ứng dụng |
| IaaS | Chỉ cung cấp hạ tầng máy chủ ảo/network, người dùng lo mọi thứ phía trên |

## 9. How
Triển khai SaaS đòi hỏi kiến trúc **Multi-tenancy** (đa thuê bao):
- Database design: Mỗi tenant một database hoặc một bảng lớn có `tenant_id`.
- Authentication: Cần hệ thống SSO (Single Sign-On) cho doanh nghiệp.
- Billing: Tích hợp hệ thống thanh toán (như Stripe) để quản lý subscription.

## 10. Production concerns
### Multi-tenancy Isolation
Đảm bảo dữ liệu của tenant A không bao giờ bị rò rỉ sang tenant B. Điều này yêu cầu kiểm soát chặt chẽ ở tầng Application (ví dụ: dùng Filter/Interceptor để inject `tenant_id` vào mọi query).

### Scaling
Cần hệ thống tự động co giãn (auto-scaling) để đáp ứng số lượng người dùng tăng đột biến mà không ảnh hưởng đến trải nghiệm.

### Compliance
Phải tuân thủ các chuẩn bảo mật quốc tế (GDPR, SOC2, HIPAA) tùy theo lĩnh vực.

## 11. Common mistakes
- Mistake: Không thiết kế `tenant_id` ngay từ đầu trong các bảng dữ liệu.
  Fix: Chuyển đổi sang mô hình multi-tenancy từ đầu, tránh việc migrate dữ liệu cực kỳ phức tạp sau này.
- Mistake: Bỏ qua việc log hành động của người dùng (Audit Logs) để truy vết lỗi hoặc vi phạm bảo mật.
  Fix: Luôn có service chuyên trách ghi nhận logs theo từng tenant.

## 12. Sample project
**Tên: SaaS Project Management Tool**
Xây dựng một hệ thống quản lý task:
- Mỗi tổ chức (tenant) có danh sách user riêng.
- Một user có thể thuộc nhiều tổ chức (hoặc chỉ một).
- Phải đảm bảo logic query luôn có `WHERE tenant_id = ?`.

## 13. Interview
### Core Q&A
1. Q: Kiến trúc Multi-tenancy là gì?
   A: Là kiến trúc trong đó một instance duy nhất của ứng dụng phục vụ nhiều khách hàng (tenants) khác nhau, nhưng dữ liệu và cấu hình của họ được cách ly hoàn toàn.
2. Q: Làm sao để cô lập dữ liệu hiệu quả trong SaaS?
   A: Có 3 cách phổ biến: Database-per-tenant (tách biệt vật lý), Schema-per-tenant (tách biệt bảng), và Shared-database (dùng cột `tenant_id`).

### Scenario
**Tình huống:** Bạn đang thiết kế hệ thống SaaS cho nhiều công ty khác nhau. Một khách hàng yêu cầu dữ liệu của họ phải được lưu riêng biệt hoàn toàn (physical isolation) thay vì dùng shared database. Bạn tư vấn thế nào?
**Giải đáp:** Phân tích nhu cầu về chi phí vs. bảo mật. Đề xuất mô hình "Hybrid" hoặc "Enterprise-tier" nơi các khách hàng trả phí cao hơn sẽ được cung cấp dedicated infrastructure.

## 14. References
- Wikipedia - SaaS: https://en.wikipedia.org/wiki/Software_as_a_service
- Martin Fowler on SaaS: https://martinfowler.com/bliki/SaaS.html
- Multi-tenancy Patterns: https://docs.microsoft.com/en-us/azure/architecture/guide/multitenant/

## 15. Real-world Code
- Spring Boot Multi-tenancy examples: https://github.com/spring-projects/spring-boot/wiki
- AWS SaaS Factory resources: https://github.com/aws-samples/aws-saas-factory-reference-architectures

## 16. Community
- SaaS Community: Reddit r/SaaS
- IndieHackers (for founders)
