---
created: 2026-05-06
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/security"
related:
  - "[[SSL for Nginx in Ubuntu.md]]"
  - "[[SSL and TLS.md]]"
  - "[[Nginx in Ubuntu.md]]"
---

## 1. What
Certbot là một công cụ phần mềm mã nguồn mở, miễn phí được thiết kế để tự động hóa việc sử dụng chứng chỉ Let's Encrypt trên các website được quản trị thủ công để bật HTTPS. Nó đóng vai trò là một ACME (Automated Certificate Management Environment) client giúp giao tiếp với máy chủ Let's Encrypt để cấp phát và gia hạn chứng chỉ.

## 2. Why
Trước khi có Certbot, việc lấy chứng chỉ SSL là một quy trình thủ công phức tạp: tạo CSR, gửi cho CA, xác thực qua email/file, rồi cài đặt vào web server. Khi chứng chỉ hết hạn, quy trình này phải lặp lại. Certbot ra đời để biến quy trình này thành hoàn toàn tự động, giúp website luôn duy trì trạng thái bảo mật mà không cần sự can thiệp của con người.

## 3. Mental Model
Hãy tưởng tượng Certbot giống như một **"Nhân viên hành chính tự động"**:
- Nhân viên này thay mặt bạn đến cơ quan cấp hộ chiếu (Let's Encrypt).
- Nhân viên tự điền đơn, tự chứng minh bạn là chủ ngôi nhà (Domain validation).
- Sau khi lấy được hộ chiếu (Certificate), nhân viên tự tay dán nó lên cửa nhà (Cấu hình Web Server).
- Quan trọng nhất, nhân viên này luôn theo dõi lịch và tự đi đổi hộ chiếu mới trước khi cái cũ hết hạn.

## 4. Where it fits
Vị trí trong hệ thống:
`Admin -> Certbot CLI -> Let's Encrypt CA -> Web Server (Nginx/Apache) Config`

## 5. When to use
- Khi bạn quản trị server Linux riêng (VPS, EC2) và cần SSL/TLS miễn phí.
- Khi muốn tự động hóa hoàn toàn việc gia hạn chứng chỉ 90 ngày của Let's Encrypt.
- Khi cần quản lý nhiều chứng chỉ cho nhiều domain khác nhau trên cùng một server.

## 6. When NOT to use
- Khi bạn sử dụng các nền tảng PaaS như Heroku, Vercel hoặc Managed Services như AWS CloudFront/ALB (họ có cơ chế SSL riêng).
- Khi bạn cần các loại chứng chỉ xác thực tổ chức (OV) hoặc xác thực mở rộng (EV) - Let's Encrypt không cấp các loại này.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Miễn phí và tự động hoàn toàn. | Cần cài đặt thêm package manager (như Snap) trên server. |
| Độ bảo mật cao, tuân thủ các tiêu chuẩn hiện đại. | Có giới hạn về số lượng chứng chỉ cấp phát trong một tuần (Rate limits). |
| Hỗ trợ hầu hết các Web Server phổ biến. | Yêu cầu quyền root để thực hiện các thay đổi cấu hình. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| acme.sh | Viết bằng Shell script, cực nhẹ, không yêu cầu dependencies phức tạp như Python. |
| LEGO | Viết bằng Go, thường dùng trong các môi trường Docker hoặc các công cụ viết bằng Go. |
| Caddy Server | Tích hợp sẵn ACME client bên trong, tự động lấy SSL mà không cần tool ngoài. |

## 9. How
Cách cài đặt Certbot theo khuyến nghị của EFF (sử dụng Snap):

```bash
# 1. Đảm bảo snapd đã được cập nhật
sudo snap install core; sudo snap refresh core

# 2. Cài đặt Certbot
sudo snap install --classic certbot

# 3. Tạo liên kết để sử dụng lệnh certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot

# 4. Lấy chứng chỉ cho Nginx (Certbot sẽ tự động cấu hình)
sudo certbot --nginx

# 5. Hoặc chỉ lấy chứng chỉ mà không sửa cấu hình web server
sudo certbot certonly --nginx
```

Kiểm tra lệnh gia hạn tự động:
`sudo certbot renew --dry-run`

## 10. Production concerns
### Auto-renewal
Certbot cài qua Snap sẽ tự động thiết lập một systemd timer. Bạn không cần thêm crontab thủ công. Kiểm tra bằng: `systemctl list-timers | grep certbot`.

### Rate Limits
Let's Encrypt giới hạn 50 chứng chỉ cho mỗi domain đăng ký mỗi tuần. Hãy cẩn thận khi thử nghiệm (nên dùng flag `--staging`).

### Hooks
Sử dụng `--deploy-hook` để thực hiện các hành động sau khi gia hạn thành công (ví dụ: copy chứng chỉ sang một thư mục khác hoặc restart một dịch vụ đặc thù).

## 11. Common mistakes
- Mistake: Không cài đặt `core` qua snap trước khi cài `certbot`.
  Fix: Luôn chạy `sudo snap install core` để đảm bảo runtime của snap ổn định.

- Mistake: Để Firewall chặn cổng 80 khi Certbot đang cố gắng xác thực.
  Fix: Let's Encrypt dùng cổng 80 để xác thực HTTP-01 challenge. Đảm bảo cổng này mở trong quá trình lấy/gia hạn chứng chỉ.

## 12. Sample project
Tối ưu hóa quy trình gia hạn:
Viết một script hook để sau khi Certbot gia hạn xong, nó sẽ tự động nén chứng chỉ cũ vào thư mục backup và gửi thông báo qua Telegram cho team DevOps.

## 13. Interview
### Core Q&A
1. Q: Làm thế nào Certbot biết khi nào cần gia hạn chứng chỉ?
   A: Mặc định, Certbot sẽ kiểm tra các chứng chỉ mỗi ngày 2 lần. Nó sẽ chỉ thực hiện gia hạn nếu chứng chỉ còn dưới 30 ngày là hết hạn.

2. Q: "Snap install core" có tác dụng gì trong quá trình cài đặt Certbot?
   A: Snap `core` cung cấp các thư viện và runtime cơ bản nhất cho các gói Snap khác. Việc cài đặt và refresh `core` đảm bảo Certbot (một ứng dụng Python phức tạp) có môi trường chạy ổn định nhất.

### Scenario
"Bạn chạy lệnh `certbot --nginx` nhưng nó báo lỗi không tìm thấy cấu hình server. Bạn xử lý thế nào?"
-> Trả lời:
1. Kiểm tra xem Nginx đã được cài đặt và đang chạy chưa.
2. Kiểm tra xem các file cấu hình trong `/etc/nginx/sites-enabled/` có chứa directive `server_name` khớp với domain bạn đang đăng ký không.
3. Nếu vẫn không được, dùng chế độ `certonly --webroot` để Certbot chỉ lấy file chứng chỉ, sau đó mình tự cấu hình Nginx bằng tay.

## 14. References
- Official Site: [certbot.eff.org](https://certbot.eff.org/)
- Documentation: [Certbot User Guide](https://eff-certbot.readthedocs.io/en/stable/)

## 15. Real-world Code
Lệnh liệt kê tất cả các chứng chỉ đang được quản lý bởi Certbot và ngày hết hạn:
`sudo certbot certificates`

## 16. Community
- EFF (Electronic Frontier Foundation)
- Let's Encrypt Community Forums
- Stack Overflow: Tag [certbot]
