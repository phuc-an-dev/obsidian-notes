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
Event Typing trong React với TypeScript là việc định nghĩa kiểu dữ liệu chính xác cho các sự kiện (events) và các phần tử mục tiêu (target elements) khi xử lý tương tác người dùng (như click, change, submit). Nó sử dụng các kiểu dữ liệu có sẵn từ namespace `React`.

## 2. Why
Nếu không có typing, tham số `event` thường bị gán kiểu `any`, dẫn đến việc không biết các thuộc tính như `event.target.value` có tồn tại hay không. Typing giúp bắt lỗi ngay khi truy cập sai thuộc tính của sự kiện và cung cấp gợi ý mã nguồn (Intellisense) cho các phương thức như `preventDefault()` hay `stopPropagation()`.

## 3. Mental Model
Hãy tưởng tượng các sự kiện như các loại **tín hiệu giao thông** khác nhau.
- `ChangeEvent` là tín hiệu từ các biển báo có thể thay đổi nội dung (input, select).
- `MouseEvent` là tín hiệu từ việc chạm/nhấn vật lý (click, hover).
Event Typing giúp bạn trang bị đúng loại máy thu tín hiệu cho từng vị trí, đảm bảo bạn không cố gắng đọc "nội dung chữ" từ một "tiếng còi xe".

## 4. Where it fits
JSX Element Handler (e.g., onChange) -> Callback Function -> **Event Type Definition** -> Logic Processing.

## 5. When to use
- Khi viết các hàm xử lý sự kiện tách rời (Handler functions) thay vì viết inline.
- Khi cần truy cập vào các thuộc tính đặc thù của phần tử gây ra sự kiện (như `value` của input).
- Khi xây dựng các form phức tạp hoặc các custom UI components.

## 6. When NOT to use
- Khi viết hàm xử lý inline ngay trong JSX (TypeScript thường tự suy luận được kiểu trong trường hợp này).
- Đối với các sự kiện cực kỳ đơn giản mà bạn không cần truy cập vào đối tượng `event`.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| An toàn tuyệt đối khi truy cập DOM | Cú pháp đôi khi dài dòng (ví dụ: `React.ChangeEvent<HTMLInputElement>`) |
| Gợi ý thuộc tính chính xác cho từng loại thẻ | Phải nhớ chính xác tên kiểu cho từng loại sự kiện |
| Dễ dàng refactor và bảo trì | Khó khăn khi xử lý các sự kiện hiếm gặp |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Inline functions | Tiện lợi, tự suy luận kiểu nhưng khó tái sử dụng và unit test. |
| SyntheticEvent | Kiểu cơ bản nhất, dùng được cho mọi event nhưng thiếu thuộc tính đặc thù. |

## 9. How
Các kiểu sự kiện phổ biến:

```tsx
import React from 'react';

const MyComponent = () => {
  // 1. Change Event cho Input
  const handleChange = (event: React.ChangeEvent<HTMLInputElement>) => {
    console.log(event.target.value);
  };

  // 2. Form Event cho Submit
  const handleSubmit = (event: React.FormEvent<HTMLFormElement>) => {
    event.preventDefault();
  };

  // 3. Mouse Event cho Button
  const handleClick = (event: React.MouseEvent<HTMLButtonElement>) => {
    console.log('Clicked at:', event.clientX, event.clientY);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input type="text" onChange={handleChange} />
      <button type="button" onClick={handleClick}>Click Me</button>
      <button type="submit">Submit</button>
    </form>
  );
};
```

## 10. Production concerns
### Maintenance
Nên thống nhất cách đặt tên kiểu (ví dụ: luôn dùng `React.ChangeEvent` thay vì import trực tiếp `ChangeEvent` từ 'react' để tránh nhầm lẫn với các thư viện khác).

### Failure
Lưu ý sự khác biệt giữa `event.target` (phần tử thực tế bị click) và `event.currentTarget` (phần tử mà handler được gán vào). Trong TypeScript, `currentTarget` thường an toàn hơn khi truy cập thuộc tính.

## 11. Common mistakes
- Mistake: Dùng `any` cho tham số event.
  Fix: Sử dụng đúng kiểu cụ thể như `React.ChangeEvent<HTMLSelectElement>`.

- Mistake: Quên truyền Generic type cho phần tử HTML (ví dụ: chỉ ghi `React.ChangeEvent` mà thiếu `<HTMLInputElement>`).
  Fix: Luôn xác định rõ phần tử mục tiêu để có gợi ý thuộc tính `target` chính xác.

## 12. Sample project
Xây dựng một `GenericForm` component nhận vào danh sách các field, tự động xử lý `onChange` cho nhiều loại input (text, checkbox, select) với typing chặt chẽ.

## 13. Interview
### Core Q&A
1. Q: Làm thế nào để định nghĩa kiểu cho một hàm nhận vào sự kiện click của cả thẻ `button` và thẻ `div`?
   A: Sử dụng Union Type cho Generic: `React.MouseEvent<HTMLButtonElement | HTMLDivElement>`.

### Scenario
1. Q: Bạn xử lý thế nào nếu một thư viện ngoài trả về một sự kiện không thuộc namespace `React`?
   A: Tôi sẽ kiểm tra xem thư viện đó có cung cấp kiểu riêng không. Nếu không, tôi có thể dùng `React.SyntheticEvent` làm kiểu cơ sở hoặc ép kiểu (Type Casting) nếu chắc chắn về cấu trúc dữ liệu.

## 14. References
- React TypeScript Cheatsheet - Events: https://react-typescript-cheatsheet.netlify.app/docs/basic/getting-started/forms_and_events/
- Official React Docs on Events: https://react.dev/reference/react-dom/components/common#social-engineering-events

## 15. Real-world Code
Kiểm tra cách các thư viện form như `React Hook Form` hoặc `Formik` định nghĩa kiểu cho các sự kiện để hỗ trợ người dùng.

## 16. Community
- YouTube: "TypeScript with React Events" - Total TypeScript.
- Blog: "Matt Pocock - The only way to type events in React".
