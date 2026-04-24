---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/react"
  - "#topic/performance"
related:
  - "[[react-lifecycle]]"
  - "[[useState]]"
---

## 1. What
`useRef` là một React Hook trả về một đối tượng ref có thể thay đổi, thuộc tính `.current` của nó được khởi tạo bằng đối số truyền vào. Giá trị của ref sẽ được bảo toàn trong suốt vòng đời của component và việc thay đổi nó sẽ không gây ra việc re-render component.

## 2. Why
Trong React, dữ liệu thường được quản lý theo mô hình khai báo (declarative). Tuy nhiên, có những trường hợp bạn cần can thiệp trực tiếp vào phần tử DOM (imperative) hoặc cần lưu trữ một giá trị mà khi thay đổi nó, bạn không muốn giao diện phải vẽ lại (ví dụ: timer ID, giá trị cũ của props). `useRef` cung cấp một "hộp chứa" bền vững cho những trường hợp này.

## 3. Mental Model
Hãy tưởng tượng component của bạn như một **người đang suy nghĩ (render)**.
- `useState` giống như những suy nghĩ trong đầu: khi suy nghĩ thay đổi, sắc mặt người đó thay đổi (UI render lại).
- `useRef` giống như một **tờ giấy ghi chú** dán lên trán người đó. Người đó có thể viết hoặc đọc thông tin trên tờ giấy đó bất cứ lúc nào mà không cần phải thay đổi luồng suy nghĩ hay sắc mặt của mình.

## 4. Where it fits
Component Definition -> const ref = useRef(initialValue) -> Access ref.current in Effects or Event Handlers -> DOM attribute `ref={ref}`.

## 5. When to use
- Tương tác với DOM: Tự động focus vào input, cuộn trang (scroll), đo đạc kích thước phần tử.
- Lưu trữ giá trị bền vững: Lưu ID của `setInterval`/`setTimeout`, lưu giá trị cũ (previous state) để so sánh.
- Tích hợp với các thư viện không phải React (như D3.js, Google Maps, video players).

## 6. When NOT to use
- Đừng dùng `useRef` để thay thế `useState` cho những thứ cần hiển thị lên màn hình.
- Tránh thay đổi hoặc đọc `ref.current` trực tiếp trong quá trình render (body của function component) vì nó phá vỡ tính thuần khiết của React. Chỉ nên thao tác trong `useEffect` hoặc Event Handlers.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Truy cập trực tiếp vào phần tử DOM | Thoát ly khỏi mô hình khai báo của React, dễ gây ra code khó hiểu |
| Lưu giá trị mà không gây re-render (tối ưu performance) | Dễ bị lạm dụng để điều khiển UI thủ công (imperative UI) |
| Bảo toàn dữ liệu qua các lần render | Phải quản lý thuộc tính `.current` một cách thủ công |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| useState | Dùng khi giá trị thay đổi cần cập nhật UI. |
| Callback Ref | Dùng khi cần thực hiện logic ngay khi phần tử DOM được gắn (mount) hoặc tháo (unmount). |

## 9. How
Cách sử dụng `useRef` với TypeScript để truy cập DOM:

```tsx
import { useRef, useEffect } from 'react';

const AutoFocusInput = () => {
  // 1. Khai báo ref với kiểu dữ liệu của phần tử HTML
  const inputRef = useRef<HTMLInputElement>(null);

  const handleClick = () => {
    // 3. Truy cập thông qua thuộc tính .current
    // Cần kiểm tra null (Optional Chaining)
    inputRef.current?.focus();
  };

  useEffect(() => {
    // Tự động focus khi component mount
    inputRef.current?.focus();
  }, []);

  return (
    <div>
      {/* 2. Gắn ref vào phần tử JSX */}
      <input ref={inputRef} type="text" />
      <button onClick={handleClick}>Focus Input</button>
    </div>
  );
};
```

## 10. Production concerns
### Memory
Giá trị trong ref sẽ tồn tại cho đến khi component bị unmount. Cần chú ý dọn dẹp (cleanup) các tài nguyên như timer hoặc kết nối socket được lưu trong ref.

### Failure
Luôn khởi tạo `useRef(null)` cho các phần tử DOM và sử dụng kiểm tra null (`?.`) vì trong lần render đầu tiên, phần tử DOM chưa tồn tại.

## 11. Common mistakes
- Mistake: Thay đổi `ref.current` trong body của component để cập nhật UI.
  Fix: UI sẽ không cập nhật. Hãy dùng `useState` nếu muốn thay đổi giao diện.

- Mistake: Quên truy cập qua thuộc tính `.current`.
  Fix: `inputRef` là một object `{ current: ... }`, bạn không thể gọi `inputRef.focus()`.

## 12. Sample project
Xây dựng một `VideoPlayer` component: Sử dụng `useRef` để lấy instance của thẻ `<video>`, sau đó tạo các nút tùy chỉnh Play/Pause gọi trực tiếp phương thức `.play()` và `.pause()` của DOM.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt lớn nhất giữa `useRef` và `useState` là gì?
   A: `useState` gây ra re-render khi giá trị thay đổi, dùng cho dữ liệu hiển thị. `useRef` không gây re-render, dùng để lưu trữ dữ liệu bền vững hoặc truy cập DOM.

### Scenario
1. Q: Làm thế nào để lấy giá trị trước đó (previous value) của một prop?
   A: Tôi sẽ dùng một `useRef`. Trong `useEffect`, tôi gán giá trị hiện tại của prop vào `ref.current`. Vì `useEffect` chạy sau khi render, nên giá trị trong `ref.current` lúc render vẫn là giá trị của lần render trước đó.

## 14. References
- React Ref Documentation: https://react.dev/learn/referencing-values-with-refs
- useRef API: https://react.dev/reference/react/useRef

## 15. Real-world Code
Kiểm tra cách các thư viện animation như `Framer Motion` hoặc `GSAP` sử dụng refs để điều khiển trực tiếp các thuộc tính CSS của phần tử DOM mà không làm chậm React.

## 16. Community
- YouTube: "Everything about useRef" - Web Dev Simplified.
- Blog: "Robin Wieruch - How to use useRef in React".
