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
Link và Navigation trong Next.js là hệ thống cho phép người dùng di chuyển giữa các trang (routes). Thành phần chính bao gồm component `<Link>` (để điều hướng khai báo) và hook `useRouter` (để điều hướng theo lập trình), kết hợp với cơ chế tối ưu hóa như prefetching.

## 2. Why
Trong các ứng dụng React truyền thống, việc dùng thẻ `<a>` sẽ khiến trình duyệt load lại toàn bộ trang (hard refresh), làm mất state và chậm trải nghiệm. Next.js cung cấp hệ thống navigation thông minh để thực hiện "Soft Navigation" (chỉ load phần thay đổi), giúp ứng dụng nhanh như ứng dụng di động.

## 3. Mental Model
Hãy tưởng tượng trang web của bạn là một bảo tàng. Thẻ `<a>` thông thường giống như việc bạn phải ra khỏi bảo tàng, đi qua cổng chính và mua vé lại từ đầu để sang phòng bên cạnh. Hệ thống Navigation của Next.js giống như một cánh cửa thông minh giữa các phòng: bạn chỉ việc bước qua, không cần check-in lại, và thậm chí nhân viên đã chuẩn bị sẵn ánh sáng ở phòng tiếp theo khi thấy bạn đang tiến gần cửa (prefetching).

## 4. Where it fits
User Interaction -> `<Link>` / `useRouter` -> Next.js Router -> Prefetching Check -> Soft/Hard Navigation Decision -> UI Update.

## 5. When to use
- Di chuyển giữa các trang nội bộ trong ứng dụng Next.js.
- Khi cần tối ưu tốc độ chuyển trang thông qua việc tải trước dữ liệu.
- Khi cần thay đổi URL dựa trên logic (ví dụ: sau khi submit form thành công).

## 6. When NOT to use
- Điều hướng đến các trang web bên ngoài (dùng thẻ `<a>` thuần).
- Khi muốn thực hiện "Hard Refresh" để xóa sạch bộ nhớ và state của ứng dụng.
- Đối với các liên kết tải xuống file (download links).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Chuyển trang tức thì (Soft Navigation) | Tăng mức tiêu thụ data do prefetching |
| Giữ vững state của Layout | Phức tạp hơn khi xử lý các scroll position tùy chỉnh |
| Tự động tải trước (Prefetching) | Có thể làm chậm các thiết bị cực yếu nếu lạm dụng quá nhiều link |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Thẻ `<a>` thuần | Khiến trang load lại hoàn toàn, không có prefetching. |
| `window.location` | Điều hướng ở tầng trình duyệt, làm mất state của React. |

## 9. How
```tsx
"use client";

import Link from 'next/link';
import { useRouter, usePathname } from 'next/navigation';

export default function Navbar() {
  const router = useRouter();
  const pathname = usePathname();

  return (
    <nav>
      {/* Khai báo Link */}
      <Link href="/dashboard" className={pathname === '/dashboard' ? 'active' : ''}>
        Dashboard
      </Link>

      {/* Điều hướng bằng lập trình */}
      <button onClick={() => router.push('/settings')}>
        Go to Settings
      </button>
    </nav>
  );
}
```

## 10. Production concerns
### Scaling
Next.js chỉ prefetch các link nằm trong viewport. Tuy nhiên, trên trang có quá nhiều link (như danh mục sản phẩm), nên cân nhắc dùng `prefetch={false}` cho các link ít quan trọng.

### Failure
Nếu mạng bị ngắt trong khi đang điều hướng, Next.js sẽ cố gắng fallback hoặc ném lỗi thông qua `error.js`.

### Monitoring
Sử dụng `useReportWebVitals` để theo dõi thời gian chuyển trang thực tế của người dùng (Next.js Navigation Timing).

## 11. Common mistakes
- Mistake: Sử dụng `useRouter` từ `next/router` thay vì `next/navigation` trong App Router.
  Fix: Luôn dùng `next/navigation` cho các dự án App Router.

- Mistake: Dùng thẻ `<a>` bên trong `<Link>` (Lỗi cũ từ phiên bản trước).
  Fix: Từ Next.js 13+, `<Link>` đã tự render thẻ `<a>`, không cần bọc thủ công.

## 12. Sample project
Tạo một thanh Side Navigation cho trang quản trị, tự động highlight mục hiện tại dựa trên `usePathname` và thực hiện chuyển hướng bảo mật sau khi logout.

## 13. Interview
### Core Q&A
1. Q: "Soft Navigation" trong Next.js là gì?
   A: Là cơ chế điều hướng mà Next.js chỉ tải các phân đoạn (segments) thay đổi của route và cập nhật URL mà không cần tải lại toàn bộ trang. Điều này giúp giữ nguyên state ở các Shared Layouts.

### Scenario
1. Q: Làm thế nào để điều hướng mà không làm thay đổi scroll position?
   A: Bạn có thể truyền thuộc tính `scroll={false}` vào component `<Link>` hoặc options của `router.push()`.

## 14. References
- Next.js Linking and Navigating: https://nextjs.org/docs/app/building-your-application/routing/linking-and-navigating
- useRouter API: https://nextjs.org/docs/app/api-reference/functions/use-router

## 15. Real-world Code
Học cách các thư viện UI như `Headless UI` hoặc `Radix UI` tích hợp với Next.js Link để đảm bảo tính accessibility (A11y).

## 16. Community
- Discussion: "Prefetching behavior in Next.js 13+" trên GitHub.
- Blog: "Making Navigation Fast" - Vercel Engineering Blog.
