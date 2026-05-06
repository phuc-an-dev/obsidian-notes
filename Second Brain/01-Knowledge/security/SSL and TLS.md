---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/others"
  - "#topic/security"
related:
  - "[[Nginx Configuration for Spring Boot in EC2.md]]"
  - "[[AWS Inbound Rules.md]]"
---

## 1. What
SSL (Secure Sockets Layer) và phiên bản hiện đại của nó là TLS (Transport Layer Security) là các giao thức bảo mật được thiết kế để thiết lập một liên kết mã hóa giữa máy chủ web (Server) và trình duyệt (Browser). Liên kết này đảm bảo rằng tất cả dữ liệu được truyền qua lại giữa chúng luôn ở trạng thái riêng tư và nguyên vẹn.

## 2. Why
Trước khi có SSL/TLS, dữ liệu được truyền qua internet dưới dạng văn bản thuần túy (Plaintext). Điều này cho phép kẻ tấn công dễ dàng thực hiện các hành vi:
- **Eavesdropping**: Đánh cắp thông tin nhạy cảm như mật khẩu, thẻ tín dụng.
- **Tampering**: Thay đổi dữ liệu trên đường truyền mà người dùng không hay biết.
- **Impersonation**: Giả mạo website để lừa đảo người dùng.
SSL/TLS ra đời để giải quyết 3 vấn đề cốt lõi: Encryption (Mã hóa), Data Integrity (Toàn vẹn dữ liệu) và Authentication (Xác thực).

## 3. Mental Model
Hãy tưởng tượng SSL/TLS giống như một **"Bao thư niêm phong có chữ ký xác thực"**:
- **Encryption**: Nội dung bức thư được viết bằng mật mã mà chỉ người gửi và người nhận mới hiểu. Nếu ai đó lấy trộm bao thư, họ cũng không đọc được gì.
- **Authentication**: Trên bao thư có con dấu của một cơ quan uy tín (Certificate Authority). Bạn nhìn vào con dấu để tin rằng bức thư thực sự đến từ đúng người bạn mong muốn.
- **Integrity**: Nếu bao thư bị rách hoặc có dấu hiệu bị cạy mở (dữ liệu bị sửa đổi), bạn sẽ biết ngay và từ chối nhận thư.

## 4. Where it fits
Vị trí trong mô hình OSI:
`Application Layer (HTTP/FTP) -> SSL/TLS Layer -> Transport Layer (TCP)`

Nó nằm giữa tầng ứng dụng và tầng giao vận, tạo ra một đường ống bảo mật cho các giao thức như HTTP (trở thành HTTPS), SMTP, IMAP.

## 5. When to use
- Mọi trang web hiện đại (bắt buộc để có SEO tốt và sự tin tưởng của người dùng).
- Các hệ thống API kết nối giữa các dịch vụ (Microservices).
- Các ứng dụng ngân hàng, thương mại điện tử, mạng xã hội.
- Giao tiếp giữa ứng dụng di động và máy chủ.

## 6. When NOT to use
- Hầu như không có trường hợp nào không nên dùng trong môi trường production ngày nay.
- Trong môi trường phát triển local (localhost) đôi khi có thể bỏ qua để đơn giản hóa, nhưng vẫn khuyến khích giả lập SSL để phát hiện sớm các lỗi về Mixed Content.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Bảo mật thông tin tuyệt đối cho người dùng. | Tốn thêm tài nguyên CPU để thực hiện quá trình mã hóa/giải mã. |
| Tăng thứ hạng SEO trên Google. | Làm tăng độ trễ (latency) nhẹ trong bước bắt tay (Handshake) ban đầu. |
| Ngăn chặn các cuộc tấn công Man-in-the-middle. | Cần quản lý việc gia hạn chứng chỉ (Certificate Renewal). |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| HTTP | Không bảo mật, dữ liệu bị truyền plaintext. |
| SSH Tunneling | Bảo mật tốt nhưng chủ yếu dùng cho quản trị hệ thống, không phù hợp cho người dùng web đại trà. |
| IPsec | Mã hóa ở tầng mạng (Layer 3), thường dùng cho VPN thay vì web. |

## 9. How
Quá trình bắt tay (TLS Handshake) diễn ra theo các bước:
1. **Client Hello**: Browser gửi phiên bản TLS hỗ trợ và danh sách các thuật toán mã hóa (Cipher suites).
2. **Server Hello**: Server chọn thuật toán và gửi lại Chứng chỉ số (Certificate) kèm Public Key.
3. **Authentication**: Browser xác thực chứng chỉ với các CA tin cậy.
4. **Key Exchange**: Browser tạo một Session Key (Symmetric Key), mã hóa nó bằng Public Key của server và gửi đi.
5. **Encryption**: Cả hai bên dùng Session Key này để mã hóa toàn bộ dữ liệu truyền tải sau đó.

## 10. Production concerns
### Certificate Renewal
Chứng chỉ thường có thời hạn 90 ngày (Let's Encrypt) hoặc 1-2 năm. Việc quên gia hạn sẽ khiến website bị trình duyệt chặn (lỗi "Your connection is not private").

### SSL Termination
Trong các hệ thống lớn, SSL thường được kết thúc tại Load Balancer hoặc Nginx (Reverse Proxy) để giảm tải cho các server ứng dụng phía sau.

### HSTS (HTTP Strict Transport Security)
Một header bảo mật yêu cầu trình duyệt luôn dùng HTTPS cho các lần truy cập sau, ngăn chặn tấn công hạ cấp giao thức (Protocol downgrade).

## 11. Common mistakes
- Mistake: Để chứng chỉ hết hạn trên Production.
  Fix: Sử dụng các công cụ tự động như Certbot để tự động gia hạn.

- Mistake: Lỗi Mixed Content (Trang HTTPS nhưng load ảnh/script từ HTTP).
  Fix: Luôn sử dụng đường dẫn tương đối hoặc đảm bảo tất cả tài nguyên đều dùng HTTPS.

## 12. Sample project
Sử dụng Certbot trên Ubuntu để cài đặt SSL miễn phí cho Nginx:
1. `sudo apt install certbot python3-certbot-nginx`
2. `sudo certbot --nginx -d yourdomain.com`
3. Certbot sẽ tự động sửa file cấu hình Nginx và thiết lập cronjob để tự động gia hạn.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa Symmetric và Asymmetric Encryption trong TLS là gì?
   A: Asymmetric (bất đối xứng) dùng cặp Public/Private key, chậm hơn, dùng để xác thực và trao đổi khóa ban đầu. Symmetric (đối xứng) dùng chung 1 khóa, nhanh hơn, dùng để mã hóa dữ liệu thực tế sau khi đã bắt tay thành công.

2. Q: SNI (Server Name Indication) là gì?
   A: Là một phần mở rộng của TLS cho phép server chạy nhiều website với các chứng chỉ SSL khác nhau trên cùng một địa chỉ IP.

### Scenario
"Khách hàng báo lỗi chứng chỉ không hợp lệ dù bạn vừa mua một chứng chỉ mới. Bạn kiểm tra gì?"
-> Trả lời: 
1. Kiểm tra xem đã cấu hình đúng "Intermediate Certificates" (Chain) chưa. 
2. Kiểm tra ngày giờ trên server/máy khách có bị sai lệch quá nhiều không.
3. Kiểm tra xem tên miền (Common Name) trong chứng chỉ có khớp chính xác với URL đang truy cập không.

## 14. References
- SSL Labs: [Qualys SSL Test](https://www.ssllabs.com/ssltest/)
- Let's Encrypt: [Documentation](https://letsencrypt.org/docs/)
- RFC 8446: [TLS 1.3 Specification](https://tools.ietf.org/html/rfc8446)

## 15. Real-world Code
Cấu hình SSL chuẩn cho Nginx:
```nginx
server {
    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/domain/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/domain/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
}
```

## 16. Community
- Reddit: r/security
- Stack Overflow: Tag [ssl] [tls]
- Blog: Cloudflare Blog (rất nhiều bài viết hay về TLS).
