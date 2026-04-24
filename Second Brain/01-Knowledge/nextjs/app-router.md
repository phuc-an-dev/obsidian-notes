---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/react"
  - "#topic/routing"
related:
  - "[[nextjs]]"
---

## 1. What
App Router là hệ thống routing mới của Next.js (từ phiên bản 13), được xây dựng trên nền tảng React Server Components. Nó sử dụng thư mục `app` làm gốc và hỗ trợ các tính năng hiện đại như Shared Layouts, Nested Routing, và Streaming.

## 2. Why
Pages Router cũ gặp hạn chế trong việc chia sẻ layout phức tạp và tối ưu hóa hiệu năng (do phải load JS cho toàn bộ trang ở client). App Router ra đời để tận dụng Server Components, giúp giảm lượng JavaScript gửi xuống client và cho phép quản lý layout lồng nhau một cách tự nhiên.

## 3. Mental Model
Hãy tưởng tượng App Router như một hệ thống cây phân cấp. Mỗi thư mục là một "node" trong lộ trình. File `layout.tsx` giống như cái khung của một căn phòng, còn `page.tsx` là đồ nội thất bên trong. Khi bạn di chuyển giữa các phòng trong cùng một tầng, cái khung (layout) vẫn giữ nguyên, bạn chỉ thay đổi đồ nội thất (page).

## 4. Where it fits
Browser Request -> App Router Engine -> Layout Tree Resolution -> Server Components Execution -> HTML Streaming -> Client Hydration.

## 5. When to use
- Khi bắt đầu dự án Next.js mới (đây là hướng đi chính thức của Next.js).
- Khi cần xây dựng dashboard phức tạp có nhiều tầng layout lồng nhau.
- Khi muốn tối ưu Core Web Vitals bằng cách render phần lớn component trên server.

## 6. When NOT to use
- Khi dự án đang chạy ổn định trên Pages Router và không có nhu cầu refactor lớn.
- Khi các thư viện bên thứ ba bạn dùng chưa hỗ trợ tốt React Server Components (RSC).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Giảm bundle size client đáng kể | Đường cong học tập dốc (RSC concept) |
| Shared Layouts cực kỳ mạnh mẽ | Một số thư viện CSS-in-JS cũ chưa hỗ trợ |
| Hỗ trợ Streaming và Suspense mặc định | Khó debug lỗi hydration hơn |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Pages Router | Đơn giản hơn, dựa trên Client-side rendering nhiều hơn. |
| React Router | Dành cho ứng dụng React SPA thuần túy. |

## 9. How
```tsx
// app/dashboard/layout.tsx
export default function DashboardLayout({ children }: { children: React.ReactNode }) {
  return (
    <section>
      <nav>Sidebar</nav>
      {children}
    </section>
  );
}

// app/dashboard/page.tsx
export default function Page() {
  return <h1>Welcome to Dashboard</h1>;
}
```

## 10. Production concerns
### Scaling
App Router tận dụng tối đa cơ chế caching của Next.js. Cần hiểu rõ `force-dynamic` và `revalidate` để scale dữ liệu động.

### Failure
Sử dụng file `error.js` và `not-found.js` ở từng cấp độ thư mục để catch lỗi cục bộ, tránh sập toàn bộ ứng dụng.

### Monitoring
Theo dõi thời gian phản hồi của Server Components (TTFB) vì quá trình fetch dữ liệu xảy ra trên server trước khi trả về HTML.

## 11. Common mistakes
- Mistake: Khai báo `"use client"` ở quá cao trong cây component.
  Fix: Chỉ đẩy `"use client"` xuống các component thực sự cần interactivity (button, form, state).

- Mistake: Quên rằng component trong thư mục `app` mặc định là Server Component.
  Fix: Luôn ghi nhớ không thể dùng `useState` hay `useEffect` trong Server Component.

## 12. Sample project
Xây dựng một hệ thống Dashboard quản lý kho hàng với Sidebar (Root Layout), danh sách sản phẩm (Nested Page), và trang chi tiết sản phẩm (Dynamic Route).

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt lớn nhất giữa App Router và Pages Router là gì?
   A: App Router dựa trên Server Components theo mặc định và hỗ trợ Nested Layouts, trong khi Pages Router dựa trên Page-based routing và render toàn bộ trang ở client sau khi fetch data qua `getStaticProps` hoặc `getServerSideProps`.

### Scenario
1. Q: Bạn làm thế nào để fetch data trong App Router?
   A: Trong App Router, chúng ta có thể dùng `async/await` trực tiếp ngay trong Server Component. Việc fetch data trở nên đơn giản hơn vì không cần các hàm đặc biệt như `getServerSideProps`.

## 14. References
- Official Docs: https://nextjs.org/docs/app
- GitHub Repo: https://github.com/vercel/next.js
- Blog Post: https://nextjs.org/blog/next-13

## 15. Real-world Code
Tham khảo `nextjs/app-playground` trên GitHub để thấy các pattern phức tạp về Parallel Routes và Intercepting Routes.

## 16. Community
- Discord: Next.js Official Server
- Twitter: #NextJS #Vercel
- Blog: Lee Robinson's Blog (VP of DX at Vercel)
