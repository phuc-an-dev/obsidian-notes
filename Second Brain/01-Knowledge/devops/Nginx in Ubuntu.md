---
created: 2026-05-06
tags:
  - "#type/tutorial"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/http"
related:
  - "[[nginx-reverse-proxy.md]]"
  - "[[Common Ubuntu Commands.md]]"
  - "[[Nginx Configuration for Spring Boot in EC2.md]]"
  - "[[Common Nginx Commands.md]]"
  - "[[SSL for Nginx in Ubuntu.md]]"
  - "[[Certbot.md]]"
---

## 1. What
Nginx in Ubuntu là quá trình cài đặt và thiết lập cơ bản Nginx (phát âm là "engine-x") - một web server mã nguồn mở mạnh mẽ, đồng thời là một reverse proxy và load balancer, trên hệ điều hành Ubuntu Linux.

## 2. Why
Trước khi có Nginx, Apache là lựa chọn số một nhưng gặp vấn đề về hiệu năng khi xử lý hàng ngàn kết nối cùng lúc (C10k problem). Nginx ra đời với kiến trúc hướng sự kiện (event-driven) giúp xử lý lượng truy cập cực lớn với tài nguyên RAM và CPU cực thấp. Trên Ubuntu, Nginx là lựa chọn hàng đầu để phục vụ file tĩnh và làm cổng vào cho các ứng dụng Backend.

## 3. Mental Model
Hãy tưởng tượng Nginx giống như một **"Nhân viên tiếp tân"** cực kỳ nhanh nhẹn ở sảnh một tòa nhà:
- Khi khách (User) đến, nhân viên này tiếp nhận yêu cầu ngay lập tức.
- Nếu khách chỉ hỏi xin tài liệu (File tĩnh), nhân viên tự tay đưa luôn.
- Nếu khách muốn gặp chuyên gia (Backend app), nhân viên sẽ dẫn khách đến đúng phòng (Reverse proxy).
- Nhân viên này có thể tiếp hàng trăm khách cùng lúc mà không hề bối rối hay làm chậm quy trình.

## 4. Where it fits
Vị trí trong hệ thống:
`User -> Internet -> Nginx (Port 80/443) -> Application Server (Node.js/Spring Boot) -> Database`

## 5. When to use
- Khi cần chạy một website tĩnh (HTML/CSS/JS).
- Khi cần một Reverse Proxy để bảo vệ và điều hướng traffic cho ứng dụng Backend.
- Khi cần cấu hình SSL/TLS (HTTPS) tập trung tại một nơi.
- Khi cần Load Balancing để chia tải cho nhiều server phía sau.

## 6. When NOT to use
- Khi ứng dụng của bạn cực kỳ đơn giản và đã được triển khai trên các nền tảng tự động hoàn toàn như Vercel hoặc Netlify (nơi họ đã lo sẵn tầng web server).
- Khi bạn cần các tính năng đặc thù mà chỉ Apache hỗ trợ (như `.htaccess` trong từng thư mục - mặc dù hiện nay rất hiếm).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Hiệu năng cực cao với ít tài nguyên. | Cấu hình có phần khắt khe hơn Apache (không hỗ trợ .htaccess). |
| Cộng đồng hỗ trợ và tài liệu cực kỳ phong phú. | Việc thêm module đôi khi yêu cầu phải biên dịch lại từ mã nguồn (mặc dù bản Ubuntu đã có sẵn các module phổ biến). |
| Hỗ trợ xử lý file tĩnh và làm Reverse proxy xuất sắc. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Apache | Lâu đời, linh hoạt với .htaccess nhưng tốn tài nguyên hơn khi tải cao. |
| Caddy | Tự động cấu hình SSL (HTTPS) cực nhanh, viết bằng Go, dễ cấu hình hơn Nginx. |
| HAProxy | Chuyên dụng cho Load Balancing ở tầng 4 và tầng 7 với hiệu năng cực cao. |

## 9. How
Quy trình cài đặt cơ bản trên Ubuntu:

```bash
# 1. Cập nhật danh sách gói
sudo apt update

# 2. Cài đặt Nginx
sudo apt install nginx -y

# 3. Kiểm tra trạng thái
sudo systemctl status nginx

# 4. Cấu hình Firewall (UFW) cho phép HTTP/HTTPS
sudo ufw allow 'Nginx Full'

# 5. Các lệnh quản lý cơ bản
sudo systemctl stop nginx     # Dừng
sudo systemctl start nginx    # Chạy
sudo systemctl restart nginx  # Khởi động lại
sudo systemctl reload nginx   # Tải lại cấu hình (không ngắt kết nối)
```

Kiểm tra kết nối bằng cách truy cập `http://your_server_ip`. Bạn sẽ thấy trang "Welcome to nginx!".

## 10. Production concerns
### Security
Luôn ẩn phiên bản Nginx trong response header để tránh hacker biết thông tin:
`server_tokens off;` trong file `/etc/nginx/nginx.conf`.

### Virtual Hosts
Sử dụng `Server Blocks` để chạy nhiều website trên cùng một server. File cấu hình nên đặt tại `/etc/nginx/sites-available/` và tạo link sang `/etc/nginx/sites-enabled/`.

### Logging
Log mặc định nằm tại `/var/log/nginx/access.log` và `error.log`. Hãy dùng `logrotate` (có sẵn trên Ubuntu) để tránh file log quá lớn làm đầy ổ cứng.

## 11. Common mistakes
- Mistake: Sửa file cấu hình xong không kiểm tra lỗi mà restart ngay.
  Fix: Luôn chạy `sudo nginx -t` để kiểm tra cú pháp trước khi reload/restart.

- Mistake: Quên phân quyền (Permissions) cho thư mục chứa code website.
  Fix: Đảm bảo user `www-data` có quyền đọc các file trong thư mục web của bạn.

## 12. Sample project
Thiết lập một con Ubuntu server:
1. Cài đặt Nginx.
2. Tạo một file `index.html` đơn giản tại `/var/www/my_site/index.html`.
3. Tạo một Server Block mới để nhận diện domain `my-site.local`.
4. Trỏ domain đó về server và kiểm tra kết quả.

## 13. Interview
### Core Q&A
1. Q: Tại sao Nginx lại nhanh hơn Apache trong việc xử lý nhiều kết nối?
   A: Apache tạo một tiến trình (process) hoặc luồng (thread) mới cho mỗi kết nối, gây tốn RAM khi số lượng khách tăng. Nginx dùng kiến trúc hướng sự kiện (event-driven), một tiến trình có thể xử lý hàng ngàn kết nối cùng lúc mà không gây quá tải.

2. Q: Sự khác biệt giữa `reload` và `restart` trong Nginx là gì?
   A: `restart` sẽ tắt hẳn và bật lại dịch vụ, làm ngắt các kết nối hiện tại. `reload` sẽ đọc lại cấu hình và áp dụng cho các tiến trình mới mà không ngắt các kết nối đang xử lý.

### Scenario
"Website của bạn báo lỗi 502 Bad Gateway sau khi bạn thêm Nginx làm Reverse Proxy. Bạn sẽ làm gì?"
-> Trả lời:
1. Kiểm tra log lỗi: `sudo tail -f /var/log/nginx/error.log`.
2. Kiểm tra xem ứng dụng Backend (Node.js/Java) có đang chạy không.
3. Kiểm tra xem port và địa chỉ IP trong cấu hình `proxy_pass` của Nginx có khớp với Backend không.

## 14. References
- Official Documentation: [nginx.org/en/docs](https://nginx.org/en/docs/)
- Ubuntu Community Wiki: [Nginx](https://help.ubuntu.com/community/Nginx)

## 15. Real-world Code
Cấu hình mẫu cho một trang web tĩnh:
```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/example.com;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

## 16. Community
- Nginx Forum: [forum.nginx.org](https://forum.nginx.org/)
- Reddit: r/nginx
- Stack Overflow: Tag [nginx]
