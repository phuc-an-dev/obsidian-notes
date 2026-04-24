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
---

## 1. What
Server Components và Client Components là hai loại component trong Next.js App Router. Server Components (RSC) được render hoàn toàn trên server, trong khi Client Components được render trên server (để lấy HTML ban đầu) và sau đó được "hydrate" (kích hoạt tính tương tác) trên trình duyệt.

## 2. Why
Trước đây, React render mọi thứ ở client, dẫn đến bundle JS lớn và performance kém trên thiết bị yếu. Việc tách biệt Server/Client Components giúp giảm lượng JavaScript gửi xuống trình duyệt, tận dụng sức mạnh tính toán của server cho việc truy vấn dữ liệu và tăng tính bảo mật cho các logic nhạy cảm.

## 3. Mental Model
Hãy tưởng tượng một nhà hàng cao cấp:
- **Server Components**: Là đầu bếp trong bếp. Họ có quyền truy cập trực tiếp vào kho thực phẩm (database), xử lý các món ăn phức tạp và chỉ đưa món ăn đã hoàn thành ra bàn. Khách hàng không thấy được quá trình nấu nướng (logic code).
- **Client Components**: Là nhân viên phục vụ tại bàn. Họ tương tác trực tiếp với khách (click, scroll, form), xử lý các yêu cầu tức thời của khách hàng ngay tại chỗ.

## 4. Where it fits
Server Component (Gốc) -> Client Component (Lá) -> Server Component (truyền qua children/props).
*Lưu ý: Bạn không thể import trực tiếp Server Component vào Client Component.*

## 5. When to use
- **Server Components (Mặc định)**:
  - Khi cần fetch dữ liệu từ database hoặc API.
  - Khi chứa thông tin nhạy cảm (API keys, tokens).
  - Khi component lớn nhưng không cần tương tác (giảm bundle size).
- **Client Components (Dùng `"use client"`)**:
  - Khi cần interactivity: `onClick`, `onChange`.
  - Khi dùng Hooks: `useState`, `useEffect`, `useContext`.
  - Khi dùng Browser APIs: `window`, `localStorage`, `canvas`.

## 6. When NOT to use
- Đừng dùng `"use client"` cho toàn bộ trang nếu chỉ có một nút bấm cần tương tác.
- Đừng dùng Server Component nếu bạn cần cập nhật UI ngay lập tức dựa trên hành động người dùng mà không muốn reload trang.

## 7. Trade-offs
| Đặc tính | Server Components | Client Components |
|----------|-------------------|-------------------|
| Bundle JS | 0 (Không gửi JS xuống client) | Có gửi JS xuống client |
| Data Fetching | Trực tiếp (async/await) | Qua useEffect hoặc thư viện (Query) |
| Interactivity | Không | Có |
| Access to DB | Có | Không |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Next.js Pages Router | Mọi component đều là Client Component (hydrate toàn bộ). |
| Remix | Sử dụng Loaders/Actions để tách biệt logic server nhưng UI vẫn là hydrate-heavy. |

## 9. How
```tsx
// app/page.tsx (Mặc định là Server Component)
import MyClientComponent from './MyClientComponent';

export default async function Page() {
  const data = await fetch('https://api.com/data').then(res => res.json());

  return (
    <main>
      <h1>Server Component Data: {data.title}</h1>
      <MyClientComponent initialCount={0} />
    </main>
  );
}

// app/MyClientComponent.tsx
"use client"; // Đánh dấu là Client Component
import { useState } from 'react';

export default function MyClientComponent({ initialCount }: { initialCount: number }) {
  const [count, setCount] = useState(initialCount);
  return <button onClick={() => setCount(count + 1)}>Count: {count}</button>;
}
```

## 10. Production concerns
### Scaling
Server Components giúp giảm tải cho thiết bị người dùng nhưng tốn tài nguyên CPU trên server. Cần kết hợp với caching hiệu quả.

### Failure
Nếu Server Component bị lỗi trong quá trình fetch data, toàn bộ cây component đó có thể bị ảnh hưởng. Hãy dùng `error.js` và `Suspense` để cô lập lỗi.

### Monitoring
Theo dõi kích thước Bundle JS bằng `next-bundle-analyzer` để đảm bảo không lạm dụng Client Components.

## 11. Common mistakes
- Mistake: Import Server Component vào file có dòng `"use client"`.
  Fix: Truyền Server Component vào Client Component thông qua `children` hoặc props.

- Mistake: Quên rằng Client Component vẫn render HTML ở server lần đầu.
  Fix: Kiểm tra `typeof window !== 'undefined'` nếu dùng các API chỉ có ở trình duyệt trong body component.

## 12. Sample project
Tạo một trang chi tiết sản phẩm:
- Server Component fetch thông tin sản phẩm và reviews.
- Client Component xử lý việc thêm vào giỏ hàng và tab chuyển đổi mô tả.

## 13. Interview
### Core Q&A
1. Q: Tại sao Server Component giúp ứng dụng nhanh hơn?
   A: Vì nó loại bỏ mã nguồn của component đó khỏi bundle JavaScript gửi xuống client, giúp giảm thời gian parse/execute JS và tải trang nhanh hơn trên các thiết bị yếu.

### Scenario
1. Q: Bạn có một trang danh sách bài viết. Bạn muốn có thanh tìm kiếm lọc bài viết ngay lập tức. Bạn sẽ tổ chức components như thế nào?
   A: Toàn bộ trang và danh sách bài viết là Server Component. Thanh tìm kiếm là một Client Component riêng biệt. Khi nhập liệu, Client Component này có thể cập nhật URL (search params) và Next.js sẽ tự động re-render các Server Component tương ứng một cách mượt mà.

## 14. References
- React Server Components: https://react.dev/reference/react/use-server
- Next.js RSC Docs: https://nextjs.org/docs/app/building-your-application/rendering/server-components

## 15. Real-world Code
Hầu hết các thư viện UI hiện đại như `shadcn/ui` đều sử dụng `"use client"` cho các component phức tạp như Dialog, Dropdown vì chúng cần tương tác DOM.

## 16. Community
- YouTube: "Rethinking React" bởi Dan Abramov.
- Blog: "Josh W. Comeau - Making Sense of React Server Components".
