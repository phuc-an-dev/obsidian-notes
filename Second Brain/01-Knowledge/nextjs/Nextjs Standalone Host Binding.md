---
created: 2026-05-07
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/javascript"
  - "#topic/nextjs"
related:
  - "[[nextjs.md]]"
  - "[[Docker in Ubuntu.md]]"
---

## 1. What
Trong Next.js 14 (và các bản gần đây), khi sử dụng chế độ `output: 'standalone'`, file `server.js` được tạo ra mặc định sẽ lắng nghe (bind) trên địa chỉ loopback `127.0.0.1` thay vì `0.0.0.0` nếu không có cấu hình biến môi trường `HOSTNAME`.

## 2. Why
Việc bind vào `127.0.0.1` là một biện pháp bảo mật mặc định để đảm bảo server chỉ chấp nhận kết nối từ chính máy đang chạy nó. Tuy nhiên, điều này gây ra vấn đề nghiêm trọng trong môi trường **Docker** hoặc **Cloud**, nơi container/instance cần lắng nghe trên tất cả các network interfaces (`0.0.0.0`) để các yêu cầu từ bên ngoài (qua port mapping) có thể đi tới được ứng dụng.

## 3. Mental Model
Hãy tưởng tượng server của bạn là một **"Quầy thông tin"** bên trong một tòa nhà (Docker Container).
- **`127.0.0.1`**: Quầy chỉ trả lời những người đứng ngay sát quầy bên trong tòa nhà. Người đứng ngoài cửa tòa nhà (Internet/Docker Host) gọi vào sẽ không ai nghe thấy.
- **`0.0.0.0`**: Quầy bật loa phóng thanh hướng ra tất cả các cửa. Bất kể ai gọi từ đâu tới, quầy cũng sẽ nghe thấy và trả lời.

## 4. Where it fits
Build Process (`next build`) -> Output `standalone` folder -> `node server.js` -> **Network Interface Binding**.

## 5. When to use
Vấn đề này luôn xuất hiện khi:
- Triển khai Next.js standalone trong Docker.
- Triển khai trên Kubernetes.
- Chạy trên EC2 hoặc VPS nằm sau một Reverse Proxy mà không dùng `localhost`.

## 6. When NOT to use
- Khi bạn chạy server trực tiếp trên máy tính cá nhân để test nội bộ và muốn bảo mật tối đa (chỉ máy bạn truy cập được).

## 7. Trade-offs
| Host Binding | Pros | Cons |
|------|------|------|
| **127.0.0.1** | Bảo mật cao, tránh lộ port ra internet. | Không thể truy cập từ bên ngoài Docker container. |
| **0.0.0.0** | Truy cập được từ mọi nơi, cần thiết cho Docker/Cloud. | Có thể lộ port nếu không có Firewall/Security Group che chắn. |

## 8. Alternatives
- Sử dụng Reverse Proxy (Nginx) trên cùng một mạng nội bộ để chuyển tiếp traffic vào `127.0.0.1`, nhưng cách này phức tạp hơn việc chỉ đơn giản là đổi host binding.

## 9. How
Để khắc phục lỗi không thể truy cập ứng dụng Next.js trong Docker, bạn cần đặt biến môi trường `HOSTNAME` thành `0.0.0.0`.

### Cách 1: Trong Dockerfile
```dockerfile
# ... các bước build trước đó ...
ENV HOSTNAME="0.0.0.0"
ENV PORT=3000

CMD ["node", "server.js"]
```

### Cách 2: Trong docker-compose.yml
```yaml
services:
  web:
    image: my-nextjs-app
    environment:
      - HOSTNAME=0.0.0.0
      - PORT=3000
    ports:
      - "3000:3000"
```

## 10. Production concerns
### Health Checks
Nếu Docker health check của bạn gọi tới `localhost:3000` nhưng server lại bind vào `0.0.0.0`, nó vẫn hoạt động bình thường vì `0.0.0.0` bao hàm cả `localhost`.

### Cloud Services
Các dịch vụ như Google Cloud Run hoặc AWS Fargate thường yêu cầu server phải lắng nghe trên `0.0.0.0`. Nếu không cấu hình `HOSTNAME`, dịch vụ sẽ báo lỗi "Port not reached" và không thể khởi động.

## 11. Common mistakes
- Mistake: Chỉ map port trong Docker (`-p 3000:3000`) mà quên set `HOSTNAME=0.0.0.0`. Ứng dụng chạy nhưng trình duyệt báo "Connection Refused".
  Fix: Luôn set `HOSTNAME=0.0.0.0` khi đóng gói Docker.

- Mistake: Nhầm lẫn giữa `HOST` và `HOSTNAME`. Next.js standalone sử dụng `HOSTNAME`.

## 12. Sample project
Tạo một dự án Next.js 14, cấu hình `next.config.js`:
```javascript
module.exports = {
  output: 'standalone',
}
```
Build và chạy thử trong Docker mà không có biến `HOSTNAME`, sau đó thêm biến vào để thấy sự khác biệt.

## 13. Interview
### Core Q&A
1. Q: Tại sao Next.js standalone mặc định bind vào 127.0.0.1?
   A: Đây là cài đặt mặc định an toàn của Node.js server được Next.js kế thừa, nhằm tránh việc vô tình mở rộng bề mặt tấn công khi không cần thiết.

2. Q: Ý nghĩa của địa chỉ 0.0.0.0 là gì?
   A: Nó không phải là một địa chỉ IP thực tế để truy cập, mà là một ký hiệu cho server biết: "Hãy lắng nghe trên tất cả các card mạng (IP) hiện có của máy này".

### Scenario
"Bạn deploy Next.js standalone lên Docker, log báo 'Listening on http://localhost:3000' nhưng bạn không thể truy cập từ máy host qua http://localhost:3000. Bạn giải thích và xử lý thế nào?"
-> Trả lời: Lỗi do server đang bind vào loopback interface (`127.0.0.1`) bên trong container, nên traffic từ máy host (đi qua bridge network) không thể chạm tới. Tôi sẽ thêm biến môi trường `HOSTNAME=0.0.0.0` để server lắng nghe trên card mạng của container.

## 14. References
- Next.js Docs: [Output Standalone](https://nextjs.org/docs/app/api-reference/next-config-js/output#standalone)
- GitHub Issue: [Next.js standalone default host discussion](https://github.com/vercel/next.js/discussions/41974)

## 15. Real-world Code
Nghiên cứu file `server.js` trong thư mục `.next/standalone/server.js` sau khi build để thấy đoạn code Node.js xử lý `process.env.HOSTNAME`.

## 16. Community
- Stack Overflow: [Can't access Next.js app in Docker standalone mode](https://stackoverflow.com/questions/75815183/)
- Reddit: r/nextjs - "Standalone mode tips and tricks".
