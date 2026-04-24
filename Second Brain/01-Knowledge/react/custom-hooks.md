---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/react"
  - "#topic/state-management"
related:
  - "[[react-lifecycle]]"
  - "[[useState]]"
---

## 1. What
Custom Hooks là các hàm JavaScript có tên bắt đầu bằng "use", cho phép bạn trích xuất logic của component vào các hàm có thể tái sử dụng. Chúng có thể gọi các hooks khác của React (như `useState`, `useEffect`) bên trong chính nó.

## 2. Why
Khi ứng dụng phát triển, nhiều component có thể chia sẻ cùng một logic (ví dụ: fetch dữ liệu, theo dõi trạng thái mạng, xử lý form). Custom Hooks giúp tránh lặp lại code (DRY - Don't Repeat Yourself), làm cho các component trở nên gọn gàng hơn và dễ dàng kiểm thử logic một cách độc lập.

## 3. Mental Model
Hãy tưởng tượng Custom Hooks như các **module tiện ích** hoặc các **bộ công cụ chuyên dụng**.
Thay vì mỗi người thợ (Component) phải tự xây dựng máy khoan, máy cắt từ đầu, bạn tạo ra một bộ công cụ tiêu chuẩn (Custom Hook). Khi cần dùng, người thợ chỉ cần mượn bộ công cụ đó về và sử dụng, đảm bảo mọi người thợ đều dùng chung một tiêu chuẩn kỹ thuật.

## 4. Where it fits
React Component -> Call **Custom Hook** -> Hook Internal Logic (State/Effect) -> Return Data/Functions -> Component Rendering.

## 5. When to use
- Khi có logic phức tạp liên quan đến vòng đời (useEffect) lặp lại ở nhiều nơi.
- Khi muốn tách biệt logic xử lý dữ liệu (Business Logic) ra khỏi logic hiển thị (UI Logic).
- Khi muốn bọc các thư viện bên thứ ba (như Firebase, Socket.io) để dùng trong React một cách idiomatic.

## 6. When NOT to use
- Các logic đơn giản không liên quan đến React state hoặc effects (chỉ cần dùng hàm JavaScript thường).
- Khi logic đó chỉ dùng duy nhất ở một component và không quá phức tạp (tránh over-engineering).
- Đừng dùng Custom Hook để chứa các component JSX (mặc dù có thể nhưng không khuyến khích).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Tái sử dụng logic cực cao | Có thể làm tăng độ phức tạp khi debug (deep nesting) |
| Component UI sạch sẽ, dễ đọc | Khó khăn trong việc chia sẻ state giữa các lần gọi khác nhau |
| Dễ dàng viết Unit Test cho logic | Đòi hỏi phải hiểu rõ các quy tắc của Hooks (Rules of Hooks) |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Higher-Order Components (HOC) | Cách cũ, dễ gây ra "wrapper hell" và khó track props. |
| Render Props | Linh hoạt nhưng làm cây JSX trở nên lồng nhau và khó đọc. |

## 9. How
Ví dụ về `useFetch` hook:

```tsx
import { useState, useEffect } from 'react';

function useFetch<T>(url: string) {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    const fetchData = async () => {
      try {
        const response = await fetch(url);
        const json = await response.json();
        setData(json);
      } catch (err) {
        setError('Failed to fetch data');
      } finally {
        setLoading(false);
      }
    };
    fetchData();
  }, [url]);

  return { data, loading, error };
}

// Cách dùng trong component
const MyComponent = () => {
  const { data, loading } = useFetch<{name: string}>('/api/user');
  if (loading) return <p>Loading...</p>;
  return <h1>{data?.name}</h1>;
}
```

## 10. Production concerns
### Performance
Cẩn thận với việc tạo ra các object/array mới trong giá trị trả về của hook. Nên dùng `useMemo` hoặc `useCallback` nếu giá trị trả về được dùng làm dependency cho các hooks khác ở phía component.

### Maintenance
Luôn tuân thủ quy tắc đặt tên bắt đầu bằng `use` để các công cụ linter có thể kiểm tra các vi phạm quy tắc Hooks.

## 11. Common mistakes
- Mistake: Gọi Custom Hook bên trong vòng lặp hoặc câu lệnh điều kiện.
  Fix: Luôn gọi hook ở cấp cao nhất của component (Top-level).

- Mistake: Quên truyền dependencies vào `useEffect` bên trong custom hook.
  Fix: Sử dụng eslint-plugin-react-hooks để đảm bảo tính chính xác của mảng dependency.

## 12. Sample project
Xây dựng bộ hooks `useLocalStorage`, `useWindowSize`, và `useAuth` để quản lý các tính năng phổ biến trong một ứng dụng web hiện đại.

## 13. Interview
### Core Q&A
1. Q: Custom Hook có chia sẻ state giữa các component cùng sử dụng nó không?
   A: Không. Mỗi lần gọi Custom Hook sẽ tạo ra một instance state hoàn toàn độc lập cho component đó. Nếu muốn chia sẻ state, phải dùng Context API hoặc State Management (Redux/Zustand).

### Scenario
1. Q: Làm thế nào để đảm bảo Custom Hook của bạn không bị leak bộ nhớ khi component bị unmount?
   A: Tôi sẽ sử dụng hàm cleanup bên trong `useEffect` (ví dụ: `return () => clearInterval(id)` hoặc abort fetch request) để dọn dẹp các tài nguyên không còn cần thiết.

## 14. References
- Reusing Logic with Custom Hooks: https://react.dev/learn/reusing-logic-with-custom-hooks
- useHooks-ts (Thư viện mẫu): https://usehooks-ts.com/

## 15. Real-world Code
Nghiên cứu các thư viện như `react-use` hoặc `ahooks` để xem các pattern chuyên sâu về Custom Hooks.

## 16. Community
- YouTube: "The Power of Custom Hooks" - Dave Gray.
- Blog: "Kent C. Dodds - Myths about Custom Hooks".
