---
created: 2026-04-23
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/react"
  - "#topic/state-management"
related: "[[tanstack-react-query]]"
---

## 1. What
React Hook Form là một thư viện quản lý form trong React tập trung vào hiệu năng, tính linh hoạt và khả năng mở rộng. Nó sử dụng uncontrolled components thông qua refs để giảm thiểu số lần re-render không cần thiết so với các giải pháp controlled components truyền thống.

## 2. Why
Trước khi có React Hook Form, việc quản lý form trong React thường gặp các vấn đề:
- **Hiệu năng kém**: Mỗi lần người dùng gõ phím, toàn bộ form hoặc component cha bị re-render do state thay đổi (Controlled Components).
- **Boilerplate code**: Phải viết quá nhiều code để quản lý `value` và `onChange` cho từng input.
- **Validation phức tạp**: Tự viết logic validation tốn thời gian và khó bảo trì.
- **Độ phức tạp tăng dần**: Khi form có hàng chục trường dữ liệu hoặc các trường động (Field Array), việc quản lý state trở nên cực kỳ khó khăn.

## 3. Mental Model
Hãy tưởng tượng React Hook Form như một **Người Quản Trò (Registry)**. Thay vì bạn phải tự tay cầm từng món đồ (input) và theo dõi xem nó đang ở đâu (state), bạn chỉ cần dán một cái nhãn (register) lên món đồ đó và đưa cho Người Quản Trò. Khi cần lấy thông tin hoặc kiểm tra xem món đồ có hợp lệ không, bạn chỉ việc hỏi Người Quản Trò. Người Quản Trò này chỉ đứng nhìn và chỉ lên tiếng khi thực sự cần thiết (submit hoặc validation fail), giúp không gian (component) luôn yên tĩnh và nhẹ nhàng.

## 4. Where it fits
React Hook Form nằm ở lớp giao diện người dùng, đóng vai trò cầu nối giữa Input UI và Data Schema.
`User Input -> React Hook Form (Register/Controller) -> Schema Validation (Zod/Yup) -> Submit Handler -> API (TanStack Query)`

## 5. When to use
- Các form có quy mô trung bình đến lớn, nhiều trường dữ liệu.
- Khi cần tối ưu hiệu năng (giảm re-render).
- Cần tích hợp chặt chẽ với các thư viện validation như Zod hoặc Yup.
- Các form động (dynamic fields) nơi người dùng có thể thêm/bớt hàng loạt input.

## 6. When NOT to use
- Các form cực kỳ đơn giản (chỉ có 1-2 input và không cần validation).
- Khi bạn cần kiểm soát tuyệt đối từng keystroke để thực hiện logic side-effect phức tạp mà việc sử dụng `watch` hoặc `subscription` của thư viện trở nên rườm rà hơn state thông thường.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Hiệu năng vượt trội do giảm re-render. | Khó debug hơn vì state không nằm trực tiếp trong React State tree truyền thống. |
| Bundle size nhỏ (không phụ thuộc bên thứ ba). | Cần hiểu về Uncontrolled Components và Refs. |
| API đơn giản, dễ học và sử dụng. | Việc tích hợp với các UI Library (MUI, AntD) cần dùng thêm component Controller. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Formik | Phổ biến, dùng Controlled Components, re-render nhiều hơn và bundle size lớn hơn. |
| React Final Form | Dùng Pattern Subscription, hiệu năng tốt nhưng API phức tạp hơn. |
| Native HTML Form | Đơn giản nhất, không cần thư viện nhưng khó quản lý validation và state phức tạp. |

## 9. How
```tsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import * as z from "zod";

const schema = z.object({
  username: z.string().min(3, "Username must be at least 3 characters"),
  email: z.string().email("Invalid email address"),
});

type FormData = z.infer<typeof schema>;

export const LoginForm = () => {
  const {
    register,
    handleSubmit,
    formState: { errors },
  } = useForm<FormData>({
    resolver: zodResolver(schema),
  });

  const onSubmit = (data: FormData) => {
    console.log(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <input {...register("username")} placeholder="Username" />
        {errors.username && <span>{errors.username.message}</span>}
      </div>
      
      <div>
        <input {...register("email")} placeholder="Email" />
        {errors.email && <span>{errors.email.message}</span>}
      </div>

      <button type="submit">Submit</button>
    </form>
  );
};
```

## 10. Production concerns
### Scaling
Sử dụng `FormProvider` để chia nhỏ form lớn thành nhiều sub-components mà không cần truyền props sâu (prop drilling).

### Failure
Xử lý lỗi validation từ phía server bằng cách sử dụng hàm `setError` để map lỗi từ API trả về vào đúng từng field cụ thể.

### Monitoring
Sử dụng `formState.isSubmitting` và `formState.isValid` để quản lý trạng thái UI (loading states, disabled buttons).

## 11. Common mistakes
- Mistake: Gọi hàm `register` bên trong một vòng lặp hoặc điều kiện mà không cung cấp key ổn định.
  Fix: Luôn đảm bảo `register` được gọi ổn định hoặc sử dụng `useFieldArray` cho các trường động.

- Mistake: Không sử dụng `Controller` khi tích hợp với các thư viện UI bên ngoài như MUI hoặc Select2.
  Fix: Sử dụng component `Controller` hoặc hook `useController` để wrap các component không hỗ trợ ref hoặc là controlled component.

## 12. Sample project
Xây dựng một "Dynamic Survey Builder":
- Người dùng có thể thêm/xóa câu hỏi.
- Mỗi câu hỏi có nhiều loại (Text, Multiple Choice).
- Validation: Mỗi loại câu hỏi có luật validation riêng.
- Constraint: Chỉ sử dụng `useFieldArray` và tối ưu re-render để ngay cả khi có 100 câu hỏi, việc nhập liệu vẫn mượt mà.

## 13. Interview
### Core Q&A
1. Q: Tại sao React Hook Form lại nhanh hơn Formik?
   A: React Hook Form sử dụng Refs để thu thập dữ liệu từ input mà không gây ra re-render toàn bộ component khi người dùng nhập liệu. Formik dựa vào state của React, dẫn đến việc re-render lại component mỗi khi có thay đổi trong bất kỳ field nào.

2. Q: Khi nào bạn nên sử dụng `watch` và khi nào nên sử dụng `getValues`?
   A: `watch` sẽ trigger re-render khi giá trị thay đổi, phù hợp khi bạn cần hiển thị dữ liệu thay đổi lên UI ngay lập tức (như preview). `getValues` lấy giá trị hiện tại mà không gây re-render, phù hợp khi bạn chỉ cần đọc giá trị trong một hàm xử lý (như khi bấm nút click).

### Scenario
Tình huống: Bạn có một form đăng ký rất dài với 50 trường. Khách hàng phàn nàn rằng trình duyệt bị lag khi họ gõ vào các trường cuối cùng. Bạn sẽ giải quyết thế nào?
Giải pháp: Chuyển đổi form sang React Hook Form. Sử dụng `register` thay vì controlled state. Chia nhỏ form thành các component nhỏ hơn và sử dụng `shouldUnregister: false` nếu cần giữ lại dữ liệu khi các field bị ẩn/hiện. Sử dụng `useDeferredValue` của React nếu có các logic tính toán nặng đi kèm.

## 14. References
- Official Docs: https://react-hook-form.com/
- GitHub Repo: https://github.com/react-hook-form/react-hook-form
- Spec / RFC: Uncontrolled Components (React Docs)
- Changelog: https://github.com/react-hook-form/react-hook-form/releases

## 15. Real-world Code
- Shadcn/UI Form Component: Sử dụng React Hook Form làm core quản lý.
- Bulletproof React: Pattern quản lý form trong project quy mô lớn.

## 16. Community
- Reddit: r/reactjs
- Stack Overflow: Tag [react-hook-form]
- Blog: Bill Luo (creator) personal blog and medium.
- Talk: React Hook Form - Performant, flexible and extensible forms with hooks.
