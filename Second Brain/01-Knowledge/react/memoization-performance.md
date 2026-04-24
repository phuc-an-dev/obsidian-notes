---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/react"
  - "#topic/performance"
related:
  - "[[react-lifecycle]]"
  - "[[use-ref]]"
---

## 1. What
Memoization trong React là kỹ thuật tối ưu hóa bằng cách ghi nhớ kết quả của các lần tính toán hoặc các component đã render. Nó bao gồm: `React.memo` (cho components), `useMemo` (cho giá trị), và `useCallback` (cho functions).

## 2. Why
Mặc định, khi một component cha re-render, toàn bộ con của nó cũng re-render. Với các tính toán đắt đỏ hoặc cây component sâu, việc này gây lãng phí tài nguyên và làm giảm FPS của ứng dụng. Memoization giúp bỏ qua các bước render hoặc tính toán không cần thiết nếu dữ liệu đầu vào (props/dependencies) không thay đổi.

## 3. Mental Model
Hãy tưởng tượng bạn là một **người thợ làm bánh**.
- **Mặc định**: Khách hỏi gì bạn cũng phải vào bếp nhào bột, nướng bánh từ đầu (Re-render).
- **Memoization**: Bạn có một chiếc tủ trưng bày. Nếu khách hỏi loại bánh đã làm sẵn (Dữ liệu không đổi), bạn chỉ việc lấy trong tủ ra đưa cho khách mà không cần bật lò nướng (Skip render/calculation). Bạn chỉ làm lại bánh mới khi công thức hoặc yêu cầu của khách thay đổi.

## 4. Where it fits
State/Props Change -> **Memoization Check (Shallow Comparison)** -> [Calculate/Render] OR [Return Cached Result].

## 5. When to use
- Khi có các phép tính toán cực kỳ nặng (xử lý mảng hàng vạn phần tử, thuật toán phức tạp) -> `useMemo`.
- Khi truyền hàm callback xuống các component con đã được bọc bởi `React.memo` -> `useCallback`.
- Các component hiển thị dữ liệu tĩnh lớn hoặc không thay đổi thường xuyên -> `React.memo`.

## 6. When NOT to use
- Đừng lạm dụng cho mọi component hoặc mọi giá trị đơn giản (như cộng 2 số). Việc lưu trữ cache và kiểm tra dependencies cũng tốn tài nguyên.
- Khi props truyền vào luôn thay đổi ở mỗi lần render (như render một mảng mới mỗi lần).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Tăng tốc độ phản hồi của UI (giảm lag) | Tốn bộ nhớ để lưu trữ cache |
| Giảm tải cho CPU | Code trở nên phức tạp và khó đọc hơn |
| Ngăn chặn các hiệu ứng phụ (side effects) không mong muốn | Dễ gây ra lỗi nếu quên khai báo dependencies |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Component Composition | Cách tốt nhất để giảm re-render mà không cần dùng đến memoization. |
| Web Workers | Dùng cho các tác vụ tính toán cực nặng bên ngoài luồng chính của UI. |

## 9. How
Bộ ba tối ưu hiệu năng:

```tsx
import React, { useState, useMemo, useCallback } from 'react';

// 1. React.memo: Chỉ render lại nếu text thay đổi
const ChildComponent = React.memo(({ onClick, text }: any) => {
  console.log('Child Render');
  return <button onClick={onClick}>{text}</button>;
});

const Parent = () => {
  const [count, setCount] = useState(0);

  // 2. useMemo: Chỉ tính toán lại khi count thay đổi
  const expensiveValue = useMemo(() => {
    return count * 1000; // Giả sử là phép tính nặng
  }, [count]);

  // 3. useCallback: Giữ tham chiếu hàm không đổi giữa các lần render
  const handleClick = useCallback(() => {
    console.log('Clicked');
  }, []); // Mảng rỗng: hàm không bao giờ tạo mới

  return (
    <div>
      <p>Value: {expensiveValue}</p>
      <button onClick={() => setCount(count + 1)}>Inc</button>
      <ChildComponent text="Click me" onClick={handleClick} />
    </div>
  );
};
```

## 10. Production concerns
### Performance
Sử dụng **React DevTools Profiler** để đo đạc xem component nào thực sự cần tối ưu. Không nên tối ưu dựa trên cảm tính (Premature Optimization).

### Monitoring
Theo dõi các chỉ số FPS trên thiết bị người dùng cuối thông qua các thư viện Web Vitals để đảm bảo trải nghiệm mượt mà.

## 11. Common mistakes
- Mistake: Dùng `useCallback` nhưng không dùng `React.memo` cho component con.
  Fix: `useCallback` chỉ có tác dụng khi nó giúp component con tránh re-render (thông qua so sánh tham chiếu).

- Mistake: Khai báo thiếu dependency trong `useMemo`/`useCallback`.
  Fix: Luôn sử dụng linter `eslint-plugin-react-hooks` để đảm bảo mảng dependency đầy đủ.

## 12. Sample project
Xây dựng một ứng dụng Filter danh sách lớn: State quản lý text tìm kiếm và mảng 10.000 items. Sử dụng `useMemo` để lọc danh sách và `React.memo` cho từng item để đảm bảo việc gõ phím tìm kiếm không bị delay.

## 13. Interview
### Core Q&A
1. Q: So sánh tham chiếu (Reference Equality) là gì?
   A: Trong JS, `[] === []` là false. React dùng so sánh này để kiểm tra props. Memoization giúp giữ cho tham chiếu của array/object/function là duy nhất (true) qua các lần render nếu dữ liệu không đổi.

### Scenario
1. Q: Tại sao bạn không nên bọc tất cả component trong `React.memo`?
   A: Vì việc so sánh props ở mỗi lần render cũng tốn CPU. Với các component đơn giản hoặc các component mà props luôn thay đổi, việc bọc memo sẽ làm ứng dụng chậm hơn do phải thực hiện thêm bước so sánh vô ích.

## 14. References
- React Performance: https://react.dev/learn/render-and-commit
- useMemo API: https://react.dev/reference/react/useMemo
- useCallback API: https://react.dev/reference/react/useCallback

## 15. Real-world Code
Nghiên cứu mã nguồn của `React Virtualized` hoặc `React Window` để xem cách họ tối ưu render cho danh sách cực lớn.

## 16. Community
- YouTube: "Why React Re-renders" - Josh W. Comeau.
- Blog: "Kent C. Dodds - When to useMemo and useCallback".
