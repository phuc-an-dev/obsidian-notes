---
created: 2026-04-24
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/react"
  - "#topic/state-management"
related:
  - "[[context-api-typing]]"
  - "[[react-lifecycle]]"
---

## 1. What
Zustand là một thư viện quản lý trạng thái (state management) nhỏ gọn, nhanh chóng và có khả năng mở rộng cho React. Nó dựa trên mô hình hooks, không yêu cầu boilerplate phức tạp như Redux và giải quyết được vấn đề re-render thừa thãi của Context API.

## 2. Why
Trước khi có Zustand, lập trình viên thường phải chọn giữa Context API (dễ gây re-render toàn bộ cây component) hoặc Redux (quá nhiều code rườm rà như Actions, Reducers, Types). Zustand ra đời để cung cấp một giải pháp "tối giản" nhất: tạo store đơn giản như một object và truy cập nó qua hooks một cách hiệu quả.

## 3. Mental Model
Hãy tưởng tượng Zustand như một **Bảng thông báo chung (Whiteboard)** đặt ở sảnh công ty.
Bất kỳ nhân viên nào (Component) cũng có thể đến đọc thông tin trên bảng hoặc viết thêm thông tin mới vào đó. Điều đặc biệt là mỗi nhân viên chỉ quan tâm đến một dòng cụ thể trên bảng. Khi dòng đó thay đổi, chỉ nhân viên đó mới cần chú ý, những người khác vẫn tiếp tục làm việc bình thường mà không bị làm phiền (Selective re-render).

## 4. Where it fits
External Store (Vanilla JS) -> React Hook (Zustand) -> Selective Component Updates.

## 5. When to use
- Quản lý Global State trong các ứng dụng React từ nhỏ đến lớn.
- Khi muốn chia sẻ dữ liệu giữa các component cách xa nhau mà không muốn dùng Prop Drilling.
- Khi cần một giải pháp quản lý state có hiệu năng cao và ít code boilerplate.
- Khi cần lưu trữ state bền vững (Persist state) xuống LocalStorage một cách dễ dàng.

## 6. When NOT to use
- Các trạng thái chỉ dùng duy nhất trong một component (nên dùng `useState`).
- Các ứng dụng cực kỳ đơn giản mà Context API đã đủ đáp ứng.
- Khi team của bạn đã quá quen thuộc và có hệ thống quy chuẩn chặt chẽ với Redux trong các dự án Enterprise khổng lồ.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Cực kỳ ít code boilerplate | Ít tính năng "opinionated" (ép buộc cấu trúc) hơn Redux |
| Hiệu năng tối ưu mặc định (qua selectors) | Cộng đồng và hệ sinh thái middleware nhỏ hơn Redux |
| Không cần Provider bọc ở tầng Root | Dễ dẫn đến việc lạm dụng Global State nếu không kiểm soát tốt |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Redux Toolkit | Tiêu chuẩn cho dự án lớn, cấu trúc chặt chẽ nhưng vẫn nặng nề hơn. |
| Jotai / Recoil | Quản lý state theo hướng "Atomic", phù hợp cho các state dạng đồ thị. |
| Context API | Tích hợp sẵn nhưng khó tối ưu re-render cho dữ liệu lớn. |

## 9. How
Cách tạo và sử dụng Store với TypeScript:

```tsx
import { create } from 'zustand';

interface CounterState {
  count: number;
  increment: (by: number) => void;
  reset: () => void;
}

// 1. Khởi tạo store
const useCounterStore = create<CounterState>((set) => ({
  count: 0,
  increment: (by) => set((state) => ({ count: state.count + by })),
  reset: () => set({ count: 0 }),
}));

// 2. Sử dụng trong component với Selector
const CounterComponent = () => {
  // Chỉ re-render khi thuộc tính 'count' thay đổi
  const count = useCounterStore((state) => state.count);
  const increment = useCounterStore((state) => state.increment);

  return (
    <div>
      <h1>Count: {count}</h1>
      <button onClick={() => increment(1)}>+1</button>
    </div>
  );
};
```

## 10. Production concerns
### Performance
Luôn luôn sử dụng **Selectors** khi lấy dữ liệu từ store. Tránh việc lấy nguyên object `const state = useStore()` vì nó sẽ gây re-render khi bất kỳ thuộc tính nào trong store thay đổi.

### Persistence
Sử dụng middleware `persist` để tự động lưu state vào LocalStorage hoặc SessionStorage, giúp dữ liệu không bị mất khi người dùng F5 trang web.

### Debugging
Tích hợp sẵn với **Redux DevTools** thông qua middleware `devtools` để dễ dàng theo dõi lịch sử thay đổi của state.

## 11. Common mistakes
- Mistake: Lấy toàn bộ store object vào component.
  Fix: Luôn dùng selector: `const value = useStore(state => state.value)`.

- Mistake: Thay đổi state trực tiếp (mutating state).
  Fix: Luôn sử dụng hàm `set()` và đảm bảo trả về một object mới (Immutability).

## 12. Sample project
Xây dựng một hệ thống Giỏ hàng (Shopping Cart): Store lưu trữ danh sách sản phẩm, tổng tiền, và các hàm thêm/xóa/sửa số lượng. Sử dụng middleware `persist` để giỏ hàng vẫn tồn tại khi người dùng quay lại sau.

## 13. Interview
### Core Q&A
1. Q: Tại sao Zustand lại nhanh hơn Context API?
   A: Vì Zustand hoạt động bên ngoài cây component của React. Nó sử dụng một cơ chế subscription thông minh để chỉ thông báo cho các component có sử dụng đúng phần dữ liệu vừa thay đổi, thay vì render lại mọi thứ từ Provider trở xuống.

### Scenario
1. Q: Làm thế nào để truy cập state của Zustand bên ngoài một React Component (ví dụ trong một file API helper)?
   A: Bạn có thể dùng `useStore.getState()` để lấy giá trị hiện tại và `useStore.setState()` để cập nhật mà không cần thông qua hooks.

## 14. References
- Zustand GitHub: https://github.com/pmndrs/zustand
- Documentation: https://docs.pmnd.rs/zustand/getting-started/introduction

## 15. Real-world Code
Nghiên cứu các project của `pmndrs` (như React Three Fiber) thường xuyên sử dụng Zustand để quản lý các trạng thái phức tạp của môi trường 3D.

## 16. Community
- YouTube: "Zustand - The best React state management library" - t3․gg.
- Reddit: r/reactjs thường xuyên có các bài so sánh Zustand vs Redux.
