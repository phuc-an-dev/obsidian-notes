---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/react"
  - "#topic/state-management"
related:
  - "[[typescript]]"
  - "[[useState]]"
  - "[[props-typing]]"
---

## 1. What
useState Typing là việc sử dụng TypeScript Generics hoặc Type Inference (suy luận kiểu) để định nghĩa kiểu dữ liệu cho trạng thái (state) của một component. Nó xác định biến state có thể chứa những giá trị nào và hàm setter tương ứng phải nhận vào kiểu dữ liệu gì.

## 2. Why
Nếu không có typing, state có thể vô tình bị thay đổi thành một kiểu dữ liệu không mong muốn (ví dụ: từ object thành null mà không có kiểm tra), dẫn đến lỗi "Cannot read property of null" ở Runtime. TypeScript giúp đảm bảo tính nhất quán của dữ liệu xuyên suốt vòng đời của component.

## 3. Mental Model
Hãy tưởng tượng `useState` như một **ngăn kéo có nhãn**. Nếu bạn dán nhãn "Chỉ đựng Táo" (Typing), hệ thống sẽ ngăn cản bạn bỏ một "Quả Cam" vào đó. Khi bạn lấy đồ từ ngăn kéo ra, bạn cũng chắc chắn 100% đó là một quả táo mà không cần phải kiểm tra lại.

## 4. Where it fits
Component Definition -> useState<Type>(InitialValue) -> State Access & Update.

## 5. When to use
- Khi giá trị khởi tạo là `null` hoặc `undefined` nhưng sau đó sẽ là một object/array.
- Khi state là một object phức tạp hoặc một mảng các objects.
- Khi state có thể là một trong nhiều kiểu khác nhau (Union Types).
- Khi kiểu dữ liệu của state khác với kiểu dữ liệu của giá trị khởi tạo.

## 6. When NOT to use
- Đối với các kiểu dữ liệu nguyên bản (primitive) đơn giản như `string`, `number`, `boolean` nếu giá trị khởi tạo đã thể hiện rõ kiểu đó (TypeScript tự suy luận được).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Ngăn chặn lỗi gán sai kiểu dữ liệu | Code trông phức tạp hơn với các cú pháp Generics `< >` |
| Tự động gợi ý thuộc tính của state (Intellisense) | Đòi hỏi phải định nghĩa Interface/Type trước khi dùng |
| An toàn khi xử lý các giá trị `null/undefined` | Cần xử lý kỹ các trường hợp Type Assertion nếu dùng thư viện ngoài |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Type Inference | Tốt cho primitives, không cần viết thêm code nhưng yếu với object phức tạp. |
| useReducer | Tốt hơn khi state có cấu trúc cực kỳ phức tạp và nhiều logic thay đổi. |

## 9. How
Các cách định nghĩa kiểu phổ biến:

```tsx
interface User {
  id: string;
  name: string;
}

// 1. Suy luận kiểu (Inference) - Dùng cho primitives
const [count, setCount] = useState(0); // Tự hiểu là number

// 2. Sử dụng Generics - Dùng cho Object/Array
const [user, setUser] = useState<User | null>(null);

// 3. Sử dụng Union Types
const [status, setStatus] = useState<'loading' | 'success' | 'error'>('loading');

// 4. Mảng các Objects
const [items, setItems] = useState<User[]>([]);
```

## 10. Production concerns
### Scaling
Nên sử dụng các Types/Interfaces dùng chung từ thư viện trung tâm của dự án để đảm bảo `useState` trong UI đồng bộ với dữ liệu từ Backend API.

### Failure
Luôn cân nhắc khởi tạo giá trị mặc định an toàn (ví dụ: mảng rỗng `[]` thay vì `null`) để tránh phải kiểm tra optional chaining `?.` quá nhiều trong phần JSX.

## 11. Common mistakes
- Mistake: Không định nghĩa kiểu cho state khởi tạo là mảng rỗng.
  Fix: `useState<string[]>([])` thay vì `useState([])` (TS sẽ hiểu là `never[]`).

- Mistake: Ép kiểu bằng `as` quá nhiều (Type Assertion).
  Fix: Khai báo rõ ràng kiểu ở phần Generic `<Type>` để TS kiểm tra thực sự.

## 12. Sample project
Xây dựng một component `UserSearch`: State lưu trữ kết quả tìm kiếm là một mảng `User[]`, trạng thái loading, và thông báo lỗi. Yêu cầu định nghĩa kiểu chặt chẽ cho tất cả các states này.

## 13. Interview
### Core Q&A
1. Q: Tại sao `const [val, setVal] = useState([])` lại gây lỗi khi bạn cố `setVal(["hello"])`?
   A: Vì TypeScript suy luận kiểu của `[]` là `never[]`. Để sửa, ta cần dùng Generic: `useState<string[]>([])`.

### Scenario
1. Q: Bạn xử lý thế nào nếu state nhận dữ liệu từ một API trả về object có thể thiếu một vài trường?
   A: Tôi sẽ định nghĩa Interface với các optional properties (`?`) và sử dụng nó trong `useState`. Khi hiển thị, tôi dùng Optional Chaining hoặc Nullish Coalescing để đảm bảo UI không bị crash.

## 14. References
- React TS CheatSheet: https://react-typescript-cheatsheet.netlify.app/docs/basic/getting-started/hooks/#usestate
- MDN TypeScript: https://www.typescriptlang.org/docs/handbook/2/generics.html

## 15. Real-world Code
Kiểm tra các project sử dụng `TanStack Query` (React Query) - họ sử dụng Generics rất nặng cho các trạng thái dữ liệu.

## 16. Community
- YouTube: "React Hooks with TypeScript" - Ben Awad / Jack Herrington.
- Blog: "Matt Pocock - Total TypeScript" (Chuyên gia về TS trong React).
