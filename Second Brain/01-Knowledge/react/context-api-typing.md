---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/react"
  - "#topic/state-management"
related:
  - "[[typescript]]"
  - "[[props-typing]]"
  - "[[use-state-typing]]"
---

## 1. What
Context API là một tính năng của React cho phép chia sẻ dữ liệu (state, functions) xuyên suốt cây component mà không cần phải truyền props thủ công qua từng cấp (prop drilling). Typing cho Context đảm bảo rằng các component tiêu thụ dữ liệu (consumers) luôn nhận đúng cấu trúc dữ liệu đã định nghĩa.

## 2. Why
Khi ứng dụng lớn dần, việc truyền dữ liệu từ component cha xuống component cháu chắt qua nhiều lớp trung gian (Prop Drilling) trở nên cực kỳ khó bảo trì và dễ gây lỗi. Context API giải quyết vấn đề này bằng cách tạo ra một "kho lưu trữ dùng chung". TypeScript giúp ngăn chặn lỗi khi truy cập Context ở những nơi không có Provider hoặc khi cấu hình dữ liệu sai lệch.

## 3. Mental Model
Hãy tưởng tượng Context API như một **Đài phát thanh (Provider)**.
Các component trong hệ thống giống như những chiếc **Radio (Consumers)**. Đài phát thanh phát sóng trên một tần số cụ thể (Context Type). Bất kỳ chiếc radio nào bật đúng tần số đó đều có thể nghe thấy thông tin mà không cần phải kéo dây trực tiếp từ đài phát thanh đến từng nhà.

## 4. Where it fits
Create Context -> Define Type -> Wrap App with Provider -> Use Context in Child Components.

## 5. When to use
- Quản lý các trạng thái toàn cục (Global State): Theme (Sáng/Tối), Thông tin người dùng (Auth), Ngôn ngữ (i18n).
- Khi dữ liệu cần được truy cập bởi rất nhiều component ở các cấp độ khác nhau.

## 6. When NOT to use
- Đừng dùng Context cho các trạng thái chỉ dùng trong một phạm vi nhỏ (nên dùng Props hoặc State nâng cao).
- Tránh dùng Context cho các dữ liệu thay đổi với tần suất cực cao (ví dụ: tọa độ chuột, input typing) vì nó gây re-render toàn bộ các component con đang tiêu thụ Context đó.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Giải quyết triệt để vấn đề Prop Drilling | Gây Re-render không mong muốn nếu không tối ưu |
| Tích hợp sẵn trong React, không cần thư viện ngoài | Làm cho component khó tái sử dụng độc lập (bị phụ thuộc vào Context) |
| Typing giúp Intellisense hoạt động hoàn hảo | Cấu hình Boilerplate ban đầu hơi nhiều với TS |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Zustand / Redux | Quản lý state mạnh mẽ hơn, tối ưu performance tốt hơn cho app lớn. |
| Composition | Bọc component thay vì truyền data, giữ cho component thuần khiết hơn. |

## 9. How
Cách định nghĩa Context an toàn với TypeScript:

```tsx
import React, { createContext, useContext, useState, ReactNode } from 'react';

// 1. Định nghĩa Interface cho dữ liệu trong Context
interface ThemeContextType {
  theme: 'light' | 'dark';
  toggleTheme: () => void;
}

// 2. Tạo Context với giá trị mặc định là undefined
const ThemeContext = createContext<ThemeContextType | undefined>(undefined);

// 3. Tạo Provider Component
export const ThemeProvider = ({ children }: { children: ReactNode }) => {
  const [theme, setTheme] = useState<'light' | 'dark'>('light');
  const toggleTheme = () => setTheme(prev => prev === 'light' ? 'dark' : 'light');

  return (
    <ThemeContext.Provider value={{ theme, toggleTheme }}>
      {children}
    </ThemeContext.Provider>
  );
};

// 4. Tạo Custom Hook để sử dụng Context an toàn (Check undefined)
export const useTheme = () => {
  const context = useContext(ThemeContext);
  if (!context) {
    throw new Error('useTheme must be used within a ThemeProvider');
  }
  return context;
};
```

## 10. Production concerns
### Performance
Mỗi khi `value` của Provider thay đổi, tất cả các component sử dụng `useContext` đó đều re-render. Nên tách nhỏ các Context (ví dụ: `UserContext`, `ThemeContext`) thay vì gộp chung vào một cái duy nhất.

### Maintenance
Sử dụng pattern Custom Hook (`useTheme`) giúp che giấu logic kiểm tra `undefined` và làm code ở phía component sạch hơn.

## 11. Common mistakes
- Mistake: Để giá trị mặc định của `createContext` là một object rỗng `{}` khi chưa có dữ liệu thực tế.
  Fix: Sử dụng `undefined` và kiểm tra lỗi trong Custom Hook để báo lỗi sớm nếu quên bọc Provider.

- Mistake: Truyền một object mới tạo trực tiếp vào `value={{...}}` mà không dùng `useMemo`.
  Fix: Nếu object đó chứa các giá trị tính toán phức tạp, hãy dùng `useMemo` để giữ tham chiếu ổn định.

## 12. Sample project
Tạo một ứng dụng Multi-language: Sử dụng Context để lưu ngôn ngữ hiện tại (`vi`/`en`) và một hàm `t(key)` để dịch chuỗi. Bọc toàn bộ ứng dụng và cho phép người dùng đổi ngôn ngữ từ bất kỳ đâu.

## 13. Interview
### Core Q&A
1. Q: Làm thế nào để tránh Re-render tất cả component khi dùng Context?
   A: Tách nhỏ các Context, sử dụng `React.memo` cho các component con, hoặc sử dụng các thư viện state management chuyên dụng nếu bài toán quá phức tạp.

### Scenario
1. Q: Bạn sẽ làm gì nếu cần chia sẻ state giữa 2 component cách xa nhau nhưng không muốn dùng Context cho toàn bộ App?
   A: Tôi sẽ tạo một Provider bọc quanh node cha chung gần nhất của 2 component đó thay vì bọc ở tầng Root (App). Điều này giúp giới hạn phạm vi ảnh hưởng của Context.

## 14. References
- React Context Documentation: https://react.dev/learn/passing-data-deeply-with-context
- TypeScript Context Guide: https://react-typescript-cheatsheet.netlify.app/docs/basic/getting-started/context/

## 15. Real-world Code
Kiểm tra mã nguồn của `next-auth` để xem cách họ dùng `SessionProvider` để cung cấp thông tin đăng nhập cho toàn bộ ứng dụng Next.js.

## 16. Community
- YouTube: "React Context API with TypeScript" - Net Ninja.
- Blog: "Kent C. Dodds - How to use React Context effectively".
