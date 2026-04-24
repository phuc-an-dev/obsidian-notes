---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/react"
  - "#topic/routing"
related:
  - "[[nextjs]]"
  - "[[app-router]]"
---

## 1. What
Layout System trong Next.js là cơ chế cho phép chia sẻ giao diện người dùng (UI) giữa các trang khác nhau trong một lộ trình (route). Layout duy trì trạng thái (state), duy trì tính tương tác, và không render lại (re-render) khi người dùng di chuyển giữa các trang con.

## 2. Why
Trước đây, việc tạo Sidebar hoặc Navbar dùng chung yêu cầu phải bọc component ở tầng cao nhất (như `_app.js` trong Pages Router), gây khó khăn khi muốn có các Layout khác nhau cho từng phân đoạn web. Layout System ra đời để giải quyết vấn đề chia sẻ UI theo từng cấp độ và tối ưu hiệu năng bằng cách tránh render lại những phần không thay đổi.

## 3. Mental Model
Hãy tưởng tượng Layout như một cái khung ảnh. Khi bạn thay đổi bức ảnh bên trong (Page), cái khung (Layout) vẫn giữ nguyên vị trí, không bị tháo ra lắp lại. Nếu bạn có một khung ảnh nhỏ hơn lồng bên trong (Nested Layout), nó cũng hoạt động tương tự.

## 4. Where it fits
Root Layout (`app/layout.tsx`) -> Nested Layout (`app/dashboard/layout.tsx`) -> Template (`app/dashboard/template.tsx`) -> Page (`app/dashboard/page.tsx`).

## 5. When to use
- Khi cần các thành phần UI xuất hiện trên nhiều trang: Navbar, Footer, Sidebar.
- Khi muốn duy trì trạng thái của UI (ví dụ: thanh tìm kiếm giữ nguyên nội dung khi chuyển trang).
- Khi muốn tối ưu hiệu năng bằng cách giảm số lượng phần tử cần render lại.

## 6. When NOT to use
- Đừng dùng Layout nếu bạn muốn component đó phải "reset" (chạy lại `useEffect`, reset state) mỗi khi chuyển trang. Trong trường hợp này, hãy dùng **Template**.
- Không nên để quá nhiều logic xử lý dữ liệu nặng trong Root Layout nếu nó làm chậm quá trình render ban đầu của toàn bộ app.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Duy trì state cực tốt khi điều hướng | Phức tạp hơn khi cần đồng bộ giữa Layout và Page |
| Hiệu năng cao (Partial Rendering) | Khó xử lý các animation cần reset hoàn toàn |
| Cấu trúc code sạch sẽ, phân cấp rõ ràng | Dễ nhầm lẫn giữa Layout và Template |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Template | Giống Layout nhưng tạo instance mới (mount lại) trên mỗi lần chuyển trang. |
| `_app.js` (Pages Router) | Chỉ có một layout duy nhất cho toàn bộ ứng dụng. |

## 9. How
```tsx
// app/dashboard/layout.tsx
export default function DashboardLayout({
  children, // Phải có children đại diện cho Page hoặc Nested Layout
}: {
  children: React.ReactNode
}) {
  return (
    <div className="dashboard-container">
      <aside>Sidebar Menu</aside>
      <main>{children}</main>
    </div>
  )
}
```

## 10. Production concerns
### Scaling
Root Layout bắt buộc phải chứa thẻ `<html>` và `<body>`. Đây là nơi lý tưởng để nhúng các script theo dõi (Analytics) hoặc cấu hình font chữ hệ thống.

### Failure
Sử dụng `loading.js` cùng cấp với Layout để tạo trải nghiệm "Instant Loading State" trong khi Layout vẫn đang render.

### Monitoring
Lưu ý rằng Layout là Server Component theo mặc định. Cần kiểm tra kỹ nếu bạn định sử dụng các Context Provider (phải tạo Client Component trung gian).

## 11. Common mistakes
- Mistake: Quên truyền `children` prop trong Layout.
  Fix: Luôn khai báo và render `{children}` để các route con có thể hiển thị.

- Mistake: Khai báo `"use client"` cho toàn bộ Root Layout chỉ để dùng một nút bấm.
  Fix: Tách nút bấm đó ra một file riêng làm Client Component và import vào Layout.

## 12. Sample project
Tạo một ứng dụng E-learning:
- **Root Layout**: Navbar và Footer chính.
- **Course Layout**: Sidebar danh sách bài học (giữ nguyên vị trí khi chuyển bài).
- **Lesson Page**: Nội dung chi tiết từng bài.

## 13. Interview
### Core Q&A
1. Q: Layout và Template khác nhau điểm nào?
   A: Layout duy trì trạng thái và không mount lại khi chuyển trang. Template tạo một instance mới hoàn toàn mỗi khi điều hướng, thường dùng cho các logic cần reset như animation vào/ra hoặc `useEffect` theo dõi trang.

### Scenario
1. Q: Làm thế nào để thay đổi Metadata (tiêu đề trang) trong một Layout lồng nhau?
   A: Bạn có thể export một object `metadata` hoặc hàm `generateMetadata` từ bất kỳ Layout hay Page nào. Next.js sẽ tự động hợp nhất (merge) chúng theo thứ tự từ Root đến Page.

## 14. References
- Next.js Layouts: https://nextjs.org/docs/app/building-your-application/routing/pages-and-layouts
- Templates: https://nextjs.org/docs/app/api-reference/file-conventions/template

## 15. Real-world Code
Xem cấu trúc thư mục của dự án `taxonomy` (shadcn/ui) để học cách họ tổ chức nhiều tầng layout cho Dashboard và Marketing site.

## 16. Community
- Blog: "Why you should use Layouts in Next.js 13" - Vercel.
- Discussion: GitHub Discussions về việc truyền dữ liệu từ Layout xuống Page.
