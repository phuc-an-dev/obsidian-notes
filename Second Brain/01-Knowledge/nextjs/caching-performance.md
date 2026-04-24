---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/react"
  - "#topic/performance"
related:
  - "[[nextjs]]"
  - "[[app-router]]"
  - "[[rendering-strategies]]"
---

## 1. What
Caching trong Next.js (App Router) là một hệ thống đa lớp nhằm giảm thiểu việc lặp lại các tác vụ fetch dữ liệu và render trang. Nó bao gồm 4 tầng chính: Request Memoization, Data Cache, Full Route Cache, và Router Cache.

## 2. Why
Việc fetch dữ liệu từ API hoặc database và render React components tốn kém tài nguyên (CPU, Network). Caching giúp ứng dụng phản hồi nhanh hơn, giảm tải cho backend và tiết kiệm chi phí băng thông bằng cách tái sử dụng kết quả từ những yêu cầu trước đó.

## 3. Mental Model
Hãy tưởng tượng Caching như một chuỗi cửa hàng tiện lợi:
- **Request Memoization**: Giống như trí nhớ ngắn hạn của nhân viên thu ngân trong một ca trực (một request).
- **Data Cache**: Giống như kho hàng tại cửa hàng, lưu trữ nguyên liệu thô (dữ liệu API).
- **Full Route Cache**: Giống như đồ ăn đã nấu sẵn và bày trên kệ, khách chỉ việc lấy (HTML/RSC payload).
- **Router Cache**: Giống như túi đồ khách hàng vừa mua và đang cầm trên tay (cache ở trình duyệt).

## 4. Where it fits
Request -> [Router Cache (Client)] -> [Full Route Cache (Server)] -> [Data Cache (Server)] -> [Request Memoization (Server)] -> Data Source.

## 5. When to use
- **Request Memoization**: Luôn được áp dụng tự động cho các hàm `fetch` trong một chu kỳ render.
- **Data Cache**: Dùng để lưu trữ kết quả từ API ngoài hoặc database qua nhiều request khác nhau.
- **Full Route Cache**: Dùng cho các trang tĩnh (Static Routes) để tăng tốc độ phản hồi HTML.
- **Router Cache**: Dùng để chuyển trang tức thì (Instant navigation) ở phía client.

## 6. When NOT to use
- Đừng dùng caching cho các dữ liệu yêu cầu tính bảo mật cao và thay đổi theo từng người dùng (như số dư tài khoản).
- Tránh caching cho các route mang tính chất Real-time (như chat, giá chứng khoán cập nhật từng giây).
- Khi sử dụng các hàm dynamic như `cookies()` hoặc `headers()`, Full Route Cache sẽ bị bỏ qua.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Giảm TTFB và thời gian tải trang | Dữ liệu có thể bị cũ (Stale data) |
| Tiết kiệm chi phí API và Database | Phức tạp trong việc quản lý cơ chế Invalidation |
| Tăng khả năng chịu tải của hệ thống | Tốn dung lượng lưu trữ trên server/Edge |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Redis | Caching bên ngoài, linh hoạt hơn nhưng cần quản lý server riêng. |
| React Query | Quản lý cache ở Client, tốt cho ứng dụng tương tác mạnh nhưng không hỗ trợ SSR SEO tốt bằng Next.js native cache. |

## 9. How
```tsx
// 1. Data Cache với Revalidation (Time-based)
const data = await fetch('https://api.com/data', { 
  next: { revalidate: 3600 } // Cache trong 1 giờ
});

// 2. Data Cache với Tags (On-demand)
const data2 = await fetch('https://api.com/data', { 
  next: { tags: ['posts'] } 
});

// Revalidate thủ công (thường dùng trong Server Actions)
// revalidateTag('posts');

// 3. Opt-out Caching (Dynamic rendering)
const data3 = await fetch('https://api.com/data', { 
  cache: 'no-store' 
});
```

## 10. Production concerns
### Scaling
Trong môi trường multi-region, Data Cache của Next.js (trên Vercel) được phân phối toàn cầu. Nếu tự host, bạn cần cấu hình storage chung cho cache để đảm bảo tính đồng nhất.

### Failure
Nếu cache bị lỗi hoặc không truy cập được, Next.js sẽ tự động fallback về việc fetch dữ liệu trực tiếp, đảm bảo ứng dụng vẫn hoạt động.

### Monitoring
Sử dụng header `X-Nextjs-Cache` để kiểm tra trạng thái (HIT, MISS, STALE) của từng request trong môi trường Production.

## 11. Common mistakes
- Mistake: Quên rằng `fetch` trong Server Components được cache mặc định.
  Fix: Sử dụng `{ cache: 'no-store' }` nếu dữ liệu cần luôn luôn mới.

- Mistake: Revalidate nhầm tag hoặc không revalidate sau khi thay đổi dữ liệu trong DB.
  Fix: Luôn gọi `revalidatePath` hoặc `revalidateTag` ngay sau khi thực hiện các thao tác ghi dữ liệu (mutation).

## 12. Sample project
Xây dựng một trang E-commerce với trang danh sách sản phẩm dùng ISR (revalidate mỗi 5 phút) và trang chi tiết sản phẩm dùng On-demand Revalidation khi admin cập nhật giá trong hệ thống CMS.

## 13. Interview
### Core Q&A
1. Q: Request Memoization khác gì với Data Cache?
   A: Request Memoization chỉ tồn tại trong vòng đời của một request (tái sử dụng data giữa các component trong cùng một trang). Data Cache tồn tại xuyên suốt nhiều request và nhiều người dùng khác nhau trên server cho đến khi bị invalid hoặc hết hạn.

### Scenario
1. Q: Làm thế nào để xóa cache của một trang cụ thể khi người dùng thực hiện update thông tin?
   A: Tôi sẽ sử dụng hàm `revalidatePath('/path-to-page')` hoặc gán một tag cụ thể cho dữ liệu đó và dùng `revalidateTag('my-tag')` trong một Server Action.

## 14. References
- Next.js Caching: https://nextjs.org/docs/app/building-your-application/caching
- Revalidating: https://nextjs.org/docs/app/building-your-application/data-fetching/incremental-static-regeneration

## 15. Real-world Code
Nghiên cứu kiến trúc của `Vercel Platforms Starter Kit` để thấy cách họ quản lý cache cho hàng ngàn tên miền phụ khác nhau.

## 16. Community
- YouTube: "Next.js 13 Caching Explained" - Jack Herrington.
- Blog: "Delivering fast applications with Next.js caching layers" - Vercel Blog.
