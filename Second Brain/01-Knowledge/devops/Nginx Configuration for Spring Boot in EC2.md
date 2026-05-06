---
created: 2026-05-06
tags:
  - "#type/tutorial"
  - "#status/draft"
  - "#lang/spring"
  - "#topic/http"
related:
  - "[[Nginx in Ubuntu.md]]"
  - "[[aws-ec2-instance.md]]"
  - "[[mvn clean package in Spring Boot.md]]"
---

## 1. What
Nginx Configuration for Spring Boot in EC2 là quá trình thiết lập Nginx đóng vai trò là một Reverse Proxy nằm phía trước ứng dụng Spring Boot trên một máy ảo AWS EC2. Nginx sẽ tiếp nhận các yêu cầu từ internet (thường ở cổng 80 hoặc 443) và chuyển tiếp (forward) chúng đến ứng dụng Spring Boot đang chạy ngầm (thường ở cổng 8080).

## 2. Why
Mặc dù Spring Boot có sẵn embedded server (Tomcat/Netty), việc sử dụng Nginx làm lớp đệm mang lại nhiều lợi ích:
- **Security**: Che giấu cổng 8080 của ứng dụng, chỉ mở cổng 80/443 cho thế giới bên ngoài.
- **SSL Termination**: Xử lý chứng chỉ HTTPS tại Nginx, giúp giảm tải cho ứng dụng Spring Boot.
- **Static Content**: Nginx xử lý file tĩnh (ảnh, CSS, JS) nhanh hơn Tomcat rất nhiều.
- **Header Management**: Dễ dàng thêm hoặc sửa đổi các HTTP headers cho mục đích bảo mật hoặc định danh.

## 3. Mental Model
Hãy tưởng tượng Nginx là một **"Cánh cửa bảo vệ"** và Spring Boot là **"Phòng làm việc bên trong"**.
- Khách hàng không thể đi thẳng vào phòng làm việc.
- Họ phải đi qua cửa bảo vệ (Nginx).
- Bảo vệ sẽ kiểm tra thẻ (SSL), ghi chép thông tin khách (Headers) rồi mới dẫn khách vào phòng (Proxy pass).
- Nếu khách chỉ muốn lấy tờ rơi (File tĩnh), bảo vệ sẽ đưa luôn ở cửa mà không cần làm phiền người bên trong.

## 4. Where it fits
Luồng dữ liệu:
`User -> Internet -> EC2 (Port 80/443) -> Nginx -> Localhost (Port 8080) -> Spring Boot App`

## 5. When to use
- Khi triển khai ứng dụng Spring Boot thực tế trên AWS EC2.
- Khi cần cấu hình domain name và HTTPS cho ứng dụng.
- Khi một máy ảo EC2 chạy nhiều ứng dụng Spring Boot khác nhau (Virtual Hosting).

## 6. When NOT to use
- Khi sử dụng các dịch vụ Managed như AWS App Runner hoặc AWS Fargate kết hợp với Application Load Balancer (ALB) - nơi ALB đã đảm nhận vai trò của Nginx.
- Môi trường phát triển local đơn giản.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Tăng cường bảo mật và hiệu năng. | Thêm một thành phần cần quản lý và cấu hình. |
| Dễ dàng scale-up bằng cách thêm nhiều instance phía sau Nginx. | Cần cấu hình đúng để tránh mất thông tin IP gốc của khách hàng. |
| Hỗ trợ nén dữ liệu (Gzip) tốt. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| AWS ALB | Managed service của AWS, mạnh mẽ hơn nhưng tốn phí hơn Nginx cài trực tiếp. |
| Apache | Chậm hơn và tốn tài nguyên hơn Nginx trong kịch bản reverse proxy. |
| Spring Cloud Gateway | Phù hợp để làm API Gateway trong microservices, nhưng vẫn thường đứng sau Nginx. |

## 9. How
Cấu hình Server Block cơ bản tại `/etc/nginx/sites-available/spring-app`:

```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Tăng timeout cho các request lâu (như export báo cáo)
        proxy_connect_timeout 90s;
        proxy_read_timeout 90s;
    }

    # Phục vụ file tĩnh (nếu có)
    location /static/ {
        root /var/www/spring-app;
    }
}
```

Sau đó tạo link sang `sites-enabled` và restart Nginx.

## 10. Production concerns
### X-Forwarded Headers
Spring Boot cần biết nó đang chạy sau một proxy để tạo ra các URL đúng (đặc biệt khi dùng OAuth2 hoặc Redirect). Trong `application.properties`, cần thêm:
`server.forward-headers-strategy=native`

### Client Max Body Size
Nếu ứng dụng có tính năng upload file lớn, phải tăng giới hạn của Nginx (mặc định 1MB):
`client_max_body_size 10M;`

### Gzip Compression
Bật Gzip để giảm dung lượng dữ liệu truyền tải:
`gzip on;`
`gzip_types text/plain application/json;`

## 11. Common mistakes
- Mistake: Không chuyển tiếp header `X-Forwarded-Proto`.
  Fix: Luôn thêm header này để Spring Boot biết request gốc là HTTP hay HTTPS.

- Mistake: Quên mở cổng 80/443 trong AWS Security Group cho EC2.
  Fix: Kiểm tra Inbound Rules của Security Group trên AWS Console.

## 12. Sample project
Thiết lập một hệ thống:
1. Build file JAR của Spring Boot và chạy bằng Systemd service trên EC2.
2. Cài đặt Nginx và cấu hình Reverse Proxy trỏ vào cổng 8080.
3. Sử dụng Let's Encrypt (Certbot) để cấu hình HTTPS tự động cho Nginx.

## 13. Interview
### Core Q&A
1. Q: Tại sao cần `proxy_set_header Host $host;`?
   A: Để Spring Boot nhận biết được domain mà khách hàng đang truy cập, giúp ích cho việc xử lý đa tên miền hoặc redirect đúng địa chỉ.

2. Q: Làm thế nào để Nginx phục vụ ảnh trực tiếp mà không gọi vào Spring Boot?
   A: Sử dụng một `location` block trỏ vào thư mục chứa ảnh trên đĩa cứng và dùng lệnh `root` hoặc `alias`.

### Scenario
"Người dùng phàn nàn rằng họ bị log out ngay lập tức sau khi đăng nhập qua HTTPS. Lỗi do đâu?"
-> Trả lời: Có thể do Nginx chưa forward header `X-Forwarded-Proto`, dẫn đến Spring Boot nghĩ request là HTTP và tạo ra Cookie không có thuộc tính `Secure`, hoặc redirect khách hàng về trang login bản HTTP.

## 14. References
- Spring Boot Documentation: [Running behind a front-end proxy server](https://docs.spring.io/spring-boot/docs/current/reference/html/howto.html#howto.webserver.use-behind-a-proxy-server)
- Nginx Blog: [NGINX as a Reverse Proxy for Spring Boot](https://www.nginx.com/blog/nginx-reverse-proxy-spring-boot/)

## 15. Real-world Code
Nghiên cứu các cấu hình Nginx trong các dự án JHipster hoặc các Docker Compose stack dành cho Spring Boot.

## 16. Community
- Reddit: r/SpringBoot
- Stack Overflow: Tag [spring-boot] [nginx]
- Spring Blog.
