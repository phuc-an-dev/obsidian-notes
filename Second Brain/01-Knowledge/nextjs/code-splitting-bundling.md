---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/react"
  - "#topic/performance"
related:
  - "[[nextjs]]"
  - "[[server-vs-client-components]]"
---

## 1. What
Code Splitting là kỹ thuật chia nhỏ bundle JavaScript lớn thành các phần nhỏ hơn (chunks), chỉ tải về trình duyệt khi cần thiết. Bundling là quá trình gộp các file mã nguồn và thư viện vào một vài file JavaScript tối ưu để trình duyệt xử lý hiệu quả. Next.js thực hiện việc này tự động dựa trên từng route.

## 2. Why
Một ứng dụng React lớn có thể có bundle size lên đến vài MB. Nếu tải toàn bộ JS ngay lần đầu, người dùng sẽ phải chờ rất lâu (TBT - Total Blocking Time cao). Code Splitting giúp giảm lượng JS ban đầu, tăng tốc độ tương tác và tiết kiệm băng thông cho người dùng.

## 3. Mental Model
Hãy tưởng tượng ứng dụng của bạn là một bộ bách khoa toàn thư. Thay vì bắt người dùng mang theo cả bộ sách 20 tập (Big Bundle), bạn chỉ đưa cho họ đúng tập sách mà họ đang muốn đọc (Page-based Splitting). Nếu họ muốn xem một sơ đồ phức tạp trong đó, bạn mới đưa cho họ tờ phụ bản bổ sung (Dynamic Import).

## 4. Where it fits
Source Code -> Webpack / Turbopack (Build time) -> Route Chunks & Shared Chunks -> Browser (Runtime loading).

## 5. When to use
- Khi có các component nặng chỉ dùng ở một vài trang (ví dụ: Google Maps, Chart.js).
- Khi có các thư viện bên thứ ba lớn không cần thiết cho quá trình render ban đầu.
- Khi muốn tối ưu hóa tiến trình tải trang (Lazy loading).

## 6. When NOT to use
- Đừng chia nhỏ các component quá bé (dưới vài KB) vì chi phí thiết lập request HTTP mới có thể lớn hơn lợi ích mang lại.
- Tránh lazy load các component nằm trong khung hình đầu tiên (Above the fold) vì sẽ gây hiện tượng nháy UI.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Tăng tốc độ tải trang ban đầu (FCP, LCP) | Tăng số lượng HTTP requests |
| Giảm lượng JS cần parse và execute | Có thể gây trễ khi người dùng tương tác lần đầu với phần được tách |
| Tối ưu hóa bộ nhớ trình duyệt | Phức tạp hơn trong việc quản lý trạng thái loading |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Manual Splitting | Tự cấu hình Webpack, cực kỳ phức tạp và dễ sai sót. |
| React.lazy | Giải pháp chuẩn của React, nhưng Next.js `next/dynamic` mạnh mẽ hơn (hỗ trợ SSR). |

## 9. How
```tsx
import dynamic from 'next/dynamic'

// Tách nhỏ component nặng và hiển thị loading khi đang tải
const HeavyChart = dynamic(() => import('../components/HeavyChart'), {
  loading: () => <p>Loading chart...</p>,
  ssr: false, // Tùy chọn tắt render ở server nếu chỉ dùng Browser API
})

export default function Dashboard() {
  return (
    <div>
      <h1>My Dashboard</h1>
      <HeavyChart />
    </div>
  )
}
```

## 10. Production concerns
### Scaling
Next.js tự động quản lý việc đặt tên và phiên bản (versioning) cho các chunks, giúp việc caching ở CDN và trình duyệt hoạt động hiệu quả khi bạn cập nhật code.

### Failure
Nếu mạng bị lỗi khi đang tải một chunk động, Next.js có thể ném lỗi. Cần bọc các component này trong Error Boundary.

### Monitoring
Sử dụng thư viện `@next/bundle-analyzer` để trực quan hóa kích thước các bundle và xác định phần nào đang chiếm dung lượng lớn nhất.

## 11. Common mistakes
- Mistake: Import động ngay tại top-level của file thay vì bên trong component.
  Fix: `next/dynamic` nên được dùng để khai báo component ở ngoài, nhưng bản thân việc import sẽ được thực thi khi component đó được render.

- Mistake: Không sử dụng `{ ssr: false }` cho các component dùng các đối tượng chỉ có ở trình duyệt (như `window`, `document`).
  Fix: Luôn tắt SSR cho các thư viện thao tác trực tiếp với DOM mà không hỗ trợ môi trường Node.js.

## 12. Sample project
Tạo một trang landing page có chứa một Modal video nặng. Chỉ khi người dùng click vào nút "Xem Video", mã nguồn của trình phát video mới được tải về và hiển thị.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa `next/dynamic` và `React.lazy` là gì?
   A: `next/dynamic` là một wrapper của `React.lazy` được tối ưu cho Next.js, hỗ trợ render phía server (SSR) và cho phép hiển thị component loading một cách dễ dàng hơn.

### Scenario
1. Q: Bạn làm thế nào để giảm bundle size của một trang sử dụng thư viện `lodash` cực nặng?
   A: Thay vì import cả thư viện (`import _ from 'lodash'`), tôi sẽ chỉ import function cần dùng (`import debounce from 'lodash/debounce'`) hoặc sử dụng dynamic import để tải nó chỉ khi thực sự cần thực thi logic đó.

## 14. References
- Next.js Optimizing Bundles: https://nextjs.org/docs/app/building-your-application/optimizing/scripts
- next/dynamic API: https://nextjs.org/docs/app/api-reference/functions/dynamic

## 15. Real-world Code
Kiểm tra cấu trúc file trong thư mục `.next/static/chunks` sau khi chạy lệnh `npm run build` để thấy cách Next.js phân tách mã nguồn.

## 16. Community
- Tool: Webpack Bundle Analyzer.
- Blog: "How Next.js dynamic imports work under the hood".
