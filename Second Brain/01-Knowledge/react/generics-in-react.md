---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/react"
  - "#topic/error-handling"
related:
  - "[[typescript]]"
  - "[[props-typing]]"
---

## 1. What
Generics trong React là việc sử dụng tính năng Generic của TypeScript để tạo ra các components, hooks hoặc functions có khả năng làm việc với nhiều kiểu dữ liệu khác nhau mà vẫn đảm bảo tính an toàn về kiểu (Type-safety).

## 2. Why
Đôi khi bạn cần xây dựng một component (như Table, Select, List) mà cấu trúc dữ liệu đầu vào không cố định. Nếu dùng `any`, bạn mất đi sự hỗ trợ của TypeScript. Generics cho phép bạn "tham số hóa" kiểu dữ liệu, giúp component linh hoạt nhưng vẫn bắt được lỗi nếu dữ liệu truyền vào không khớp với cách sử dụng.

## 3. Mental Model
Hãy tưởng tượng Generic như một **chiếc khuôn đúc đa năng**.
Thay vì tạo ra một chiếc khuôn chỉ đúc được sắt hoặc chỉ đúc được nhựa, bạn tạo ra một chiếc khuôn có thể nhận bất kỳ vật liệu lỏng nào (Type Parameter). Khi bạn đổ vật liệu vào, chiếc khuôn sẽ tự định hình theo vật liệu đó và cho ra sản phẩm có đúng tính chất của vật liệu bạn đã chọn.

## 4. Where it fits
Component Declaration `<T>` -> Props Definition `data: T[]` -> Usage in JSX -> Component Invocation `<MyComponent<User> ... />`.

## 5. When to use
- Xây dựng các Reusable Components: Table, List, Dropdown, Form.
- Khi viết Custom Hooks xử lý dữ liệu (ví dụ: `useFetch<T>`, `useLocalStorage<T>`).
- Khi cần ràng buộc mối quan hệ giữa các props (ví dụ: prop `value` và hàm `onChange` phải cùng kiểu).

## 6. When NOT to use
- Khi component chỉ làm việc với một kiểu dữ liệu cố định và duy nhất.
- Khi việc sử dụng Generic làm cho code trở nên quá phức tạp và khó đọc mà không mang lại lợi ích rõ rệt về an toàn kiểu.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Khả năng tái sử dụng code cực cao | Cú pháp phức tạp, khó tiếp cận cho người mới |
| Đảm bảo tính nhất quán dữ liệu từ đầu đến cuối | Có thể làm chậm quá trình biên dịch nếu lồng nhau quá sâu |
| Intellisense hoạt động chính xác với dữ liệu động | Đôi khi cần viết thêm các Type Constraints để giới hạn kiểu |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Union Types | Tốt nếu danh sách các kiểu dữ liệu là cố định và ít. |
| any / unknown | Mất an toàn kiểu hoặc phải ép kiểu (Type Casting) thủ công. |

## 9. How
Ví dụ về một Generic List component:

```tsx
interface ListProps<T> {
  items: T[];
  renderItem: (item: T) => React.ReactNode;
  keyExtractor: (item: T) => string | number;
}

function GenericList<T>({ items, renderItem, keyExtractor }: ListProps<T>) {
  return (
    <ul>
      {items.map((item) => (
        <li key={keyExtractor(item)}>
          {renderItem(item)}
        </li>
      ))}
    </ul>
  );
}

// Cách dùng
const users = [{ id: 1, name: 'An Phuc' }];
<GenericList 
  items={users} 
  keyExtractor={(u) => u.id} 
  renderItem={(u) => <span>{u.name}</span>} 
/>
```

## 10. Production concerns
### Performance
Generics chỉ tồn tại ở thời điểm biên dịch TypeScript, không ảnh hưởng đến hiệu năng Runtime của JavaScript.

### Maintenance
Sử dụng các ràng buộc (Constraints) như `<T extends { id: string }>` để đảm bảo dữ liệu truyền vào luôn có các thuộc tính bắt buộc, giúp giảm thiểu check null/undefined trong code.

## 11. Common mistakes
- Mistake: Không định nghĩa Generic cho Arrow Function đúng cách trong file `.tsx`.
  Fix: Dùng `<T,>` hoặc `<T extends unknown>` để tránh trình biên dịch nhầm lẫn với thẻ JSX.

- Mistake: Quá lạm dụng Generic cho những trường hợp đơn giản.
  Fix: Chỉ dùng khi thực sự cần tính linh hoạt cho nhiều kiểu dữ liệu khác nhau.

## 12. Sample project
Xây dựng một `FormManager` component sử dụng Generics để quản lý trạng thái của bất kỳ object dữ liệu nào, hỗ trợ validation và submit với kiểu dữ liệu trả về chính xác.

## 13. Interview
### Core Q&A
1. Q: Tại sao cần `extends` trong Generics?
   A: Để giới hạn phạm vi của kiểu dữ liệu. Ví dụ `<T extends object>` đảm bảo T luôn là một object, giúp ta có thể truy cập các phương thức của object mà không sợ lỗi.

### Scenario
1. Q: Làm thế nào để tạo một Hook `useAPI` nhận vào một kiểu trả về `T` và trả về dữ liệu đúng kiểu đó?
   A: Tôi sẽ định nghĩa `function useAPI<T>(url: string): { data: T | null }`. Khi gọi hook, người dùng có thể truyền vào interface: `useAPI<User[]>('/users')`.

## 14. References
- TypeScript Generics: https://www.typescriptlang.org/docs/handbook/2/generics.html
- TS Generics in React: https://react-typescript-cheatsheet.netlify.app/docs/advanced/patterns_by_usecase/#generic-components

## 15. Real-world Code
Nghiên cứu `TanStack Table` (React Table) - một thư viện sử dụng Generics cực kỳ mạnh mẽ để quản lý mọi loại dữ liệu bảng.

## 16. Community
- YouTube: "TypeScript Generics in 10 Minutes".
- Blog: "Total TypeScript - Transform your React components with Generics".
