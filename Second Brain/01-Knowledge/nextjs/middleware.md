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
Middleware trong Next.js là một đoạn mã cho phép bạn can thiệp vào yêu cầu (request) trước khi nó hoàn thành. Nó chạy trước khi nội dung được render, cho phép bạn thay đổi phản hồi (response) bằng cách rewrite, redirect, sửa đổi request/response headers, hoặc phản hồi trực tiếp.

## 2. Why
Trước khi có Middleware, việc kiểm tra quyền truy cập hoặc chuyển hướng phải thực hiện ở từng trang (client-side hoặc SSR), dẫn đến lặp code và trải nghiệm người dùng kém (ví dụ: bị nháy trang khi redirect). Middleware tập trung hóa các logic này tại một nơi duy nhất và chạy cực nhanh ở tầng Edge.

## 3. Mental Model
Hãy tưởng tượng Middleware như một nhân viên bảo vệ đứng ngay tại cửa kiểm soát của một tòa nhà văn phòng. Bất kỳ ai muốn vào (request) đều phải đi qua nhân viên này. Bảo vệ có thể: kiểm tra thẻ tên (auth), chỉ đường sang cửa khác (redirect), hoặc ghi chép lại thông tin người vào (logging) trước khi cho phép họ vào phòng ban cụ thể (page/route).

## 4. Where it fits
Incoming Request -> Middleware -> Page / API Route / Static Asset.

## 5. When to use
- **Authentication/Authorization**: Kiểm tra session/cookie trước khi cho phép vào các trang private.
- **Server-Side Redirects**: Chuyển hướng dựa trên điều kiện (ví dụ: chuyển từ `/` sang `/login`).
- **Path Rewriting**: Thay đổi cấu trúc URL cho A/B testing hoặc hỗ trợ legacy URLs.
- **Bot Detection**: Chặn các request từ bot không mong muốn.
- **Logging & Analytics**: Thu thập dữ liệu request cơ bản.

## 6. When NOT to use
- Đừng dùng Middleware cho các tác vụ nặng về tính toán hoặc truy vấn database lớn (vì Middleware chạy trên Edge Runtime có giới hạn về tài nguyên và timeout).
- Không nên dùng Middleware để thay thế hoàn toàn logic nghiệp vụ trong API Routes.
- Đừng fetch dữ liệu quá nhiều trong Middleware vì nó làm tăng đáng kể độ trễ (latency) cho mọi request.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Chạy cực nhanh (Edge Runtime) | Giới hạn bộ nhớ và thư viện (không dùng được mọi Node.js API) |
| Tập trung hóa logic bảo mật | Tăng độ trễ cho mọi request nếu logic phức tạp |
| Giảm tải cho server chính | Khó debug vì chạy trong môi trường sandbox |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Higher-Order Components (HOC) | Chạy ở client, dễ bị lộ logic và nháy UI. |
| Logic trong `getServerSideProps` | Chạy chậm hơn và phải lặp lại ở từng trang. |
| Edge Functions (Vercel) | Linh hoạt hơn nhưng không được tích hợp sâu vào routing như Middleware. |

## 9. How
```tsx
// middleware.ts (đặt ở gốc project hoặc thư mục src)
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/request'

export function middleware(request: NextRequest) {
  const token = request.cookies.get('session')

  if (!token && request.nextUrl.pathname.startsWith('/dashboard')) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  return NextResponse.next()
}

// Giới hạn Middleware chỉ chạy cho các path cụ thể
export const config = {
  matcher: '/dashboard/:path*',
}
```

## 10. Production concerns
### Scaling
Middleware được phân phối toàn cầu qua Edge Network (nếu dùng Vercel), giúp scale tự động theo traffic mà không cần cấu hình server.

### Failure
Nếu Middleware gặp lỗi (crash), toàn bộ request bị chặn có thể trả về lỗi 500. Cần bọc logic trong `try-catch` cực kỳ cẩn thận.

### Monitoring
Theo dõi thời gian thực thi của Middleware (Execution Time). Vercel cung cấp dashboard để xem Middleware có đang làm chậm ứng dụng hay không.

## 11. Common mistakes
- Mistake: Sử dụng các thư viện Node.js thuần như `fs` hoặc `crypto` (loại cũ).
  Fix: Chỉ sử dụng các API được Edge Runtime hỗ trợ (Web chuẩn như Web Crypto, Fetch).

- Mistake: Không cấu hình `matcher` khiến Middleware chạy cho cả file tĩnh (images, css).
  Fix: Luôn sử dụng `config.matcher` để loại trừ các assets tĩnh nhằm tối ưu hiệu năng.

## 12. Sample project
Tạo một hệ thống đa ngôn ngữ (Internationalization): Middleware phát hiện ngôn ngữ trình duyệt từ header `accept-language` và tự động rewrite URL sang `/en`, `/vi` tương ứng mà không làm thay đổi URL trên thanh địa chỉ.

## 13. Interview
### Core Q&A
1. Q: Middleware chạy ở đâu?
   A: Middleware chạy trên Edge Runtime (một tập hợp con của Node.js API được tối ưu cho tốc độ), thường được triển khai tại các vị trí gần người dùng nhất (Edge locations).

### Scenario
1. Q: Bạn làm thế nào để thực hiện A/B testing với Middleware?
   A: Tôi sẽ dùng Middleware để gán người dùng vào một nhóm (ví dụ: 'bucket-a' hoặc 'bucket-b') dựa trên cookie. Sau đó tôi dùng `NextResponse.rewrite` để hiển thị các phiên bản trang khác nhau mà người dùng không hề biết URL đã bị thay đổi.

## 14. References
- Next.js Middleware Docs: https://nextjs.org/docs/app/building-your-application/routing/middleware
- Edge Runtime API: https://nextjs.org/docs/app/api-reference/edge

## 15. Real-world Code
Tham khảo cách `next-intl` hoặc các thư viện auth như `Clerk` triển khai middleware để xử lý ngôn ngữ và bảo mật.

## 16. Community
- YouTube: "Next.js Middleware in 5 Minutes" - Lee Robinson.
- Discussion: Các thread trên GitHub về giới hạn của Edge Runtime so với Node.js.
