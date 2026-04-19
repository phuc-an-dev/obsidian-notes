---
created: 2026-04-17
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/javascript"
  - "#lang/typescript"
related: "[[javascript/axios]]"
---
## 1. What
`TypeScript` là một ngôn ngữ lập trình mã nguồn mở được phát triển bởi Microsoft. Nó là một **superset** (tập siêu) của JavaScript, bổ sung thêm **static typing** (kiểu tĩnh) và các tính năng hướng đối tượng vào ngôn ngữ này. Mã nguồn TypeScript sau đó sẽ được biên dịch (transpile) sang JavaScript thuần để có thể chạy trên trình duyệt hoặc Node.js.

## 2. Why
JavaScript thuần là một ngôn ngữ **dynamic typing**, dẫn đến các vấn đề:
- **Runtime Errors**: Lỗi chỉ xuất hiện khi ứng dụng đang chạy (ví dụ: `undefined is not a function`).
- **Refactoring Pain**: Khó khăn khi thay đổi cấu trúc code trong dự án lớn vì không biết biến nào đang được dùng ở đâu.
- **Poor IntelliSense**: IDE không thể gợi ý code chính xác vì không biết kiểu dữ liệu của biến.
TypeScript giải quyết bằng cách phát hiện lỗi ngay trong lúc viết code (Compile-time).

## 3. Mental Model
> "Hãy coi JavaScript như một **'Cuộc dạo chơi tự do'** trên thảo nguyên, bạn có thể đi bất cứ đâu nhưng dễ sụp hố. TypeScript giống như việc lắp thêm một **'Hệ thống GPS và hàng rào bảo vệ'**. GPS (IntelliSense) chỉ đường cho bạn chính xác, và hàng rào (Types) ngăn bạn không bước vào những vùng nguy hiểm. Bạn vẫn đang đi trên cùng một thảo nguyên (JS), nhưng an toàn và tự tin hơn nhiều."

## 4. Where it fits
`TypeScript Code (.ts) → TypeScript Compiler (tsc) → JavaScript Code (.js) → Browser/Node.js`

## 5. When to use
- Dự án quy mô trung bình đến lớn, có nhiều thành viên cùng tham gia.
- Khi xây dựng các thư viện (Library) để người dùng khác dễ dàng sử dụng nhờ gợi ý kiểu.
- Khi muốn giảm thiểu tối đa các lỗi vặt (logic errors) và tăng tốc độ bảo trì code lâu dài.

## 6. When NOT to use
- Các script cực nhỏ, chỉ vài dòng code để xử lý nhanh một tác vụ.
- Khi bạn đang học những khái niệm cơ bản nhất của JavaScript (nên nắm vững JS trước khi qua TS).
- Các dự án prototype cần tốc độ "mì ăn liền" và không có ý định duy trì.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Phát hiện lỗi sớm, giảm bug trong production. | Tốn thời gian thiết lập ban đầu (tsconfig, build tool). |
| Tài liệu hóa code tự động thông qua các Types/Interfaces. | Code dài dòng hơn do phải khai báo kiểu. |
| Hỗ trợ refactoring cực kỳ mạnh mẽ và an toàn. | Đòi hỏi lập trình viên phải học thêm các khái niệm mới. |

## 8. Alternatives
- **JSDoc**: Sử dụng comment để gợi ý kiểu trong JS (nhẹ nhàng hơn nhưng không mạnh bằng).
- **Flow**: Thư viện kiểm tra kiểu của Facebook (hiện tại ít phổ biến hơn TS).

## 9. How (Minimal Example)
```typescript
// 1. Khai báo Interface
interface User {
  id: number;
  name: string;
  email?: string; // Optional property
}

// 2. Sử dụng Type trong function
function greetUser(user: User): string {
  return `Hello, ${user.name}!`;
}

const myUser: User = { id: 1, name: "An Phuc" };

console.log(greetUser(myUser));

// 3. Generics (Tính linh hoạt)
function getFirstItem<T>(arr: T[]): T {
  return arr[0];
}

const firstNum = getFirstItem([1, 2, 3]); // Type là number
```

## 10. Production concerns
- **Strict Mode**: Luôn bật `strict: true` trong `tsconfig.json` để tận dụng tối đa sức mạnh bảo vệ của TS.
- **Any Type**: Tránh sử dụng `any` bằng mọi giá trong production vì nó vô hiệu hóa toàn bộ lợi ích của TS.
- **Source Maps**: Bật source maps để khi debug trên trình duyệt, bạn có thể nhìn thấy trực tiếp code TS thay vì code JS đã biên dịch.

## 11. Common mistakes
- ❌ **Mistake**: Lạm dụng `any` khi gặp một kiểu dữ liệu phức tạp.
  ✅ **Fix**: Sử dụng `unknown` nếu thực sự chưa biết kiểu, hoặc dùng `Record<string, any>` nếu là object.
- ❌ **Mistake**: Khai báo kiểu cho những thứ hiển nhiên (ví dụ: `let x: number = 5`).
  ✅ **Fix**: Tận dụng **Type Inference** (TS tự suy luận kiểu), chỉ khai báo khi cần thiết.

## 12. Sample project
**Tên project**: Type-safe Weather App.
**Constraint**: Gọi API thời tiết và hiển thị dữ liệu.
**Yêu cầu**: 
- Phải định nghĩa Interface cho dữ liệu trả về từ API (Response Data).
- Tạo một Union Type cho trạng thái thời tiết: `'SUNNY' | 'CLOUDY' | 'RAINY'`.
- Sử dụng Generic để viết một hàm fetch dữ liệu dùng chung.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa `Interface` và `Type` là gì?
   A: `Interface` chủ yếu dùng cho object và có khả năng "mở rộng" (declaration merging). `Type` linh hoạt hơn, có thể dùng cho Union, Intersection, Tuple và các kiểu dữ liệu nguyên thủy.
2. Q: `any`, `unknown` và `never` khác nhau như thế nào?
   A: `any` cho phép làm mọi thứ (không an toàn). `unknown` là kiểu an toàn hơn `any` (phải kiểm tra kiểu trước khi dùng). `never` đại diện cho giá trị không bao giờ xảy ra (ví dụ hàm luôn throw error).
3. Q: Generics trong TypeScript dùng để làm gì?
   A: Giúp tạo ra các thành phần (function, class, interface) có khả năng hoạt động với nhiều kiểu dữ liệu khác nhau mà vẫn giữ được tính an toàn (type safety).
4. Q: Enum có nhược điểm gì và giải pháp thay thế là gì?
   A: Enum trong TS có thể tạo ra code dư thừa khi biên dịch sang JS. Giải pháp hiện đại thường là dùng `const object` kết hợp với `as const` hoặc Union Types.

### Scenario
1. Tình huống: Bạn đang chuyển đổi một dự án JS cũ sang TS và gặp hàng ngàn lỗi đỏ.
   Giải quyết: Không nên cố sửa hết ngay. Hãy cấu hình `allowJs: true` và `checkJs: false`, sau đó chuyển đổi từng file một sang `.ts`, bắt đầu từ các util function đơn giản.
2. Tình huống: Bạn nhận dữ liệu từ một API bên thứ ba và TS báo lỗi vì dữ liệu đó không khớp với interface bạn định nghĩa.
   Giải quyết: Sử dụng **Type Assertion** (`as MyInterface`) nếu bạn chắc chắn dữ liệu đúng, hoặc tốt hơn là dùng một thư viện validate như `Zod` để kiểm tra dữ liệu ở runtime.
3. Tình huống: Bạn muốn một biến chỉ có thể nhận giá trị là "Success" hoặc "Error".
   Giải quyết: Sử dụng **Literal Union Type**: `type Status = "Success" | "Error";`. Điều này ngăn chặn việc gán nhầm các chuỗi khác.
