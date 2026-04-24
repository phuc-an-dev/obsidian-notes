---
created: 2026-04-22
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/http"
related: "[[RESTful API]]"
---

## 1. What
Nginx Reverse Proxy là một cấu hình mà trong đó Nginx đóng vai trò là một máy chủ trung gian đứng trước các máy chủ backend (như Node.js, Java Spring, Python App). Nó nhận yêu cầu từ client, chuyển tiếp tới backend phù hợp, sau đó nhận phản hồi và gửi lại cho client.


## 2. Why
Trước khi có Reverse Proxy, việc để các ứng dụng backend tiếp xúc trực tiếp với internet gặp nhiều vấn đề: bảo mật kém, khó quản lý SSL cho nhiều dịch vụ, và không có khả năng cân bằng tải. Nginx Reverse Proxy ra đời để giải quyết các bài toán về bảo mật, hiệu năng và khả năng mở rộng.


## 3. Mental Model
Hãy tưởng tượng Nginx như một **"Lễ tân"** trong một tòa nhà văn phòng. Khách hàng (Client) không thể đi thẳng vào phòng của nhân viên (Backend Server). Họ phải gặp Lễ tân, Lễ tân sẽ kiểm tra yêu cầu và chỉ dẫn họ đến đúng phòng hoặc tự mình mang tài liệu vào và mang kết quả ra cho khách. Nhân viên bên trong không cần biết mặt khách hàng, họ chỉ làm việc với Lễ tân.


## 4. Where it fits
Client (Browser) -> Internet -> **Nginx (Reverse Proxy)** -> Internal Network -> Backend Server (App 1, App 2).


## 5. When to use
- Khi cần ẩn địa chỉ IP và cấu trúc hệ thống backend bên trong (Security).
- Khi muốn chạy nhiều ứng dụng trên cùng một server nhưng dùng chung cổng 80/443.
- Khi cần cài đặt chứng chỉ SSL tại một nơi duy nhất (SSL Termination).
- Khi cần cân bằng tải (Load Balancing) giữa nhiều instance của backend.
- Khi cần nén dữ liệu (Gzip) hoặc cache nội dung tĩnh để giảm tải cho backend.


## 6. When NOT to use
- Đối với các ứng dụng cực kỳ đơn giản, chạy local hoặc không yêu cầu bảo mật/scale (tuy nhiên vẫn khuyến khích dùng).
- Khi ứng dụng yêu cầu kết nối TCP/UDP đặc thù mà Nginx không hỗ trợ tốt bằng các tool chuyên dụng khác (dù Nginx hiện nay đã hỗ trợ stream module).


## 7. Trade-offs
| Pros | Cons |
|------|------|
| Tăng cường bảo mật (Anonymity). | Thêm một điểm lỗi (Single Point of Failure) nếu không cấu hình High Availability. |
| SSL Termination tập trung. | Thêm một chút độ trễ (latency) do phải qua trung gian. |
| Dễ dàng scale backend mà không thay đổi IP public. | Cấu hình phức tạp hơn nếu không quen với cú pháp Nginx. |


## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Apache HTTP Server | Phổ biến, nhưng Nginx thường nhanh hơn ở các kết nối đồng thời cao. |
| HAProxy | Chuyên dụng cho Load Balancing, cực kỳ mạnh mẽ nhưng cấu hình phức tạp hơn Nginx. |
| Traefik | Phù hợp tuyệt vời cho môi trường Docker/Kubernetes (Auto-discovery). |


## 9. How
Cấu hình Reverse Proxy cơ bản:

```nginx
server {
    listen 80;
    server_name myapp.com;

    location / {
        # Chuyển tiếp request tới backend chạy ở port 3000
        proxy_pass http://127.0.0.1:3000;

        # Giữ lại thông tin gốc của client
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Cân bằng tải (Load Balancing):
```nginx
upstream my_backend {
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}

server {
    location / {
        proxy_pass http://my_backend;
    }
}
```


## 10. Production concerns
### Scaling
Sử dụng chỉ thị `upstream` để dễ dàng thêm/bớt server backend mà không làm gián đoạn dịch vụ.

### Failure
Xử lý lỗi `502 Bad Gateway` (Backend sập) hoặc `504 Gateway Timeout` (Backend xử lý quá lâu). Cần cấu hình `proxy_read_timeout` hợp lý.

### Monitoring
Sử dụng `access_log` và `error_log` để theo dõi lưu lượng và phát hiện sự cố kịp thời.


## 11. Common mistakes
- Mistake: Quên không thiết lập `proxy_set_header Host $host`.
  Fix: Luôn thêm header này để backend biết domain nào đang được gọi (quan trọng khi chạy Multi-tenant).

- Mistake: Không giới hạn `client_max_body_size`.
  Fix: Mặc định Nginx chỉ cho upload 1MB. Cần tăng lên (ví dụ: `client_max_body_size 20M;`) nếu app có tính năng upload file.


## 12. Sample project
Thiết lập một hệ thống gồm 1 Nginx đứng trước 2 container Docker chạy Node.js, cấu hình SSL qua Let's Encrypt và bật Gzip compression.


## 13. Interview
### Core Q&A
1. Q: Reverse Proxy khác gì với Forward Proxy?
   A: Forward Proxy đại diện cho Client (ẩn danh client), Reverse Proxy đại diện cho Server (ẩn danh backend server).

2. Q: Làm sao để xử lý lỗi 502 Bad Gateway?
   A: Kiểm tra xem ứng dụng backend có đang chạy không, port có đúng không, và firewall có chặn kết nối giữa Nginx và backend không.

### Scenario
"Hệ thống của bạn có 3 backend, nhưng 1 cái cấu hình yếu hơn 2 cái còn lại. Bạn làm thế nào?"
-> Sử dụng tham số `weight` trong block `upstream` (ví dụ: `server backend1 weight=3; server backend2 weight=1;`).


## 14. References
- Official Docs: [https://nginx.org/en/docs/http/ngx_http_proxy_module.html](https://nginx.org/en/docs/http/ngx_http_proxy_module.html)
- Nginx Full Config Guide: [https://www.nginx.com/resources/wiki/start/topics/examples/full/](https://www.nginx.com/resources/wiki/start/topics/examples/full/)


## 15. Real-world Code
Hầu hết các hệ thống Kubernetes sử dụng Nginx Ingress Controller - thực chất là một bản nâng cao của Nginx Reverse Proxy.


## 16. Community
- Reddit: r/nginx
- Stack Overflow: Tag [nginx]
- Nginx Community Forum.
