---
created: 2026-04-24
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/react"
  - "#topic/performance"
related:
  - "[[react-lifecycle]]"
---

## 1. What
Next.js là một React framework mã nguồn mở cung cấp các giải pháp cho Server-Side Rendering (SSR), Static Site Generation (SSG), và Incremental Static Regeneration (ISR). Nó giúp xây dựng các ứng dụng web tối ưu SEO và có trải nghiệm người dùng nhanh chóng.

## 2. Why
React thuần túy là Client-Side Rendering (CSR), dẫn đến vấn đề SEO kém (do crawler không đọc được nội dung render bằng JS) và trải nghiệm người dùng ban đầu chậm (phải chờ load bundle JS lớn). Next.js giải quyết vấn đề này bằng cách tiền xử lý trang web trên server.

## 3. Mental Model
Hãy coi React thuần là một bộ linh kiện IKEA mà bạn phải tự lắp ráp tại nhà khách hàng (Client). Next.js là một xưởng mộc lớn, nơi họ có thể lắp ráp sẵn đồ nội thất (SSR) hoặc đóng thùng sẵn các mẫu phổ biến (SSG) trước khi giao đến, giúp khách hàng sử dụng được ngay lập tức.

## 4. Where it fits
Request -> Next.js Server (App Router/Pages Router) -> Fetch Data -> Render HTML -> Response to Client -> Hydration (React takes over).

## 5. When to use
- Khi dự án yêu cầu SEO tốt (E-commerce, Blog, News).
- Khi cần tối ưu First Contentful Paint (FCP).
- Khi muốn tận dụng hệ thống Routing dựa trên file system mạnh mẽ.
- Khi cần các API Routes tích hợp sẵn.

## 6. When NOT to use
- Các ứng dụng Dashboard nội bộ không cần SEO.
- Các ứng dụng cực kỳ đơn giản mà React SPA là quá đủ.
- Khi không muốn phụ thuộc vào kiến trúc server phức tạp của Vercel/Node.js.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| SEO vượt trội và Performance tốt | Cấu hình Server phức tạp hơn SPA |
| Hỗ trợ nhiều chế độ render (SSR, SSG, ISR) | Thời gian Build có thể lâu với SSG lớn |
| Tự động tối ưu Image, Font, Script | Khó debug các lỗi chỉ xảy ra ở phía Server |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Remix | Tập trung vào web standards và tối ưu data loading mạnh mẽ. |
| Gatsby | Mạnh về SSG và hệ sinh thái GraphQL. |
| Astro | Tối ưu cho nội dung tĩnh, "Zero JS by default". |

## 9. How
```tsx
// App Router: app/page.tsx
export default async function Page() {
  const data = await fetch('https://api.example.com/data');
  const result = await data.json();

  return (
    <main>
      <h1>Next.js Server Component</h1>
      <pre>{JSON.stringify(result, null, 2)}</pre>
    </main>
  );
}
```

## 10. Production concerns
### Scaling
Sử dụng Vercel là cách dễ nhất, hoặc deploy Docker container lên AWS/K8s. Cần lưu ý về caching ở tầng CDN.

### Failure
Xử lý `error.tsx` cho từng phân đoạn route và sử dụng `loading.tsx` để hiển thị UI chờ.

### Monitoring
Sử dụng Next.js Analytics hoặc tích hợp Sentry để theo dõi hiệu năng (Web Vitals) và lỗi server-side.

## 11. Common mistakes
- Mistake: Sử dụng "use client" bừa bãi ở tất cả các file.
  Fix: Giữ các component là Server Component càng nhiều càng tốt, chỉ dùng Client Component ở lá của cây component.

- Mistake: Không tận dụng caching của hàm `fetch` trong App Router.
  Fix: Hiểu rõ cơ chế Data Cache và Request Memoization để tránh gọi API thừa.

## 12. Sample project
Xây dựng một trang Blog cá nhân sử dụng Contentlayer để đọc file Markdown (SSG), tích hợp ISR để cập nhật bài viết mà không cần build lại toàn bộ.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa SSR và SSG trong Next.js là gì?
   A: SSR (Server-Side Rendering) tạo HTML cho mỗi request, phù hợp dữ liệu thay đổi liên tục. SSG (Static Site Generation) tạo HTML một lần lúc build, phù hợp dữ liệu ít thay đổi, cho tốc độ load cực nhanh.

### Scenario
1. Q: Làm thế nào để xử lý một trang sản phẩm có hàng triệu item mà vẫn muốn dùng SSG?
   A: Sử dụng ISR (Incremental Static Regeneration) kết hợp với `fallback: 'blocking'`. Chỉ build sẵn các sản phẩm hot, các sản phẩm còn lại sẽ được render và cache lần đầu khi có người truy cập.

## 14. References
- Official Docs: https://nextjs.org/docs
- GitHub Repo: https://github.com/vercel/next.js
- Spec / RFC: https://nextjs.org/blog
- Changelog: https://nextjs.org/docs/app/building-your-application/upgrading

## 15. Real-world Code
Học tập từ repo `vercel/commerce` để xem cách họ xử lý E-commerce quy mô lớn với Next.js.

## 16. Community
- Reddit: r/nextjs
- Stack Overflow: [next.js] tag
- Blog: Vercel Blog
- Talk: Next.js Conf (YouTube)
