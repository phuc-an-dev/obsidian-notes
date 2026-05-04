---
created: 2026-04-28
tags:
  - "#type/concept"
  - "#status/done"
  - "#lang/javascript"
  - "#topic/performance"
related:
  - "[[typescript]]"
---

## 1. What
`Array.prototype.some()` là một phương thức bậc cao (higher-order function) dùng để kiểm tra xem có ít nhất một phần tử trong mảng thỏa mãn điều kiện được cung cấp bởi một hàm test hay không. Nó trả về giá trị Boolean (`true`/`false`).

## 2. Why
Trước khi có `some()`, lập trình viên thường phải dùng vòng lặp `for` hoặc `forEach` kết hợp với một biến cờ (flag) và lệnh `break` để thoát vòng lặp sớm khi tìm thấy kết quả. `some()` ra đời để cung cấp một cú pháp khai báo (declarative) ngắn gọn hơn, tự động xử lý việc ngắt vòng lặp (short-circuiting) giúp tối ưu hiệu năng và code sạch hơn.

## 3. Mental Model
Hãy tưởng tượng bạn là một bảo vệ kiểm tra vé tại cửa rạp phim cho một nhóm người. Bạn chỉ cần tìm thấy **ít nhất một người** có vé hợp lệ là có thể báo cáo "Nhóm này có người có vé" (`true`) và không cần kiểm tra những người còn lại. Chỉ khi bạn đã kiểm tra tất cả mọi người mà không ai có vé, bạn mới báo cáo "Không ai có vé" (`false`).

## 4. Where it fits
`some()` nằm trong nhóm các phương thức Iteration của Array:
`Array -> Iteration Methods -> some() / every() / find() / filter()`

Sơ đồ hoạt động:
`Input Array -> Callback Test -> Short-circuit (nếu gặp true) -> Boolean Output`

## 5. When to use
- Khi cần kiểm tra sự tồn tại của một mẫu dữ liệu trong mảng.
- Kiểm tra quyền hạn: "User có ít nhất một trong các quyền này không?".
- Validation form: "Có trường nào đang bị lỗi không?".
- Kiểm tra trùng lặp đơn giản.

## 6. When NOT to use
- Khi bạn cần lấy giá trị của phần tử đó (Hãy dùng `find()`).
- Khi bạn cần lấy tất cả các phần tử thỏa mãn (Hãy dùng `filter()`).
- Khi bạn muốn biến đổi dữ liệu (Hãy dùng `map()`).
- Khi bạn cần kiểm tra **tất cả** phần tử (Hãy dùng `every()`).
- Đừng dùng `some()` nếu bạn định thực hiện side effects (như gọi API) bên trong callback vì nó có thể ngắt quãng bất cứ lúc nào.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Ngắn gọn, dễ đọc (Declarative). | Không thể truy cập vào chỉ số của phần tử sau khi vòng lặp kết thúc. |
| Hiệu năng tốt nhờ cơ chế Short-circuiting. | Không phù hợp cho các tác vụ cần duyệt qua toàn bộ mảng (Side effects). |
| Trả về Boolean trực tiếp, phù hợp cho câu lệnh if. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `find()` | Trả về giá trị phần tử đầu tiên tìm thấy thay vì Boolean. |
| `includes()` | Chỉ kiểm tra giá trị nguyên thủy (primitive), không dùng được hàm callback để kiểm tra điều kiện phức tạp. |
| `every()` | Chỉ trả về `true` nếu **tất cả** phần tử thỏa mãn. |
| `filter()` | Duyệt qua toàn bộ mảng và trả về mảng mới, chậm hơn `some()` nếu chỉ cần kiểm tra tồn tại. |

## 9. How
```javascript
const users = [
  { id: 1, name: 'An', role: 'user' },
  { id: 2, name: 'Phuc', role: 'admin' },
  { id: 3, name: 'Gemini', role: 'user' }
];

// Kiểm tra xem trong danh sách có admin không
const hasAdmin = users.some(user => user.role === 'admin');
console.log(hasAdmin); // true

// Kiểm tra mảng rỗng: luôn trả về false cho bất kỳ điều kiện nào
const isEmpty = [].some(x => x > 0);
console.log(isEmpty); // false
```

## 10. Production concerns
### Scaling
Với mảng cực lớn (hàng triệu phần tử), `some()` nhanh hơn `filter()` hoặc `forEach()` vì nó dừng ngay khi đạt điều kiện. Tuy nhiên, nếu phần tử thỏa mãn nằm ở cuối mảng, nó vẫn có độ phức tạp O(n).

### Failure
Cẩn thận với mảng thưa (sparse arrays). `some()` sẽ bỏ qua các lỗ hổng (empty slots) trong mảng, điều này có thể dẫn đến kết quả không mong đợi nếu logic của bạn dựa trên độ dài mảng.

### Monitoring
Trong production, nếu callback của `some()` thực hiện các tính toán nặng, cần log lại thời gian thực thi để đảm bảo không chặn Event Loop quá lâu.

## 11. Common mistakes
- Mistake: Quên `return` trong callback (khi không dùng arrow function rút gọn).
  Fix: Luôn đảm bảo callback trả về một giá trị truthy/falsy.

- Mistake: Kỳ vọng `some()` chạy qua mọi phần tử để thực hiện side effect.
  Fix: Dùng `forEach()` hoặc vòng lặp `for...of` nếu cần duyệt hết mảng.

## 12. Sample project
Xây dựng một hàm kiểm tra quyền truy cập (Permission Checker) cho một ứng dụng SaaS. Hàm này nhận vào một danh sách các quyền của User và một danh sách các quyền cần thiết để truy cập một Route.
Constraint: Chỉ dùng `some()` và không dùng thư viện ngoài.

```javascript
const userPermissions = ['READ_POST', 'WRITE_POST', 'DELETE_COMMENT'];
const requiredPermissions = ['ADMIN', 'DELETE_POST'];

const canAccess = requiredPermissions.some(permission => 
  userPermissions.includes(permission)
);
```

## 13. Interview
### Core Q&A
1. Q: `some()` trả về gì nếu mảng rỗng?
   A: Luôn trả về `false` vì không có phần tử nào có thể thỏa mãn điều kiện.

2. Q: Sự khác biệt lớn nhất giữa `some()` và `find()` là gì?
   A: `some()` trả về kiểu dữ liệu Boolean (tiết kiệm bộ nhớ hơn nếu chỉ cần check), trong khi `find()` trả về giá trị của phần tử đầu tiên tìm thấy hoặc `undefined`.

3. Q: `some()` có làm thay đổi mảng gốc không?
   A: Không, `some()` là một phương thức bất biến (immutable). Tuy nhiên, callback truyền vào có thể làm thay đổi mảng nếu người viết code cố tình làm vậy (không khuyến khích).

### Scenario
"Bạn có một mảng các đối tượng chứa thông tin log hệ thống. Bạn cần cảnh báo ngay lập tức nếu có ít nhất một log có mức độ là 'CRITICAL'. Bạn sẽ chọn phương thức nào để tối ưu tốc độ?"
Trả lời: Chọn `some()`. Vì log hệ thống có thể rất nhiều, `some()` sẽ dừng ngay khi gặp log 'CRITICAL' đầu tiên, giúp phản ứng nhanh nhất có thể thay vì phải duyệt hết mảng như `filter()`.

## 14. References
- Official Docs: [MDN - Array.prototype.some()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/some)
- Spec / RFC: ECMAScript 5th Edition (ES5)

## 15. Real-world Code
Thường thấy trong các middleware của Express.js hoặc Auth Guard trong React/Next.js để kiểm tra Role.

## 16. Community
- Stack Overflow: [Difference between filter and some](https://stackoverflow.com/questions/41775602/difference-between-filter-and-some-in-javascript)
- Blog: [Understanding Short-circuit Evaluation in JavaScript](https://v8.dev/blog)
