---
created: 2026-05-06
tags:
  - "#type/tutorial"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/security"
related:
  - "[[SSL and TLS.md]]"
  - "[[Nginx in Ubuntu.md]]"
  - "[[Certbot.md]]"
---

## 1. What
SSL for Nginx in Ubuntu là quy trình đăng ký, cài đặt và cấu hình chứng chỉ SSL/TLS cho máy chủ web Nginx trên hệ điều hành Ubuntu. Quy trình này thường bao gồm việc sử dụng Let's Encrypt (miễn phí) thông qua công cụ Certbot hoặc cài đặt thủ công các chứng chỉ thương mại (Paid SSL).

## 2. Why
Việc chạy Nginx chỉ với HTTP (Cổng 80) khiến dữ liệu người dùng dễ bị đánh cắp. Đăng ký SSL giúp:
- Nâng cấp website lên HTTPS (Cổng 443) bảo mật.
- Tránh cảnh báo "Not Secure" từ trình duyệt Chrome/Safari.
- Đáp ứng tiêu chuẩn bắt buộc của các trình duyệt hiện đại và API của bên thứ ba (như Facebook Login, Google Maps).
- Cải thiện thứ hạng SEO.

## 3. Mental Model
Hãy tưởng tượng việc đăng ký SSL giống như việc **"Làm hộ chiếu cho website"**:
- **Let's Encrypt/CA**: Là cơ quan cấp hộ chiếu (Cục Quản lý Xuất nhập cảnh).
- **CSR (Certificate Signing Request)**: Là tờ khai thông tin cá nhân bạn gửi đi.
- **SSL Certificate**: Là cuốn hộ chiếu bạn nhận được.
- **Nginx Configuration**: Là việc bạn trình cuốn hộ chiếu đó ra ở cửa khẩu (Cổng 443) để chứng minh mình là người hợp pháp và an toàn.

## 4. Where it fits
Vị trí trong luồng vận hành:
`Domain DNS (A Record) -> Ubuntu Server -> Certbot (Challenge) -> Let's Encrypt -> SSL Certificate -> Nginx Config`

## 5. When to use
- Ngay sau khi bạn đã cấu hình xong Nginx và trỏ tên miền (Domain) về địa chỉ IP của server.
- Khi chứng chỉ cũ sắp hết hạn.
- Khi cần chuyển đổi từ HTTP sang HTTPS cho ứng dụng Spring Boot/Node.js chạy sau Nginx.

## 6. When NOT to use
- Khi bạn sử dụng Cloudflare Proxy (họ có thể cung cấp SSL miễn phí giữa Browser và Cloudflare). Tuy nhiên, vẫn nên dùng SSL giữa Cloudflare và Server (Origin SSL).
- Trong môi trường local test nội bộ không yêu cầu HTTPS.

## 7. Trade-offs
| Let's Encrypt (Certbot) | Commercial SSL (Paid) |
|-------------------------|-----------------------|
| Miễn phí 100%. | Tốn phí (từ vài chục đến hàng trăm USD). |
| Thời hạn ngắn (90 ngày) nhưng tự động gia hạn. | Thời hạn dài (1-2 năm). |
| Chỉ xác thực tên miền (DV). | Có thể xác thực doanh nghiệp (OV/EV) với độ tin cậy cao hơn. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Cloudflare SSL | Cực kỳ nhanh, không cần cài đặt trên server, nhưng không bảo mật hoàn toàn luồng từ Cloudflare về Server nếu không cấu hình thêm. |
| AWS Certificate Manager (ACM) | Miễn phí và tự động hoàn toàn nếu bạn dùng Load Balancer (ALB) hoặc CloudFront. |

## 9. How
Quy trình cài đặt SSL Let's Encrypt với Certbot:

```bash
# 1. Cài đặt Certbot và plugin Nginx
sudo apt update
sudo apt install certbot python3-certbot-nginx -y

# 2. Chạy lệnh đăng ký SSL (thay domain của bạn)
# Certbot sẽ tự tìm cấu hình Nginx và chỉnh sửa cho bạn
sudo certbot --nginx -d example.com -d www.example.com

# 3. Làm theo hướng dẫn trên màn hình (nhập email, đồng ý điều khoản)
# Chọn "Redirect" để tự động chuyển toàn bộ HTTP sang HTTPS.

# 4. Kiểm tra gia hạn tự động (Dry run)
sudo certbot renew --dry-run
```

Nếu bạn có file chứng chỉ thương mại (`.crt` và `.key`), cấu hình thủ công trong Nginx:
```nginx
server {
    listen 443 ssl;
    server_name example.com;

    ssl_certificate /path/to/your_certificate.crt;
    ssl_certificate_key /path/to/your_private.key;

    # Cấu hình bảo mật thêm
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
}
```

## 10. Production concerns
### Auto-renewal
Lệnh `certbot` trên Ubuntu sẽ tự động tạo một systemd timer hoặc cronjob để kiểm tra gia hạn mỗi 12 giờ. Kiểm tra trạng thái bằng: `systemctl list-timers`.

### Firewall
Đảm bảo đã mở cổng 443 trên Firewall của Ubuntu (UFW) và Security Group của AWS:
`sudo ufw allow 'Nginx Full'`

### Chain of Trust
Khi cài đặt thủ công, luôn sử dụng file `fullchain` (chứa cả certificate của bạn và intermediate certificate) để tránh lỗi trên một số thiết bị di động.

## 11. Common mistakes
- Mistake: Quên trỏ tên miền về IP server trước khi chạy Certbot.
  Fix: Luôn đảm bảo lệnh `ping yourdomain.com` trỏ đúng về IP của server đang cài.

- Mistake: Chặn cổng 80 trên firewall.
  Fix: Let's Encrypt cần cổng 80 để thực hiện "HTTP Challenge" xác thực quyền sở hữu tên miền.

## 12. Sample project
Thiết lập hoàn chỉnh:
1. Cài đặt Nginx trên Ubuntu.
2. Trỏ domain `dev.yourname.com` về IP.
3. Chạy Certbot để lấy SSL.
4. Cấu hình crontab để gửi mail thông báo mỗi khi Nginx được reload sau khi gia hạn SSL.

## 13. Interview
### Core Q&A
1. Q: Làm thế nào để Certbot xác thực bạn là chủ sở hữu tên miền?
   A: Certbot tạo một file tạm trong thư mục `.well-known/acme-challenge/` và Let's Encrypt server sẽ truy cập vào URL đó qua cổng 80 để kiểm tra.

2. Q: Tại sao chứng chỉ Let's Encrypt chỉ có thời hạn 90 ngày?
   A: Để tăng tính bảo mật (nếu khóa bị lộ, thời gian thiệt hại sẽ ngắn hơn) và thúc đẩy việc tự động hóa hoàn toàn quy trình quản lý chứng chỉ.

### Scenario
"Lệnh tự động gia hạn SSL thất bại, bạn sẽ kiểm tra những gì?"
-> Trả lời:
1. Kiểm tra xem cổng 80 có đang bị chặn bởi Firewall hay một ứng dụng khác không.
2. Kiểm tra xem cấu hình Nginx có bị lỗi cú pháp không (`nginx -t`).
3. Kiểm tra log của certbot tại `/var/log/letsencrypt/letsencrypt.log`.
4. Kiểm tra xem tên miền có còn trỏ đúng về IP của server không.

## 14. References
- Certbot Instructions: [certbot.eff.org](https://certbot.eff.org/instructions?os=ubuntufocal&short=nginx)
- Let's Encrypt Docs: [letsencrypt.org/docs](https://letsencrypt.org/docs/)

## 15. Real-world Code
Sử dụng script để kiểm tra ngày hết hạn của SSL từ dòng lệnh:
`echo | openssl s_client -connect google.com:443 2>/dev/null | openssl x509 -noout -dates`

## 16. Community
- Let's Encrypt Community Support: [community.letsencrypt.org](https://community.letsencrypt.org/)
- Reddit: r/sysadmin
- Stack Overflow: Tag [ssl] [certbot]
