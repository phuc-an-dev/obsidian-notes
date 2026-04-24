---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/system-design"
  - "#topic/architecture"
related:
  - "[[nextjs]]"
  - "[[RESTful API]]"
---

## 1. What
Đây là sự so sánh giữa hai mô hình kiến trúc phần mềm:
1. **Next.js**: Một Full-stack React Framework hỗ trợ SSR/SSG, tích hợp cả Frontend và Backend (API Routes) trong cùng một project.
2. **Spring Boot + React SPA**: Mô hình tách biệt hoàn toàn (Decoupled). Backend chạy bằng Java (Spring Boot) cung cấp REST API, Frontend chạy bằng React thuần (Vite/CRA) dưới dạng Single Page Application (SPA).

## 2. Why
Việc lựa chọn giữa hai stack này ảnh hưởng lớn đến tốc độ phát triển (Time-to-market), khả năng mở rộng (Scalability), và hiệu quả SEO. Next.js tối ưu cho trải nghiệm người dùng và SEO, trong khi Spring Boot tối ưu cho các hệ thống doanh nghiệp phức tạp, yêu cầu tính bảo mật và xử lý dữ liệu nặng.

## 3. Mental Model
- **Next.js** giống như một căn hộ "Full nội thất" (All-in-one). Bạn dọn vào là có đủ bếp, giường, tivi phối hợp sẵn với nhau.
- **Spring Boot + React** giống như việc bạn mua một khu đất (Backend) và tự thuê kiến trúc sư xây nhà (Frontend) lên đó. Bạn có toàn quyền quyết định mọi chi tiết nhưng tốn công kết nối điện nước (Auth, API integration) giữa hai phần.

## 4. Where it fits
- **Next.js Architecture**:
  Browser <-> Next.js Server (SSR/Edge) <-> Database/External API.
- **Spring Boot + React Architecture**:
  Browser <-> React SPA (S3/Static Host) <-> Load Balancer <-> Spring Boot API <-> Database.

## 5. When to use
- **Next.js**:
  - Cần SEO tốt (E-commerce, Landing Page, Blog).
  - Cần tốc độ tải trang đầu tiên (FCP) nhanh.
  - Team quy mô nhỏ đến vừa, muốn code nhanh (Monorepo).
- **Spring Boot + React**:
  - Hệ thống lớn (Enterprise) với hàng trăm microservices.
  - Yêu cầu xử lý đa luồng (Multi-threading) phức tạp hoặc tính toán nặng ở Backend.
  - Team Frontend và Backend làm việc độc lập hoàn toàn.

## 6. When NOT to use
- **Next.js**:
  - Khi ứng dụng chủ yếu là Dashboard nội bộ cực kỳ phức tạp và không cần SEO.
  - Khi doanh nghiệp bắt buộc sử dụng hệ sinh thái Java/Spring để tuân thủ bảo mật.
- **Spring Boot + React**:
  - Khi cần SEO mà không muốn tốn công cấu hình SSR phức tạp.
  - Các project nhỏ, cần prototype nhanh (MVP).

## 7. Trade-offs
| Tiêu chí | Next.js | Spring Boot + React |
|----------|---------|----------------------|
| SEO | Mặc định cực tốt | Khó (Cần thêm giải pháp phụ) |
| Performance | Tối ưu Client (Bundle size) | Tối ưu Server (Data processing) |
| Development | Nhanh, đồng nhất JS/TS | Chậm hơn, cần quản lý 2 stacks |
| Ecosystem | Mạnh về UI/UX | Mạnh về Enterprise/Security |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Remix + Go | Một giải pháp fullstack khác tập trung vào Web Standards. |
| Laravel + Vue | Phổ biến trong thế giới PHP, tương tự mô hình fullstack của Next.js. |

## 9. How
- **Next.js**:
```typescript
// app/api/data/route.ts (Backend) + app/page.tsx (Frontend) trong cùng repo.
export async function GET() {
  return Response.json({ message: "Hello from Next.js" });
}
```
- **Spring Boot**:
```java
// Controller.java (Backend riêng)
@RestController
public class HelloController {
    @GetMapping("/api/hello")
    public String hello() { return "Hello from Spring Boot"; }
}
```

## 10. Production concerns
### Scaling
Next.js scale tốt ở tầng Edge/CDN. Spring Boot scale tốt ở tầng xử lý logic và kết nối cơ sở dữ liệu nhờ JVM.

### Failure
Nếu Next.js server chết, cả FE và BE (nếu dùng API Routes) đều chết. Ở mô hình tách biệt, nếu BE chết, người dùng vẫn thấy giao diện FE (nhưng không có dữ liệu).

### Monitoring
Next.js theo dõi Vercel Analytics/Sentry. Spring Boot theo dõi qua Actuator/Prometheus/Grafana.

## 11. Common mistakes
- Mistake: Dùng Next.js làm Backend chính cho các tác vụ cần xử lý Java-specific libraries.
  Fix: Sử dụng Spring Boot làm Backend và Next.js làm Frontend (BFF - Backend For Frontend).

- Mistake: Cố gắng implement SSR cho React SPA thủ công thay vì dùng Next.js.
  Fix: Chuyển sang Next.js ngay từ đầu nếu SEO là yêu cầu bắt buộc.

## 12. Sample project
Xây dựng một hệ thống ngân hàng:
- **Frontend**: Next.js (để tối ưu SEO cho trang landing và marketing).
- **Backend**: Spring Boot (xử lý giao dịch, bảo mật và tích hợp hệ thống lõi).

## 13. Interview
### Core Q&A
1. Q: Tại sao chọn Next.js thay vì React SPA + Spring Boot cho một trang thương mại điện tử?
   A: Vì thương mại điện tử cần SEO để lên top tìm kiếm và tốc độ render ban đầu (SSR) để giảm tỉ lệ thoát (Bounce rate). Next.js hỗ trợ sẵn SSG/ISR cho hàng triệu sản phẩm mà vẫn đảm bảo tốc độ.

### Scenario
1. Q: Nếu hệ thống yêu cầu tích hợp với các thư viện xử lý tài chính chỉ có trong Java, bạn tổ chức stack như thế nào?
   A: Tôi sẽ dùng Spring Boot làm API Server chính. Next.js sẽ đóng vai trò Frontend và có thể có thêm một tầng BFF (API Routes) để trung chuyển/format dữ liệu trước khi gửi xuống client.

## 14. References
- Next.js Documentation: https://nextjs.org/docs
- Spring Boot Documentation: https://spring.io/projects/spring-boot

## 15. Real-world Code
Nghiên cứu kiến trúc của các trang như Netflix hoặc Airbnb: Họ thường dùng kết hợp Java/Node.js cho Backend và React/Next.js cho Frontend.

## 16. Community
- Discussion: Next.js vs Spring Boot trên Stack Overflow và Reddit.
- Conference talks: "Modern Web Architecture" tại các sự kiện tech lớn.
