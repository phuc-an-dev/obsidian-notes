---
created: 2026-04-24
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/react"
  - "#topic/error-handling"
related:
  - "[[nextjs]]"
---

## 1. What
Environment & Config trong Next.js là hệ thống quản lý các biến môi trường (Environment Variables) và cấu hình ứng dụng thông qua file `next.config.js`. Nó giúp tách biệt mã nguồn khỏi các thông tin nhạy cảm (như API keys) và các thiết lập thay đổi theo môi trường (Development, Staging, Production).

## 2. Why
Một ứng dụng thường chạy trên nhiều môi trường khác nhau. Việc cứng mã (hard-code) các URL API hoặc bí mật (secrets) trực tiếp vào code không chỉ gây mất an toàn mà còn khiến việc triển khai trở nên khó khăn. Hệ thống cấu hình của Next.js cho phép thay đổi hành vi của ứng dụng mà không cần thay đổi code.

## 3. Mental Model
Hãy tưởng tượng ứng dụng của bạn là một chiếc điện thoại di động. Phần cứng và phần mềm là mã nguồn (Code). Các "Settings" (như ngôn ngữ, độ sáng, mật khẩu wifi) chính là Environment Variables & Config. Bạn có thể mang chiếc điện thoại đó sang nước khác (Môi trường khác) và chỉ cần thay đổi Settings để nó hoạt động phù hợp mà không cần cài lại hệ điều hành.

## 4. Where it fits
`.env` files / OS Env -> Next.js Build/Runtime Process -> `process.env` / `next.config.js` -> Application Code.

## 5. When to use
- Khi cần lưu trữ API Keys, Database credentials.
- Khi cần thay đổi URL của backend API giữa Local và Production.
- Khi muốn bật/tắt các tính năng thử nghiệm (Feature Flags).
- Khi cần cấu hình các proxy, redirect hoặc rewrites ở tầng framework.

## 6. When NOT to use
- Đừng để các dữ liệu lớn hoặc logic nghiệp vụ vào biến môi trường (nên dùng database hoặc file config JSON/TS riêng).
- Không dùng biến môi trường để lưu trữ các thông tin thay đổi liên tục theo từng request của người dùng.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Bảo mật thông tin nhạy cảm | Dễ gây lỗi nếu quên khai báo biến ở môi trường mới |
| Tách biệt môi trường rõ ràng | Khó debug khi biến Build-time và Runtime bị nhầm lẫn |
| Tích hợp sâu với các nền tảng CI/CD | Phải restart server/rebuild để cập nhật một số giá trị |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Hard-coded values | Cực kỳ không an toàn và thiếu linh hoạt. |
| External Config Service | (Như AWS AppConfig) Mạnh mẽ cho hệ thống lớn nhưng tăng độ trễ và phức tạp. |
| Runtime Config (Legacy) | Next.js cũ có `publicRuntimeConfig`, hiện đã bị khai tử để ưu tiên Performance. |

## 9. How
### Biến môi trường
```bash
# .env.local
DATABASE_URL="postgres://..." # Chỉ có ở Server
NEXT_PUBLIC_API_URL="https://api.com" # Lộ ra Client (nhờ tiền tố NEXT_PUBLIC_)
```

### Cấu hình next.config.js
```javascript
/** @type {import('next').NextConfig} */
const nextConfig = {
  images: {
    remotePatterns: [{ hostname: 'example.com' }],
  },
  async redirects() {
    return [{ source: '/old', destination: '/new', permanent: true }];
  },
};
module.exports = nextConfig;
```

## 10. Production concerns
### Scaling
Khi deploy lên các nền tảng serverless như Vercel, biến môi trường được quản lý qua giao diện Dashboard. Cần đảm bảo các biến bí mật không bao giờ bị commit vào Git.

### Failure
Sử dụng các thư viện như `t3-env` hoặc `zod` để validate các biến môi trường ngay khi ứng dụng khởi động (fail-fast), tránh lỗi âm thầm khi thiếu biến.

### Monitoring
Kiểm tra log của build process để đảm bảo các biến Build-time được inject đúng giá trị.

## 11. Common mistakes
- Mistake: Quên tiền tố `NEXT_PUBLIC_` khi muốn dùng biến ở Client Component.
  Fix: Luôn thêm `NEXT_PUBLIC_` nếu biến đó cần truy cập từ trình duyệt.

- Mistake: Commit file `.env` chứa bí mật lên GitHub.
  Fix: Luôn đưa các file `.env*` vào `.gitignore` (trừ `.env.example`).

## 12. Sample project
Tạo một script khởi động ứng dụng kiểm tra xem tất cả các biến bắt buộc (`DATABASE_URL`, `STRIPE_SECRET`) đã có chưa. Nếu thiếu, in ra lỗi rõ ràng và dừng quá trình build.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa biến môi trường có và không có tiền tố `NEXT_PUBLIC_` là gì?
   A: Biến không có tiền tố chỉ truy cập được ở môi trường Node.js (Server-side). Biến có tiền tố `NEXT_PUBLIC_` sẽ được Next.js nhúng vào bundle JavaScript để truy cập được ở trình duyệt (Client-side).

### Scenario
1. Q: Bạn làm thế nào để quản lý các biến môi trường khác nhau cho `preview` branch và `production` branch trên Vercel?
   A: Tôi sẽ sử dụng tính năng "Environment Variables per Environment" trên Vercel, chỉ định rõ giá trị nào dành cho Production, Preview, hoặc Development.

## 14. References
- Next.js Env Variables: https://nextjs.org/docs/app/building-your-application/configuring/environment-variables
- next.config.js options: https://nextjs.org/docs/app/api-reference/next-config-js

## 15. Real-world Code
Tham khảo cách các dự án lớn tổ chức file `env.mjs` (sử dụng Zod) để đảm bảo Type-safety cho các biến môi trường.

## 16. Community
- Blog: "Type-safe environment variables with Zod" - T3 Stack.
- GitHub: `t3-oss/t3-env` repo.
