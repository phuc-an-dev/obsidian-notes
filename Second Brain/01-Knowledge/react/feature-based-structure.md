---
created: 2026-04-24
tags:
  - "#type/pattern"
  - "#status/draft"
  - "#lang/react"
  - "#topic/architecture"
related:
  - "[[framework-agnostic]]"
---

## 1. What
Feature-based structure là cách tổ chức thư mục trong project React dựa trên các tính năng nghiệp vụ (Features) thay vì tổ chức theo loại file kỹ thuật (Components, Hooks, Services). Mỗi tính năng sẽ có một thư mục riêng chứa tất cả các thành phần cần thiết để tính năng đó hoạt động.

## 2. Why
Với cấu trúc truyền thống (tất cả component vào thư mục `components`), project càng lớn thì các thư mục này càng trở nên khổng lồ, khó tìm kiếm và khó quản lý sự phụ thuộc. Feature-based giúp đóng gói (encapsulate) logic, làm cho code dễ mở rộng, dễ xóa bỏ (nếu tính năng không còn dùng) và giảm thiểu xung đột giữa các team làm việc trên các tính năng khác nhau.

## 3. Mental Model
Hãy tưởng tượng bạn đang xây dựng một **Thành phố (Project)**.
- **Cấu trúc truyền thống**: Bạn gom tất cả gạch vào một đống, tất cả gỗ vào một đống, tất cả thợ điện vào một khu. Khi xây nhà, bạn phải chạy qua chạy lại giữa các đống này.
- **Cấu trúc Feature-based**: Bạn chia thành phố thành các **Khu đô thị (Features)** như Khu dân cư, Khu công nghiệp, Khu mua sắm. Mỗi khu có đầy đủ gạch, gỗ và thợ riêng của mình. Khi cần sửa khu dân cư, bạn chỉ cần đến đúng khu đó mà không làm phiền các khu khác.

## 4. Where it fits
`src/` -> `features/` -> `[feature-name]/` -> `components/`, `hooks/`, `api/`, `types/`, `index.ts`.

## 5. When to use
- Dự án quy mô trung bình đến lớn (Enterprise applications).
- Làm việc theo mô hình Agile/Scrum nơi mỗi team phụ trách một module nghiệp vụ.
- Khi muốn áp dụng kiến trúc sạch (Clean Architecture) và giảm sự phụ thuộc lẫn nhau giữa các module.

## 6. When NOT to use
- Dự án cực nhỏ, chỉ có vài ba trang đơn giản.
- Các bản prototype nhanh, hackathon nơi tốc độ quan trọng hơn cấu trúc.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Dễ dàng mở rộng mà không làm rối project | Cấu trúc thư mục ban đầu trông có vẻ rườm rà |
| Cô lập lỗi tốt, dễ viết unit test theo feature | Đôi khi khó quyết định một component thuộc feature nào |
| Tăng tính minh bạch trong quản lý dependencies | Cần tuân thủ quy tắc nghiêm ngặt để tránh circular dependencies |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Atomic Design | Tổ chức theo độ phức tạp của UI (Atoms, Molecules...), mạnh về UI kit nhưng yếu về business logic. |
| MVC (Model-View-Controller) | Tổ chức theo vai trò của code, thường thấy trong các framework backend. |

## 9. How
Cấu trúc thư mục chuẩn:

```text
src/
  features/
    auth/               # Tính năng xác thực
      api/              # Các hàm gọi API đăng nhập, đăng ký
      components/       # UI chỉ dùng cho Auth (LoginForm, RegisterForm)
      hooks/            # useAuth, useSession
      types/            # Interface User, AuthResponse
      index.ts          # Public API của feature này
    products/           # Tính năng quản lý sản phẩm
      api/
      components/
      ...
  components/           # Shared components dùng cho toàn app (Button, Input)
  hooks/                # Shared hooks (useWindowSize, useLocalStorage)
  utils/                # Các hàm helper dùng chung
```

## 10. Production concerns
### Maintenance
Luôn sử dụng file `index.ts` (Barrel file) làm cổng ra duy nhất cho mỗi feature. Các feature khác chỉ được import từ file `index.ts` này để đảm bảo tính đóng gói.

### Scaling
Khi một feature trở nên quá lớn, bạn có thể chia nhỏ nó thành các sub-features bên trong theo cùng mô hình.

## 11. Common mistakes
- Mistake: Để shared components vào thư mục của một feature cụ thể.
  Fix: Nếu component được dùng bởi 2 feature trở lên, hãy đưa nó vào thư mục `src/components`.

- Mistake: Import chéo (Circular dependencies) giữa các features.
  Fix: Sử dụng công cụ `eslint-plugin-import` để phát hiện và ngăn chặn việc các feature phụ thuộc lẫn nhau một cách trực tiếp.

## 12. Sample project
Xây dựng một ứng dụng E-commerce: Chia thành các features `catalog`, `cart`, `checkout`, `profile`. Đảm bảo người dùng có thể xóa bỏ hoàn toàn feature `cart` mà không làm code của `catalog` bị lỗi biên dịch.

## 13. Interview
### Core Q&A
1. Q: Tại sao cấu trúc Feature-based lại giúp code "deletable" (dễ xóa)?
   A: Vì tất cả code liên quan đến một tính năng (UI, logic, api) đều nằm trong một thư mục duy nhất. Khi không dùng tính năng đó, bạn chỉ cần xóa thư mục đó và kiểm tra lại file `index.ts` mà không phải đi lục lọi ở khắp nơi trong project.

### Scenario
1. Q: Bạn làm thế nào để chia sẻ một function `formatCurrency` giữa feature `cart` và `catalog`?
   A: Tôi sẽ đưa function đó vào `src/utils` vì đây là một logic tiện ích dùng chung cho toàn bộ ứng dụng, không thuộc về một nghiệp vụ cụ thể nào.

## 14. References
- Bulletproof React: https://github.com/alan2207/bulletproof-react
- Software Engineering StackExchange - Folder Structure: https://softwareengineering.stackexchange.com/questions/338553

## 15. Real-world Code
Nghiên cứu cấu trúc thư mục của project `Bulletproof React` trên GitHub - đây là tiêu chuẩn vàng cho cấu trúc feature-based hiện nay.

## 16. Community
- Blog: "Feature-Based Folder Structure in React" - Robin Wieruch.
- Talk: "Architecting React Apps" tại các hội thảo React Conf.
