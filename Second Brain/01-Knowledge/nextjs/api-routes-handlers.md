---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/react"
  - "#topic/http"
related:
  - "[[nextjs]]"
  - "[[app-router]]"
---

## 1. What
API Routes (Pages Router) và Route Handlers (App Router) là cơ chế của Next.js cho phép tạo ra các điểm cuối HTTP (HTTP Endpoints) để xử lý yêu cầu phía server. Chúng cho phép bạn viết code server-side (Node.js/Edge runtime) để tương tác với cơ sở dữ liệu hoặc dịch vụ bên thứ ba.

## 2. Why
Trước đây, ứng dụng web thường cần một server backend riêng biệt (như Express.js). Next.js tích hợp sẵn cơ chế này để giảm bớt sự phức tạp trong việc triển khai, cho phép cả frontend và backend nằm chung một repo (Monorepo), chia sẻ kiểu dữ liệu (Types) và đơn giản hóa quá trình xác thực.

## 3. Mental Model
Hãy tưởng tượng Next.js là một tòa nhà văn phòng. Các trang web (Pages) là quầy lễ tân phục vụ khách hàng. API Routes/Route Handlers là các phòng ban nội bộ nằm sau cánh cửa "chỉ dành cho nhân viên". Khách hàng không nhìn thấy họ trực tiếp, nhưng lễ tân có thể gọi điện vào trong để lấy thông tin cần thiết.

## 4. Where it fits
HTTP Request -> Next.js Router -> Route Handler (route.ts) -> Business Logic -> Response (JSON/Blob).

## 5. When to use
- Khi cần tạo các webhook cho các dịch vụ như Stripe, GitHub.
- Khi cần che giấu các khóa API nhạy cảm không muốn lộ ở trình duyệt.
- Khi xây dựng các API nhẹ cho ứng dụng di động hoặc bên thứ ba.
- Khi cần xử lý các tác vụ nặng hoặc truy cập database trực tiếp.

## 6. When NOT to use
- Khi hệ thống có logic nghiệp vụ cực kỳ phức tạp và cần scale backend độc lập với frontend.
- Khi cần các tính năng chuyên sâu của backend framework như WebSockets (Next.js route handlers không hỗ trợ tốt).
- Khi bạn đã có một hệ thống microservices backend riêng biệt mạnh mẽ.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Dễ dàng triển khai (Zero config) | Khó quản lý khi logic backend quá lớn |
| Chia sẻ code/types giữa FE và BE | Giới hạn về thời gian thực hiện (Timeout trên serverless) |
| Bảo mật hơn Client-side fetching | Không hỗ trợ các giao thức như gRPC, WebSockets mặc định |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Express.js | Linh hoạt hơn, hỗ trợ middleware mạnh mẽ hơn nhưng cần server riêng. |
| Server Actions | (Next.js) Tốt hơn cho các thao tác từ Form/UI mà không cần tạo endpoint thủ công. |
| NestJS | Framework backend chuyên sâu, có cấu trúc tốt hơn cho dự án lớn. |

## 9. How
```tsx
// app/api/users/route.ts
import { NextResponse } from 'next/server';

export async function GET() {
  const users = [{ id: 1, name: 'An Phuc' }];
  return NextResponse.json(users);
}

export async function POST(request: Request) {
  const body = await request.json();
  // Xử lý logic lưu database ở đây
  return NextResponse.json({ message: 'User created', data: body }, { status: 201 });
}
```

## 10. Production concerns
### Scaling
Trên Vercel, các route này chạy dưới dạng Serverless Functions. Cần lưu ý giới hạn về số lượng request đồng thời và bộ nhớ.

### Failure
Cần bọc code trong `try-catch` và trả về mã lỗi HTTP tương ứng (400, 401, 500) để frontend có thể xử lý đúng.

### Monitoring
Sử dụng các công cụ như Logtail hoặc tích hợp sẵn log của Vercel để theo dõi các yêu cầu thất bại hoặc thời gian phản hồi chậm.

## 11. Common mistakes
- Mistake: Để file `route.ts` cùng thư mục với file `page.tsx`.
  Fix: Một phân đoạn route chỉ có thể chứa hoặc `page.tsx` hoặc `route.ts`, không được cả hai.

- Mistake: Không xử lý CORS khi gọi API từ domain khác.
  Fix: Cấu hình headers thủ công trong `NextResponse` nếu cần hỗ trợ cross-origin.

## 12. Sample project
Xây dựng một hệ thống xử lý thanh toán: Frontend gửi thông tin đơn hàng đến Route Handler, Route Handler gọi API của Stripe để tạo phiên thanh toán và trả về URL cho người dùng.

## 13. Interview
### Core Q&A
1. Q: Route Handlers hỗ trợ những phương thức HTTP nào?
   A: Hỗ trợ đầy đủ: GET, POST, PUT, PATCH, DELETE, HEAD, và OPTIONS.

### Scenario
1. Q: Bạn sẽ làm gì nếu Route Handler của bạn cần xử lý một tác vụ tốn hơn 10 giây nhưng serverless function có timeout là 10 giây?
   A: Tôi sẽ chuyển tác vụ đó sang một hàng đợi công việc (Job Queue) bên ngoài như Upstash QStash hoặc AWS SQS và trả về phản hồi 202 Accepted ngay lập tức cho client.

## 14. References
- Next.js Route Handlers: https://nextjs.org/docs/app/building-your-application/routing/route-handlers
- Edge Runtime: https://nextjs.org/docs/app/api-reference/edge

## 15. Real-world Code
Nghiên cứu các thư viện như `next-auth` để xem cách họ sử dụng Route Handlers để xử lý toàn bộ quy trình OAuth.

## 16. Community
- Discussion: Sự khác biệt giữa Server Actions và Route Handlers trên Reddit r/nextjs.
- Blog: "When to use Route Handlers vs Server Actions" - Vercel Blog.
