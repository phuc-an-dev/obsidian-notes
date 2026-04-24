---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/react"
  - "#topic/error-handling"
related:
  - "[[typescript]]"
  - "[[react-lifecycle]]"
---

## 1. What
Props Typing trong React với TypeScript là việc sử dụng hệ thống kiểu (Type System) của TypeScript để định nghĩa cấu trúc và kiểu dữ liệu cho các thuộc tính (props) mà một component nhận vào. Điều này giúp kiểm tra tính hợp lệ của dữ liệu ngay tại thời điểm lập trình (Compile-time).

## 2. Why
Trước khi có TypeScript, React sử dụng `PropTypes` để kiểm tra kiểu dữ liệu ở Runtime, nhưng nó không ngăn chặn được lỗi trước khi chạy code. TypeScript giúp bắt lỗi sớm, cung cấp tính năng tự động gợi ý (Intellisense) cực mạnh và giúp các lập trình viên khác hiểu nhanh cấu trúc của một component mà không cần đọc hết code xử lý bên trong.

## 3. Mental Model
Hãy tưởng tượng một component như một chiếc máy sản xuất, và props là nguyên liệu đầu vào. Props Typing giống như một **bản vẽ kỹ thuật** hoặc một bộ lọc ở cửa đầu vào. Nó quy định chính xác nguyên liệu phải có hình dạng gì, kích thước bao nhiêu. Nếu bạn đưa vào một mẩu gỗ thay vì một thanh sắt, máy sẽ báo lỗi ngay lập tức thay vì bị hỏng khi đang chạy.

## 4. Where it fits
TypeScript Type/Interface -> Component Declaration -> Props Destructuring -> JSX Rendering.

## 5. When to use
- Luôn luôn sử dụng khi xây dựng ứng dụng React với TypeScript.
- Khi làm việc trong dự án có nhiều thành viên để đảm bảo tính nhất quán.
- Khi xây dựng các thư viện UI (UI Kits) cho người khác dùng.

## 6. When NOT to use
- Khi dự án sử dụng JavaScript thuần túy (không dùng TypeScript).
- Các bản prototype cực nhanh, nhỏ gọn mà bạn không quan tâm đến tính an toàn (tuy nhiên vẫn khuyến khích dùng).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Bắt lỗi ngay khi đang viết code | Tốn thêm thời gian viết định nghĩa kiểu ban đầu |
| Tự động gợi ý thuộc tính cực tốt | Đường cong học tập đối với các kiểu phức tạp (Generics) |
| Tài liệu hóa component một cách tự động | Code trông dài dòng hơn một chút |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| PropTypes | Kiểm tra ở Runtime, không có Intellisense tốt bằng TS. |
| JSDoc | Hỗ trợ gợi ý nhưng không ép buộc kiểu chặt chẽ bằng TS. |

## 9. How
Sử dụng `interface` hoặc `type` để định nghĩa props:

```tsx
interface UserProps {
  name: string;
  age: number;
  email?: string; // Optional prop
  onStatusChange: (status: string) => void; // Function prop
  children: React.ReactNode; // React element prop
}

const UserProfile = ({ name, age, email, onStatusChange, children }: UserProps) => {
  return (
    <div>
      <h1>{name} ({age})</h1>
      {email && <p>{email}</p>}
      <button onClick={() => onStatusChange('active')}>Change Status</button>
      {children}
    </div>
  );
};
```

## 10. Production concerns
### Maintenance
Nên tách các interface phức tạp ra các file `.types.ts` riêng để tái sử dụng và giữ cho file component gọn gàng.

### Scaling
Sử dụng Utility Types của TypeScript như `Pick`, `Omit`, hoặc `Partial` để tạo ra các kiểu props mới từ các kiểu dữ liệu (Entities) có sẵn trong hệ thống.

## 11. Common mistakes
- Mistake: Sử dụng kiểu `any` cho props.
  Fix: Luôn định nghĩa kiểu cụ thể, hoặc dùng `unknown` nếu thực sự không biết trước.

- Mistake: Không định nghĩa kiểu cho các event handlers (ví dụ: `onChange`).
  Fix: Sử dụng các kiểu có sẵn của React như `React.ChangeEvent<HTMLInputElement>`.

## 12. Sample project
Tạo một component `Table` có tính năng Generic: Nhận vào một mảng dữ liệu bất kỳ và một hàm render hàng, đảm bảo TypeScript hiểu đúng kiểu của từng phần tử trong mảng dữ liệu đó.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa `interface` và `type` khi định nghĩa React Props?
   A: Cả hai đều dùng được. `interface` hỗ trợ mở rộng (extends) và gộp (merge) tốt hơn, thường dùng cho public API. `type` linh hoạt hơn cho các phép hợp (union) hoặc giao (intersection), thường dùng cho logic nội bộ phức tạp.

### Scenario
1. Q: Bạn làm thế nào để component của bạn nhận vào tất cả các thuộc tính của một thẻ `button` mặc định cộng thêm một vài thuộc tính tùy chỉnh?
   A: Tôi sẽ dùng tính năng kế thừa: `interface MyButtonProps extends React.ButtonHTMLAttributes<HTMLButtonElement> { customProp: string; }`.

## 14. References
- React TypeScript Cheatsheet: https://react-typescript-cheatsheet.netlify.app/
- Official TS Docs: https://www.typescriptlang.org/docs/handbook/react.html

## 15. Real-world Code
Nghiên cứu mã nguồn của `Material UI` hoặc `Chakra UI` để xem cách họ xử lý hệ thống props cực kỳ phức tạp và linh hoạt.

## 16. Community
- YouTube: "React & TypeScript - Course for Beginners" - FreeCodeCamp.
- Reddit: r/reactjs về các best practices cho TypeScript.
