---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/react"
  - "#topic/ui"
related:
  - "[[react-lifecycle]]"
  - "[[useState]]"
---

## 1. What
Conditional Rendering (Render theo điều kiện) là kỹ thuật trong React cho phép bạn quyết định component hoặc phần tử JSX nào sẽ được hiển thị lên màn hình dựa trên các logic cụ thể (state, props, hoặc kết quả của một biểu thức).

## 2. Why
Giao diện người dùng (UI) không phải là tĩnh. Nó thay đổi dựa trên trạng thái của ứng dụng: hiển thị icon loading khi đang fetch dữ liệu, hiển thị form đăng nhập nếu người dùng chưa login, hoặc hiển thị thông báo lỗi nếu có sự cố. Kỹ thuật này giúp UI phản ứng linh hoạt với dữ liệu.

## 3. Mental Model
Hãy tưởng tượng UI như một **kịch bản sân khấu**.
Tùy thuộc vào việc "đèn đang sáng" hay "đèn đang tắt" (Điều kiện), diễn viên (Component) sẽ ra biểu diễn hoặc đứng trong cánh gà. Đạo diễn (React) sẽ kiểm tra trạng thái của bối cảnh và ra lệnh cho diễn viên xuất hiện đúng lúc, đúng chỗ.

## 4. Where it fits
State Change -> Logic Evaluation -> **Conditional Block** -> JSX Output -> Virtual DOM Update.

## 5. When to use
- Hiển thị trạng thái Loading, Empty, hoặc Error.
- Phân quyền người dùng (Hiển thị các nút Admin/User).
- Chế độ Toggle (Hiện/Ẩn nội dung).
- Đa dạng hóa giao diện dựa trên loại thiết bị (Mobile/Desktop).

## 6. When NOT to use
- Đừng dùng quá nhiều logic điều kiện phức tạp lồng nhau ngay trong khối return của JSX (làm code khó đọc). Nên tách ra thành các biến hoặc function nhỏ.
- Khi sự thay đổi chỉ đơn thuần là CSS (hiện/ẩn bằng `display: none`), thay vì tháo gỡ hoàn toàn phần tử khỏi DOM.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Tối ưu hiệu năng (không render các phần tử thừa) | Có thể làm code JSX trở nên rối rắm nếu dùng nhiều `&&` lồng nhau |
| Bảo mật hơn (không gửi HTML nhạy cảm xuống client nếu chưa đủ quyền) | Dễ gặp lỗi render giá trị `0` hoặc `NaN` ngoài ý muốn |
| Trải nghiệm người dùng tốt, mượt mà | Khó kiểm soát animation khi phần tử biến mất đột ngột |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| CSS class (Hidden) | Phần tử vẫn tồn tại trong DOM, tốt cho SEO hoặc animation đơn giản. |
| React Portal | Render nội dung ở vị trí DOM khác nhưng vẫn giữ logic điều kiện. |

## 9. How
Các kỹ thuật phổ biến nhất:

```tsx
const MyComponent = ({ isLoggedIn, isLoading, items }: Props) => {
  // 1. If/Else (Ngoài JSX)
  if (isLoading) return <Loading />;

  return (
    <div>
      {/* 2. Ternary Operator (Toán tử 3 ngôi) */}
      <h1>{isLoggedIn ? 'Welcome back!' : 'Please login'}</h1>

      {/* 3. Logical && (Ngắn gọn cho việc 'Hiện hoặc Không') */}
      {items.length > 0 && <List data={items} />}

      {/* 4. Switch Case (Cho nhiều trạng thái) */}
      {(() => {
        switch(status) {
          case 'admin': return <AdminPanel />;
          case 'user': return <UserPanel />;
          default: return <GuestPanel />;
        }
      })()}
    </div>
  );
};
```

## 10. Production concerns
### Performance
Khi một phần tử bị gỡ khỏi DOM và gắn lại, nó sẽ phải trải qua toàn bộ vòng đời mount/unmount. Nếu việc này xảy ra quá thường xuyên với các component nặng, hãy cân nhắc dùng CSS để ẩn/hiện.

### Failure
Cẩn thận với lỗi "Flicker" (Nháy màn hình) khi trạng thái chuyển đổi quá nhanh giữa Loading và Data.

## 11. Common mistakes
- Mistake: Dùng `&&` với mảng rỗng hoặc số 0.
  Fix: Luôn dùng so sánh rõ ràng: `{items.length > 0 && ...}` thay vì `{items.length && ...}` (vì số 0 sẽ hiện lên UI).

- Mistake: Lồng toán tử 3 ngôi quá nhiều tầng.
  Fix: Tách logic ra thành các biến riêng biệt: `const content = isLoggedIn ? <Dashboard /> : <Landing />;`.

## 12. Sample project
Xây dựng một `MultiStepForm`: Mỗi bước của form chỉ hiển thị dựa trên giá trị của state `currentStep`, đảm bảo dữ liệu của các bước trước đó vẫn được lưu giữ an toàn.

## 13. Interview
### Core Q&A
1. Q: Tại sao đôi khi số `0` lại hiển thị trên màn hình khi dùng `condition && <Component />`?
   A: Vì trong JavaScript, `0` là một giá trị falsy nhưng React vẫn coi nó là một giá trị hợp lệ để render. Kết quả của biểu thức `0 && <JSX>` là `0`.

### Scenario
1. Q: Làm thế nào để render một danh sách nhưng hiển thị "No data" nếu danh sách rỗng?
   A: Tôi sẽ dùng toán tử 3 ngôi: `{items.length > 0 ? <List items={items} /> : <EmptyState />}`.

## 14. References
- React Official Docs: https://react.dev/learn/conditional-rendering
- JavaScript Logical Operators: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Logical_AND

## 15. Real-world Code
Kiểm tra cách các thư viện như `react-router` render các route khác nhau dựa trên URL hiện tại.

## 16. Community
- YouTube: "Conditional Rendering Patterns in React" - Jack Herrington.
- Blog: "Robin Wieruch - All the ways to conditional render in React".
