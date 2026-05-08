---
created: 2026-05-07
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/javascript"
  - "#topic/nextjs"
related:
  - "[[nextjs.md]]"
  - "[[code-splitting-bundling.md]]"
---

## 1. What
`transpilePackages` là một tùy chọn cấu hình trong `next.config.js` cho phép Next.js tự động biên dịch (transpile) và đóng gói (bundle) các phụ thuộc (dependencies) từ `node_modules` hoặc các gói nội bộ trong một Monorepo. Điều này có nghĩa là Next.js sẽ xử lý các gói này giống như mã nguồn cục bộ của dự án.

## 2. Why
Thông thường, Next.js (và hầu hết các công cụ build) giả định rằng các gói trong `node_modules` đã được biên dịch sẵn sang JavaScript thuần (ES5/ES6) để trình duyệt có thể hiểu được. Tuy nhiên:
- Trong mô hình Monorepo, các gói nội bộ thường chứa mã nguồn chưa biên dịch (TypeScript, JSX).
- Một số thư viện hiện đại chỉ cung cấp mã nguồn ESM hoặc TypeScript mà không biên dịch sẵn.
Trước Next.js 13, bạn phải dùng thư viện ngoài là `next-transpile-modules`. `transpilePackages` ra đời để thay thế hoàn toàn thư viện này một cách chính thống.

## 3. Mental Model
Hãy tưởng tượng Next.js là một **"Nhà máy chế biến thực phẩm"**.
- Mã nguồn trong `src/` là nguyên liệu tươi sống nhà máy tự xử lý.
- Các gói trong `node_modules` thường là thực phẩm đã đóng hộp sẵn (đã biên dịch), nhà máy chỉ việc dùng luôn.
- `transpilePackages` giống như một **"Giấy phép đặc biệt"** cho phép nhà máy nhận thêm một số nguyên liệu tươi sống từ bên ngoài (các packages cụ thể) và đưa chúng vào quy trình chế biến chuyên sâu giống như nguyên liệu nhà làm.

## 4. Where it fits
Next.js Build Process -> **next.config.js** -> `transpilePackages` list -> Webpack/Turbopack Configuration -> SWC Transpilation -> Output Bundle.

## 5. When to use
- Khi sử dụng kiến trúc Monorepo (Turborepo, Nx) và muốn chia sẻ code (UI components, utils) dưới dạng mã nguồn TypeScript/JSX giữa các ứng dụng.
- Khi sử dụng một thư viện từ npm nhưng thư viện đó chưa được biên dịch (ví dụ các thư viện chỉ cung cấp mã nguồn hiện đại chưa hỗ trợ các trình duyệt cũ).
- Khi gặp lỗi "Unexpected token" hoặc "You may need an appropriate loader to handle this file type" đối với một file nằm trong `node_modules`.

## 6. When NOT to use
- Đừng thêm tất cả mọi thứ vào `transpilePackages`. Việc biên dịch quá nhiều gói bên thứ ba sẽ làm chậm đáng kể thời gian build.
- Nếu thư viện đã được biên dịch tốt và chạy ổn định, không cần thiết phải transpile lại.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Tích hợp sẵn, không cần thư viện bên thứ ba. | Làm tăng thời gian Build và Dev startup do phải xử lý nhiều code hơn. |
| Hỗ trợ cực tốt cho Monorepo. | Khó gỡ lỗi nếu gói bên thứ ba có các cấu hình build phức tạp hoặc xung đột. |
| Giúp code sharing trở nên mượt mà. | N/A |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `next-transpile-modules` | Cách cũ (trước Next.js 13), hiện đã bị coi là deprecated. |
| Tự biên dịch package trước | Xuất bản package dưới dạng JS đã build. Tốt cho production nhưng chậm khi đang phát triển (phải build liên tục). |

## 9. How
Cấu hình trong `next.config.js`:

```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  // Danh sách các package cần được Next.js biên dịch
  transpilePackages: ['@acme/ui', 'my-shared-utils', 'some-modern-lib'],
}

module.exports = nextConfig
```

Sau khi cấu hình, bạn có thể import trực tiếp từ các package này mà không lo về lỗi cú pháp TypeScript/JSX.

## 10. Production concerns
### Build Time
Trong môi trường CI/CD, việc transpile nhiều packages lớn có thể làm vượt ngưỡng giới hạn thời gian build hoặc bộ nhớ.

### Bundle Size
Kiểm tra kỹ xem việc transpile có làm tăng kích thước bundle không mong muốn do bao gồm cả các phần không cần thiết của thư viện (Tree shaking có hoạt động tốt không).

## 11. Common mistakes
- Mistake: Gõ sai tên package hoặc thiếu scope (ví dụ gõ `ui` thay vì `@my-org/ui`).
  Fix: Luôn kiểm tra chính xác tên trong `package.json`.

- Mistake: Thêm các package quá lớn (như `lodash` hoặc `three.js`) vào mà không có lý do cụ thể.
  Fix: Chỉ thêm những package thực sự chứa mã nguồn chưa biên dịch.

## 12. Sample project
Thiết lập một Turborepo gồm:
1. `apps/web`: Ứng dụng Next.js.
2. `packages/ui`: Thư viện component chứa các file `.tsx`.
Cấu hình `transpilePackages: ['ui']` trong `apps/web/next.config.js` để ứng dụng web có thể sử dụng các component từ thư viện `ui`.

## 13. Interview
### Core Q&A
1. Q: Tại sao chúng ta cần `transpilePackages` trong Next.js?
   A: Để Next.js có thể xử lý và biên dịch các phụ thuộc (thường là từ Monorepo hoặc các gói hiện đại) chưa được compile sẵn, giúp chúng tương thích với cấu hình build của ứng dụng hiện tại.

2. Q: `transpilePackages` có hỗ trợ thay đổi nội dung (Hot Module Replacement) không?
   A: Có. Khi bạn thay đổi code trong package đã được khai báo, Next.js sẽ nhận diện và cập nhật giao diện ngay lập tức trong chế độ development.

### Scenario
"Dự án của bạn dùng một thư viện UI nội bộ trong Monorepo nhưng bị lỗi 'Module parse failed' khi chạy Next.js. Bạn sẽ làm gì?"
-> Trả lời: Tôi sẽ thêm tên của thư viện UI đó vào mảng `transpilePackages` trong file `next.config.js`. Điều này báo cho Next.js biết cần phải dùng bộ biên dịch (thường là SWC) để xử lý các file TypeScript/JSX bên trong thư viện đó.

## 14. References
- Next.js Docs: [transpilePackages configuration](https://nextjs.org/docs/app/api-reference/next-config-js/transpilePackages)
- Turborepo Guide: [Sharing code with transpilePackages](https://turbo.build/repo/docs/handbook/sharing-code)

## 15. Real-world Code
Nghiên cứu các template của Turborepo (`npx create-turbo@latest`) để thấy cách họ sử dụng `transpilePackages` để kết nối các workspace.

## 16. Community
- GitHub Discussions: Tìm kiếm "transpilePackages" trong repo của vercel/next.js.
- Stack Overflow: Tag [next.js] [monorepo].
