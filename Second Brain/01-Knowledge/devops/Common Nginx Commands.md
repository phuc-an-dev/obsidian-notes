---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/http"
related:
  - "[[Nginx in Ubuntu.md]]"
  - "[[Nginx Configuration for Spring Boot in EC2.md]]"
---

## 1. What
Common Nginx Commands là danh sách các lệnh CLI (Command Line Interface) thiết yếu nhất để quản lý, vận hành và xử lý sự cố với Nginx Web Server. Các lệnh này bao quát từ việc điều khiển dịch vụ hệ thống đến việc kiểm tra tính đúng đắn của cấu hình và theo dõi luồng dữ liệu.

## 2. Why
Nginx là một thành phần quan trọng trong hạ tầng, thường đóng vai trò là điểm tiếp nhận đầu tiên của traffic. Việc nắm vững các lệnh điều khiển giúp quản trị viên phản ứng nhanh với các sự cố, thay đổi cấu hình mà không làm gián đoạn dịch vụ (Zero-downtime) và hiểu rõ trạng thái hoạt động của server.

## 3. Mental Model
Hãy tưởng tượng Nginx CLI giống như **"Bảng điều khiển của một trạm biến áp"**.
- Bạn không cần phải tháo tung máy móc ra để kiểm tra.
- Các lệnh giúp bạn: Bật/Tắt nguồn (start/stop), Kiểm tra cầu chì xem có bị cháy không (config test), và Theo dõi đồng hồ đo điện năng (logs) để biết dòng điện đang chạy như thế nào.

## 4. Where it fits
Vị trí trong quy trình vận hành:
`Sysadmin/DevOps -> SSH -> Terminal -> Nginx Commands -> Nginx Master/Worker Processes`

## 5. When to use
- Khi vừa chỉnh sửa file cấu hình và cần áp dụng thay đổi.
- Khi cần kiểm tra xem Nginx có đang gặp lỗi cú pháp hay không.
- Khi cần dừng dịch vụ để bảo trì hoặc khởi động lại sau khi cập nhật hệ thống.
- Khi cần theo dõi traffic hoặc debug lỗi 4xx/5xx thông qua logs.

## 6. When NOT to use
- Khi bạn đang sử dụng các container orchestration như Kubernetes (nên dùng `kubectl logs` hoặc quản lý qua Deployment/Service).
- Khi bạn sử dụng các bảng điều khiển giao diện web (như Nginx Proxy Manager) trừ khi cần can thiệp sâu vào hệ thống bên dưới.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Thao tác trực tiếp, phản hồi ngay lập tức. | Đòi hỏi quyền sudo/root cho hầu hết các lệnh. |
| Hỗ trợ reload cấu hình không làm ngắt kết nối khách hàng. | Một lệnh restart sai thời điểm có thể gây downtime cho toàn bộ hệ thống. |
| Cung cấp thông tin debug chi tiết. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `systemctl` | Lệnh chuẩn của hệ thống Linux để quản lý service (khuyên dùng trên Ubuntu). |
| `nginx` binary | Gọi trực tiếp file thực thi của Nginx, hữu ích cho việc kiểm tra cấu hình (`-t`). |

## 9. How
Các nhóm lệnh quan trọng nhất:

### Quản lý dịch vụ (Dùng systemctl)
```bash
sudo systemctl start nginx    # Khởi động Nginx
sudo systemctl stop nginx     # Dừng Nginx ngay lập tức
sudo systemctl restart nginx  # Khởi động lại (ngắt kết nối cũ)
sudo systemctl reload nginx   # Tải lại cấu hình (không ngắt kết nối)
sudo systemctl status nginx   # Xem trạng thái và vài dòng log cuối
```

### Kiểm tra cấu hình
```bash
sudo nginx -t                 # Kiểm tra cú pháp file cấu hình (Cực kỳ quan trọng)
sudo nginx -T                 # Kiểm tra và in toàn bộ cấu hình ra màn hình
```

### Theo dõi Logs (Debug)
```bash
# Xem log truy cập theo thời gian thực
sudo tail -f /var/log/nginx/access.log

# Xem log lỗi theo thời gian thực
sudo tail -f /var/log/nginx/error.log

# Xem log của Nginx thông qua hệ thống journal
sudo journalctl -u nginx -f
```

### Thông tin phiên bản
```bash
nginx -v                      # Chỉ xem phiên bản
nginx -V                      # Xem phiên bản và các module đã được biên dịch kèm
```

## 10. Production concerns
### Zero-downtime
Luôn ưu tiên `reload` thay vì `restart` trên Production để đảm bảo khách hàng đang kết nối không bị ngắt quãng.

### Config Testing
**Bắt buộc** chạy `nginx -t` trước khi thực hiện bất kỳ lệnh `reload` hay `restart` nào. Nếu file cấu hình lỗi, Nginx sẽ không thể khởi động lại, dẫn đến downtime.

### Log Rotation
Nginx logs có thể phình to rất nhanh. Ubuntu mặc định cài sẵn `logrotate` cho Nginx, nhưng bạn nên kiểm tra file `/etc/logrotate.d/nginx` để đảm bảo chính sách lưu trữ phù hợp.

## 11. Common mistakes
- Mistake: Sửa file `.conf` xong rồi `restart` ngay mà không check.
  Fix: Luôn dùng `sudo nginx -t && sudo systemctl reload nginx`.

- Mistake: Gõ lệnh `nginx -s reload` thay vì `systemctl reload nginx` trên hệ thống dùng systemd.
  Fix: Nên dùng `systemctl` để đồng bộ với quản lý dịch vụ của hệ điều hành.

## 12. Sample project
Kịch bản bảo trì Nginx:
1. Chỉnh sửa giới hạn upload `client_max_body_size`.
2. Chạy `sudo nginx -t` thấy báo `syntax is ok`.
3. Chạy `sudo systemctl reload nginx`.
4. Mở `tail -f /var/log/nginx/access.log` và thực hiện upload thử để kiểm tra kết quả.

## 13. Interview
### Core Q&A
1. Q: Lệnh nào dùng để kiểm tra cấu hình Nginx mà không cần restart?
   A: Dùng lệnh `sudo nginx -t`.

2. Q: Làm thế nào để xem các module đã được cài đặt kèm với Nginx?
   A: Dùng lệnh `nginx -V` (V viết hoa).

### Scenario
"Nginx của bạn không khởi động được sau khi chỉnh sửa cấu hình, nhưng `nginx -t` báo mọi thứ đều ổn. Bạn làm gì?"
-> Trả lời:
1. Kiểm tra xem có dịch vụ nào khác (như Apache) đang chiếm port 80/443 không bằng lệnh `sudo netstat -tuln` hoặc `lsof -i :80`.
2. Kiểm tra `dmesg` hoặc `sudo journalctl -u nginx` để xem lỗi ở tầng hệ thống (ví dụ: thiếu quyền ghi file log, hết dung lượng đĩa).

## 14. References
- Nginx Command Line: [nginx.org/en/docs/switches.html](https://nginx.org/en/docs/switches.html)
- DigitalOcean Nginx Guide: [Nginx Essentials](https://www.digitalocean.com/community/cheatsheets/nginx-cheat-sheet-common-commands-and-configuration-setups)

## 15. Real-world Code
Nhiều kỹ sư DevOps tạo script `deploy.sh` tự động chạy `nginx -t` trước khi thực hiện copy file cấu hình mới vào server.

## 16. Community
- Reddit: r/nginx
- Stack Overflow: Tag [nginx]
- Nginx Official Blog.
