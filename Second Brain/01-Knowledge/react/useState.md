---
created: 2026-04-17
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/react"
  - "#topic/state-management"
related:
  - "[[useReducer]]"
---

## 1. What

`useState` là một React Hook giúp thêm state vào functional component. Nó trả về một mảng gồm hai phần tử: giá trị state hiện tại và một hàm để cập nhật giá trị đó. Mỗi lần state thay đổi, component sẽ re-render với giá trị state mới.

## 2. Why

Trước kia, chỉ class component mới có thể quản lý state thông qua `this.state`. Functional component không thể giữ lại state giữa các lần render, khiến chúng chỉ dùng để hiển thị dữ liệu tĩnh (presentational component). `useState` giải quyết vấn đề này bằng cách cho phép functional component có khả năng quản lý state riêng, giúp code trở nên đơn giản hơn và dễ maintain hơn.

## 3. Mental Model

Hãy tưởng tượng `useState` như một **biến ghi chép trong cuốn nhật ký cá nhân**. Mỗi lần bạn cần cập nhật thông tin, bạn không thay đổi trực tiếp trên trang cũ mà viết một trang mới và quay lại từ đầu để đọc. React cũng vậy: mỗi lần `setState`, React sẽ tạo một "version mới" của component, re-render và cập nhật DOM. State không thay đổi trực tiếp mà được thay thế bằng một state mới, giúp React biết khi nào cần cập nhật giao diện.

## 4. Where it fits

Trong vòng đời React component:
```
Component render (khởi tạo state) 
  -> User tương tác (click, input, ...) 
  -> Gọi setState (updater function) 
  -> React re-render component 
  -> DOM cập nhật 
  -> Component render lại với state mới
```

`useState` nằm ở tâm của quy trình này: nó lưu trữ dữ liệu (state) và cung cấp cách cập nhật dữ liệu.

## 5. When to use

- Cần lưu lại dữ liệu giữa các lần render (ví dụ: giá trị input, số lượng click, trạng thái modal)
- Dữ liệu thay đổi thường xuyên và ảnh hưởng đến giao diện
- Quản lý state đơn giản, không phức tạp (state complexity thấp)
- Muốn giữ logic state gần với component sử dụng nó (co-location)

## 6. When NOT to use

- State quá phức tạp với nhiều sub-states và update logic (dùng `useReducer` thay thế)
- Cần chia sẻ state giữa nhiều component (dùng Context hoặc global state management)
- State không thay đổi sau khi khởi tạo (đặt giá trị mặc định không cần `useState`)
- Dữ liệu không ảnh hưởng đến render (dùng `useRef` thay thế)
- Đang trong vòng lặp hoặc điều kiện (vi phạm Rules of Hooks)

## 7. Trade-offs

| Pros | Cons |
|------|------|
| Đơn giản, dễ hiểu và sử dụng | Mỗi `setState` sẽ trigger re-render toàn component |
| Code ngắn gọn hơn class component | Không tối ưu nếu state quá phức tạp |
| Dễ test vì functional component | Batching state updates có thể gây confusion |
| Khuyến khích co-location (logic gần data) | Nếu dùng sai cách, có thể gây memory leak |
| State cũ và mới được giữ riêng biệt | Closure trong setState có thể gây stale state |

## 8. Alternatives

| Option | So sánh |
|--------|---------|
| `useReducer` | Tốt hơn cho state phức tạp với nhiều sub-states; dễ debug vì tách rời logic update |
| Context API | Dùng để chia sẻ state giữa nhiều component, nhưng không tối ưu cho state thay đổi thường xuyên |
| Zustand / Redux | Global state management; phù hợp cho ứng dụng lớn với state phân tán |
| `useRef` | Giữ giá trị giữa các render nhưng không trigger re-render |
| Props drilling | Truyền dữ liệu từ parent xuống child, nhưng dễ trở nên cumbersome với nhiều level |

## 9. How

```jsx
import { useState } from 'react';

export default function Counter() {
  const [count, setCount] = useState(0);
  const [name, setName] = useState('');

  const handleIncrement = () => {
    setCount(count + 1);
  };

  const handleNameChange = (event) => {
    setName(event.target.value);
  };

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={handleIncrement}>Increment</button>

      <input 
        value={name} 
        onChange={handleNameChange} 
        placeholder="Enter your name"
      />
      <p>Hello, {name}!</p>
    </div>
  );
}
```

**Lazy initialization** (nếu initial state tốn chi phí tính toán):
```jsx
const [data, setData] = useState(() => {
  return expensiveCalculation();
});
```

**Functional update** (dùng state cũ để tính state mới, tránh stale state):
```jsx
setCount(prevCount => prevCount + 1);
```

## 10. Production concerns

### Scaling
- Nếu component có nhiều state, xem xét dùng `useReducer` để tập trung logic
- Với nested state, dùng spread operator hoặc immer library để tránh mutating state
- Tránh tạo object/array mới mà không cần (sẽ gây re-render child components)

### Failure
- **Stale state**: Nếu setState callback tham chiếu `state` trực tiếp mà không dùng functional update, có thể lấy giá trị cũ
- **Memory leak**: Nếu subscription/timer không được cleanup, có thể gây leak khi component unmount
- **Lost updates**: Nếu render bị interrupt, state update có thể bị lost (React 18 giải quyết phần nào)

### Monitoring
- Trace state changes bằng custom hook hoặc debugger
- Kiểm tra re-render không cần thiết (dùng React DevTools Profiler)
- Log state changes để debug trong production

## 11. Common mistakes

- **Mistake**: Mutating state trực tiếp
  ```jsx
  const [arr, setArr] = useState([1, 2, 3]);
  arr.push(4); // SAI - mutating state trực tiếp
  ```
  **Fix**: 
  ```jsx
  setArr([...arr, 4]); // ĐÚNG - tạo array mới
  ```

- **Mistake**: Gọi setState trong vòng lặp/điều kiện
  ```jsx
  for (let i = 0; i < 10; i++) {
    const [value, setValue] = useState(i); // SAI - vi phạm Rules of Hooks
  }
  ```
  **Fix**: 
  ```jsx
  const [values, setValues] = useState(Array(10).fill(0).map((_, i) => i));
  ```

- **Mistake**: Gọi setState mà không dùng functional update, dẫn đến stale state
  ```jsx
  useEffect(() => {
    setCount(count + 1);
    setCount(count + 1); // Chi chạy setCount một lần, cả hai đều tham chiếu count cũ
  }, []);
  ```
  **Fix**: 
  ```jsx
  useEffect(() => {
    setCount(prev => prev + 1);
    setCount(prev => prev + 1); // Đúng - từng cái tham chiếu giá trị mới nhất
  }, []);
  ```

- **Mistake**: Tạo object/array mới mỗi render mà không cần
  ```jsx
  const [config, setConfig] = useState(() => ({
    options: [1, 2, 3] // Tạo mới mỗi render mà không thay đổi
  }));
  ```
  **Fix**: 
  ```jsx
  const DEFAULT_CONFIG = { options: [1, 2, 3] };
  const [config, setConfig] = useState(DEFAULT_CONFIG);
  ```

## 12. Sample project

**Bài tập**: Xây dựng một **Todo App** với các constraint:
- Có thể thêm, xóa, đánh dấu hoàn thành todo
- Hiển thị số lượng todo còn lại
- Persist state vào localStorage khi thay đổi
- Không được dùng external library quản lý state
- Code phải tránh stale state và unnecessary re-renders

**Khó**: Nếu thêm feature filter (All, Active, Completed), hãy tối ưu để component chỉ re-render khi filter hoặc todo list thay đổi, không phải mỗi lần state con thay đổi.

## 13. Interview

### Core Q&A

1. **Q: `useState` trả về gì? Giải thích hai phần tử trong mảng.**
   A: `useState` trả về một mảng `[state, setState]`. Phần tử thứ nhất là giá trị state hiện tại (có thể là bất kỳ kiểu dữ liệu nào). Phần tử thứ hai là hàm để cập nhật state - khi gọi hàm này, React sẽ re-render component với state mới.

2. **Q: Tại sao không thể dùng `useState` trong vòng lặp hoặc điều kiện?**
   A: Vì React dựa vào **thứ tự** của hook calls để khớp state với component instance. Nếu số lượng hoặc thứ tự hook thay đổi giữa các render, React sẽ map state tới hook sai, gây bug. Đây là **Rules of Hooks**.

3. **Q: Sự khác biệt giữa `setCount(count + 1)` và `setCount(prev => prev + 1)`?**
   A: Cách thứ nhất truyền giá trị cụ thể; cách thứ nhất truyền hàm cập nhật. Cách thứ hai đảm bảo luôn dùng state mới nhất, tránh stale state - đặc biệt quan trọng nếu gọi setState liên tiếp hoặc trong setTimeout/callback.

4. **Q: State trong `useState` có được mutate trực tiếp được không? Tại sao?**
   A: Không. React dựa vào object identity (reference equality) để phát hiện state thay đổi. Nếu mutate trực tiếp, reference không đổi, React sẽ không re-render. Phải tạo object/array mới (shallow copy là đủ) để kích hoạt re-render.

5. **Q: Lazy initialization trong `useState` dùng khi nào?**
   A: Khi initial state tốn chi phí tính toán (ví dụ: parse JSON, call function nặng, ...). Thay vì chuyền giá trị trực tiếp, truyền hàm: `useState(() => expensiveCalculation())`. Hàm chỉ chạy lần đầu tiên.

### Scenario

1. **Scenario**: Bạn có một form với 5 input fields (name, email, phone, address, notes). Mỗi input thay đổi trigger một `setState`. Bạn nhận thấy component re-render 5 lần khi user nhập vào một input. Làm thế nào để tối ưu?
   **Answer**: Nhóm tất cả input vào một single state object: `const [form, setForm] = useState({...})` rồi update key cụ thể: `setForm({...form, name: value})`. Cách này chỉ trigger một setState, do đó một re-render. Hoặc dùng `useReducer` nếu logic update phức tạp hơn.

2. **Scenario**: Bạn fetch data từ API khi component mount. Bạn dùng `useEffect` để gọi API rồi `setState`. Nhưng state bị cập nhật sau khi component unmount (vì async). Làm sao fix?
   **Answer**: Thêm cleanup function trong `useEffect` để cancel request hoặc check xem component còn mounted không. Ví dụ: `let isMounted = true` trong effect, rồi `if (isMounted) setState(...)` trước khi return cleanup function `return () => { isMounted = false }`.

3. **Scenario**: Bạn có counter với `useState(0)`. Bạn gọi `setCount(count + 1)` ba lần liên tiếp trong một click handler. Bạn mong đợi count tăng 3 nhưng chỉ tăng 1. Tại sao?
   **Answer**: Vì `count` trong handler là giá trị khi handler được định nghĩa, không phải khi gọi. React cũng batch setState updates, nên cả 3 calls đều thấy `count` như nhau. Fix: dùng functional update `setCount(prev => prev + 1)` ba lần.

## 14. References

- **Official Docs**: https://react.dev/reference/react/useState
- **GitHub Repo**: https://github.com/facebook/react
- **Rules of Hooks**: https://react.dev/warnings/invalid-hook-call-warning
- **Batching Updates**: https://react.dev/blog/2022/03/29/react-v18-and-automatic-batching

## 15. Real-world Code

- **Vercel Next.js Examples**: https://github.com/vercel/next.js/tree/canary/examples
- **Create React App**: https://github.com/facebook/create-react-app (nhiều ví dụ functional component)
- **React Query (TanStack Query)**: https://github.com/tanstack/query (quản lý server state, tham khảo cách dùng useState cho local state)
- **Zustand**: https://github.com/pmndrs/zustand (global state management; tham khảo cách setState được implement)

## 16. Community

- **Reddit**: r/reactjs - subreddit chính thức, nhiều Q&A về useState
- **Stack Overflow**: Tag `reactjs` + `usestate` có hàng ngàn câu hỏi và answer chất lượng
- **Blog**: https://overreacted.io/ (Dan Abramov giải thích closure, stale state)
- **Talk**: "React Hooks" by Ryan Florence & Michael Jackson (React Conf 2018) - video giới thiệu hooks lần đầu
