---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/others"
  - "#topic/configuration"
related:
  - "[[github-secrets.md]]"
  - "[[Nginx Configuration for Spring Boot in EC2.md]]"
  - "[[github-environment-secrets.md]]"
  - "[[yaml.md]]"
---

## 1. What
Environment Variables (Biến môi trường) là các cặp khóa-giá trị (key-value pairs) được lưu trữ bên ngoài mã nguồn ứng dụng, trong môi trường hệ điều hành hoặc môi trường thực thi (Runtime). Chúng được sử dụng để cung cấp các thông tin cấu hình động cho ứng dụng mà không cần thay đổi code.

## 2. Why
Việc hardcode cấu hình (như địa chỉ database, cổng server) trực tiếp vào mã nguồn là một "anti-pattern" nghiêm trọng. Biến môi trường ra đời để:
- **Tách biệt cấu hình và mã nguồn**: Một bản build ứng dụng duy nhất có thể chạy trên nhiều môi trường (Dev, Test, Prod) chỉ bằng cách thay đổi biến môi trường.
- **Bảo mật**: Tránh lộ thông tin nhạy cảm khi push code lên các kho lưu trữ công khai.
- **Linh hoạt**: Dễ dàng thay đổi thông số vận hành của ứng dụng mà không cần biên dịch lại (recompile).

## 3. Mental Model
Hãy tưởng tượng ứng dụng của bạn giống như một **"Chiếc điều hòa nhiệt độ"**:
- Mã nguồn (Code) là các bảng mạch và động cơ bên trong máy.
- Biến môi trường (Env Vars) là **các nút bấm trên điều khiển từ xa**.
- Bạn không cần phải tháo tung máy ra để chỉnh nhiệt độ (Sửa code), bạn chỉ cần bấm nút trên điều khiển (Thay đổi biến môi trường) để máy hoạt động theo ý muốn.

## 4. Where it fits
Vị trí trong kiến trúc ứng dụng:
`OS / Container / CI-CD -> Environment Variables -> Application Framework (Spring Boot/Node.js) -> Logic`

## 5. When to use
- Lưu trữ các thông số kết nối Database (Host, Port, Username).
- Lưu trữ các API Keys của dịch vụ bên thứ ba (AWS, Stripe, SendGrid).
- Bật/tắt các tính năng (Feature Flags).
- Định nghĩa môi trường hiện tại (`NODE_ENV=production`, `SPRING_PROFILES_ACTIVE=prod`).

## 6. When NOT to use
- Đối với các cấu hình tĩnh, không bao giờ thay đổi và không nhạy cảm (nên dùng file `.properties` hoặc `.yaml` nội bộ).
- Khi lượng cấu hình quá lớn và phức tạp (nên cân nhắc dùng Configuration Server như Spring Cloud Config hoặc HashiCorp Consul).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Dễ dàng thay đổi và triển khai. | Dễ bị lộ nếu in toàn bộ biến môi trường ra logs. |
| Được hỗ trợ bởi hầu hết mọi ngôn ngữ và nền tảng. | Khó quản lý phiên bản (Versioning) của các biến. |
| Tuân thủ nguyên tắc "Twelve-Factor App". | Có thể gây xung đột nếu nhiều ứng dụng dùng chung tên biến trên cùng một host. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Configuration Files | Dễ quản lý bằng Git nhưng khó bảo mật và khó thay đổi động. |
| Secret Managers | Bảo mật cực cao nhưng yêu cầu gọi API để lấy giá trị, làm tăng độ phức tạp của code. |

## 9. How
Cách sử dụng biến môi trường phổ biến:

### Trong Linux Shell
```bash
export DB_URL="jdbc:postgresql://localhost:5432/mydb"
echo $DB_URL
```

### Trong Node.js
```javascript
const dbUrl = process.env.DB_URL;
```

### Trong Spring Boot (application.properties)
```properties
spring.datasource.url=${DB_URL}
```

### Sử dụng file .env (Local development)
```text
DB_URL=jdbc:postgresql://localhost:5432/mydb
SECRET_KEY=super-secret-string
```

## 10. Production concerns
### Priority
Hầu hết các framework (như Spring Boot) ưu tiên biến môi trường của hệ điều hành hơn là các giá trị trong file cấu hình. Hãy tận dụng điều này để override cấu hình khi deploy.

### Immutability in Containers
Trong Docker/Kubernetes, biến môi trường thường được gán cứng khi khởi tạo container. Để thay đổi, bạn phải restart hoặc recreate pod.

## 11. Common mistakes
Hardcode default values nhạy cảm trong code:
- Mistake: `String pass = System.getenv("PASS") != null ? System.getenv("PASS") : "admin123";`
- Fix: Không bao giờ để giá trị mặc định cho các biến nhạy cảm. Ứng dụng nên fail ngay khi khởi động nếu thiếu biến môi trường quan trọng.

## 12. Sample project
Tạo một ứng dụng "Greeting App":
1. Đọc biến `USER_NAME` từ môi trường.
2. Nếu không có, in ra "Hello Guest".
3. Viết Dockerfile để nhận biến này thông qua lệnh `docker run -e USER_NAME=AnPhuc`.

## 13. Interview
### Core Q&A
1. Q: Tại sao biến môi trường lại quan trọng trong kiến trúc Microservices?
   A: Vì mỗi dịch vụ có thể được nhân bản thành hàng trăm instance. Việc quản lý cấu hình qua biến môi trường (thường qua Kubernetes ConfigMaps/Secrets) giúp triển khai đồng loạt và nhất quán.

2. Q: Làm thế nào để truyền biến môi trường vào một Docker container?
   A: Sử dụng flag `-e` hoặc `--env-file` khi chạy lệnh `docker run`.

### Scenario
"Làm sao bạn đảm bảo các biến môi trường nhạy cảm không bị lộ khi một developer khác clone code của bạn từ GitHub?"
-> Trả lời: Tôi sẽ đưa tất cả các biến vào file `.env` và thêm file này vào `.gitignore`. Đồng thời, tôi tạo một file `.env.example` chứa các tên biến nhưng không có giá trị thật để các developer khác biết cần cấu hình những gì.

## 14. References
- The Twelve-Factor App: [III. Config](https://12factor.net/config)
- Node.js Docs: [process.env](https://nodejs.org/api/process.html#process_process_env)

## 15. Real-world Code
Nghiên cứu các thư viện như `dotenv` (Node.js) hoặc `direnv` (Linux) để thấy cách quản lý biến môi trường chuyên nghiệp.

## 16. Community
- Stack Overflow: Tag [environment-variables].
- Dev.to: Các bài viết về "Configuration Management".
