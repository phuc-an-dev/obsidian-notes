---
created: 2026-05-04
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/network"
related:
  - "[[ec2]]"
  - "[[chmod 400]]"
  - "[[ssh-connection-with-key]]"
---

## 1. What
**SSH (Secure Shell)** là một giao thức mạng dùng để thiết lập kết nối quản trị từ xa một cách an toàn giữa máy tính cá nhân (client) và máy chủ (server). Nó mã hóa toàn bộ dữ liệu truyền đi, bao gồm cả mật khẩu và các lệnh thực thi, để ngăn chặn việc bị đánh cắp thông tin trên đường truyền.

## 2. Why
Trước khi có SSH, người ta dùng các giao thức như Telnet hoặc Rlogin. Tuy nhiên, các giao thức này gửi dữ liệu dưới dạng văn bản thuần túy (plain text), nghĩa là bất kỳ ai nằm giữa kết nối đều có thể đọc được tên đăng nhập và mật khẩu của bạn. SSH ra đời như một tiêu chuẩn vàng để thay thế, mang lại sự bảo mật tuyệt đối cho việc quản lý server qua internet.

## 3. Mental Model
Hãy tưởng tượng SSH như một **"Đường ống bọc thép xuyên không"**. Khi bạn muốn vào một căn nhà (server) ở rất xa, bạn chui vào một cái ống thép cực kỳ chắc chắn. Mọi thứ bạn nói hoặc làm bên trong ống (dữ liệu) đều không ai bên ngoài nghe thấy được. Thậm chí, để vào được ống này, bạn cần có một chiếc **"Chìa khóa đặc biệt (SSH Key)"** hoặc **"Mật khẩu bảo mật"** mà chỉ bạn và chủ nhà biết.

## 4. Where it fits
SSH nằm ở tầng Application trong mô hình OSI, sử dụng giao thức TCP tại cổng mặc định là 22:
`Client Device -> SSH Protocol (Port 22) -> Encrypted Tunnel -> Remote Server Shell`

## 5. When to use
- Khi cần cấu hình, cài đặt phần mềm trên các server đám mây (AWS, Azure, Google Cloud).
- Khi cần truyền tệp tin một cách an toàn qua giao thức SFTP hoặc SCP.
- Khi cần tạo đường hầm (tunneling) để truy cập các dịch vụ nội bộ từ xa.

## 6. When NOT to use
- Khi bạn cần giao diện đồ họa (GUI) phức tạp (mặc dù SSH hỗ trợ X11 forwarding nhưng hiệu năng không tốt, nên dùng RDP hoặc VNC).
- Khi thiết bị đầu cuối không hỗ trợ giao thức SSH (các thiết bị IoT cực kỳ đơn giản).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Mã hóa cực kỳ mạnh mẽ, an toàn tuyệt đối. | Có thể bị tấn công Brute-force nếu dùng mật khẩu yếu. |
| Hỗ trợ xác thực bằng cặp Key (Public/Private) cực kỳ bảo mật. | Cổng 22 mặc định thường là mục tiêu tấn công hàng đầu. |
| Tiết kiệm băng thông vì chỉ truyền text. | Cần kiến thức về dòng lệnh (CLI) để sử dụng hiệu quả. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| **Telnet** | Không mã hóa, cực kỳ kém an toàn, hiện nay gần như không dùng. |
| **RDP (Remote Desktop)** | Chuyên cho Windows, có giao diện đồ họa, tốn băng thông hơn. |
| **AWS Systems Manager (SSM)** | Truy cập server qua trình duyệt/CLI của AWS mà không cần mở cổng 22. |

## 9. How
```bash
# Kết nối cơ bản bằng mật khẩu
ssh user@1.2.3.4

# Kết nối bằng SSH Key
ssh -i path/to/key.pem user@1.2.3.4

# Chạy lệnh từ xa mà không cần vào shell
ssh user@1.2.3.4 "ls -l /var/www"
```

## 10. Production concerns
### Scaling
Sử dụng các công cụ như `Ansible` hoặc `Terraform` để quản lý hàng loạt SSH keys trên hàng ngàn server cùng lúc thay vì cấu hình thủ công.

### Failure
Nếu bị mất Private Key, bạn sẽ hoàn toàn mất quyền truy cập vào server nếu không cấu hình các phương thức fallback (như Console access). Luôn có bản backup key an toàn.

### Monitoring
Theo dõi tệp `/var/log/auth.log` (trên Ubuntu) để phát hiện các nỗ lực đăng nhập trái phép và chặn IP bằng `Fail2Ban`.

## 11. Common mistakes
- **Mistake**: Để lộ Private Key cho người khác hoặc commit lên GitHub.
  **Fix**: Luôn giữ Private Key bí mật và dùng `.gitignore`.

- **Mistake**: Sử dụng mật khẩu thay vì SSH Key cho tài khoản root.
  **Fix**: Luôn dùng SSH Key và vô hiệu hóa Password Authentication trong `/etc/ssh/sshd_config`.

## 12. Sample project
Thiết lập một Jump Server (Bastion Host). Bạn chỉ có thể SSH vào các server nội bộ sau khi đã SSH thành công vào Jump Server này. Điều này giúp thu hẹp bề mặt tấn công của hệ thống.

## 13. Interview
### Core Q&A
1. **Q**: Cổng mặc định của SSH là bao nhiêu?
   **A**: Cổng 22.
2. **Q**: Sự khác biệt giữa Symmetric và Asymmetric encryption trong SSH?
   **A**: Asymmetric (Public/Private Key) dùng cho bước xác thực ban đầu. Symmetric (Session Key) dùng để mã hóa dữ liệu thực tế được truyền đi trong suốt phiên làm việc vì nó nhanh hơn.
3. **Q**: Làm thế nào để đổi cổng mặc định của SSH?
   **A**: Chỉnh sửa tham số `Port` trong tệp `/etc/ssh/sshd_config` và khởi động lại dịch vụ ssh.

### Scenario
**Tình huống**: Bạn gặp lỗi "Connection refused" khi cố gắng SSH vào một server AWS mới tạo. Nguyên nhân có thể là gì?
**Trả lời**: Có 3 khả năng chính: 1. Security Group của AWS chưa mở cổng 22 cho IP của tôi. 2. Dịch vụ SSH trên server chưa được khởi động. 3. Tôi đang gõ sai địa chỉ IP.

## 14. References
- Official Website: [OpenSSH](https://www.openssh.com/)
- Wikipedia: [Secure Shell](https://en.wikipedia.org/wiki/Secure_Shell)

## 15. Real-world Code
- Tệp cấu hình Client: `~/.ssh/config` dùng để lưu các bí danh kết nối.

## 16. Community
- Stack Overflow: Tag [ssh]
- Reddit: r/linuxadmin
