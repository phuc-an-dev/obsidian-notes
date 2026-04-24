---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/react"
  - "#topic/ui"
related:
  - "[[nextjs]]"
  - "[[server-vs-client-components]]"
---

## 1. What
Styling trong Next.js là hệ thống hỗ trợ nhiều phương thức định dạng giao diện người dùng, bao gồm Global CSS, CSS Modules, Tailwind CSS, Sass, và các thư viện CSS-in-JS. Next.js tối ưu hóa CSS bằng cách tự động tách nhỏ (code-splitting) và chỉ tải CSS cần thiết cho từng trang.

## 2. Why
Việc quản lý CSS trong ứng dụng lớn thường gặp các vấn đề: xung đột tên class, dung lượng file CSS quá lớn, và hiệu năng render (FOUC - Flash of Unstyled Content). Next.js cung cấp các giải pháp tích hợp sẵn để giải quyết những vấn đề này, đồng thời hỗ trợ tốt cho việc render phía server.

## 3. Mental Model
Hãy tưởng tượng việc styling như cách bạn chọn trang phục:
- **Global CSS**: Như bộ đồng phục toàn trường, ai cũng mặc giống nhau (dùng cho các style chung).
- **CSS Modules**: Như bộ đồ may đo riêng cho từng cá nhân, không lo bị đụng hàng (scope riêng cho từng component).
- **Tailwind CSS**: Như một tủ đồ với hàng ngàn phụ kiện may sẵn (utility classes), bạn chỉ cần nhặt và kết hợp chúng lại cực nhanh.

## 4. Where it fits
React Component -> Styling Strategy (Tailwind/Modules) -> PostCSS Processing -> Optimized CSS File -> Browser.

## 5. When to use
- **Tailwind CSS**: Khi muốn phát triển nhanh, duy trì tính nhất quán và giảm thiểu việc phải viết file CSS riêng.
- **CSS Modules**: Khi muốn sự cô lập (isolation) tuyệt đối và ưu tiên viết CSS truyền thống.
- **Global CSS**: Chỉ dùng cho CSS reset, font chữ hệ thống hoặc các style cần áp dụng toàn bộ ứng dụng.

## 6. When NOT to use
- Đừng dùng các thư viện CSS-in-JS cũ (như `styled-components` phiên bản cũ) trong Server Components vì chúng yêu cầu runtime JavaScript để tính toán style.
- Không lạm dụng Global CSS để style cho các component cụ thể vì dễ dẫn đến xung đột và khó bảo trì.

## 7. Trade-offs
| Phương pháp | Tốc độ phát triển | Hiệu năng | Độ linh hoạt |
|-------------|-------------------|-----------|--------------|
| Tailwind CSS | Rất nhanh | Rất cao | Cao |
| CSS Modules | Trung bình | Cao | Rất cao |
| CSS-in-JS | Nhanh | Trung bình | Cực cao |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Sass/SCSS | Cung cấp biến và nesting mạnh mẽ hơn CSS thuần, cần cài thêm `sass`. |
| Styled Components | Viết CSS ngay trong file JS, cần cấu hình đặc biệt cho SSR trong Next.js. |
| Panda CSS | Giải pháp CSS-in-JS hiện đại, render-time bằng 0, hỗ trợ tốt RSC. |

## 9. How
### CSS Modules
```tsx
// styles.module.css
.button { color: blue; }

// Component.tsx
import styles from './styles.module.css';
export default function Button() {
  return <button className={styles.button}>Click me</button>;
}
```

### Tailwind CSS
```tsx
export default function Button() {
  return <button className="bg-blue-500 text-white p-2 rounded">Click me</button>;
}
```

## 10. Production concerns
### Scaling
Sử dụng Tailwind CSS giúp giữ cho dung lượng CSS không tăng trưởng tỉ lệ thuận với số lượng trang (do tái sử dụng class).

### Failure
Nếu CSS không load được, giao diện sẽ bị vỡ hoàn toàn. Next.js giúp giảm thiểu điều này bằng cách inlining CSS quan trọng (Critical CSS).

### Monitoring
Theo dõi kích thước file CSS cuối cùng. Tailwind tự động xóa các class không dùng (Purge) trong môi trường Production.

## 11. Common mistakes
- Mistake: Import Global CSS vào file không phải `layout.tsx`.
  Fix: Global CSS chỉ được phép import tại Root Layout hoặc `_app.js`.

- Mistake: Dùng CSS-in-JS mà quên khai báo `"use client"`.
  Fix: Hầu hết CSS-in-JS cần Context API của React, nên component đó phải là Client Component.

## 12. Sample project
Tạo một hệ thống Design System nhỏ với Tailwind CSS, bao gồm các biến màu sắc (colors), khoảng cách (spacing) và các component cơ bản như Button, Input, Card.

## 13. Interview
### Core Q&A
1. Q: Next.js làm thế nào để tránh xung đột CSS?
   A: Thông qua CSS Modules, Next.js tự động hash tên class (ví dụ: `button` thành `button_a3f12`), đảm bảo class đó là duy nhất trong toàn bộ ứng dụng.

### Scenario
1. Q: Nếu bạn dùng một thư viện CSS-in-JS trong App Router, bạn cần lưu ý điều gì?
   A: Tôi cần kiểm tra xem thư viện đó có hỗ trợ Server Components hay không. Nếu không, tôi phải bọc component đó trong một Client Component và đảm bảo quá trình "Style Injection" được xử lý đúng ở phía server để tránh FOUC.

## 14. References
- Next.js Styling Docs: https://nextjs.org/docs/app/building-your-application/styling
- Tailwind CSS with Next.js: https://tailwindcss.com/docs/guides/nextjs

## 15. Real-world Code
Nghiên cứu `shadcn/ui` - một bộ component nổi tiếng kết hợp Tailwind CSS và Radix UI cực kỳ hiệu quả trong Next.js.

## 16. Community
- YouTube: "The best way to style Next.js apps" - Web Dev Simplified.
- Twitter: #TailwindCSS #NextJS.
