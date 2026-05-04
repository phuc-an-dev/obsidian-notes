---
created: 2026-05-04
tags:
  - "#type/tutorial"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/network"
related:
  - "[[SSH]]"
  - "[[chmod 400]]"
  - "[[ec2]]"
---

## 1. What
Lệnh `ssh -i "my-key.pem" ubuntu@54.209.227.108` là một lệnh cụ thể dùng để thiết lập kết nối SSH tới một máy chủ Ubuntu từ xa (có địa chỉ IP là 54.209.227.108) sử dụng một tệp khóa riêng tư (**Private Key**) tên là `my-key.pem` để xác thực thay vì dùng mật khẩu.

## 2. Why
Xác thực bằng Key (Public Key Authentication) là phương pháp bảo mật nhất để truy cập server. Nó ngăn chặn các cuộc tấn công Brute-force vì kẻ tấn công không thể đoán được nội dung của tệp key dài và phức tạp. Tham số `-i` (identity file) cho phép bạn chỉ định chính xác tệp key nào được dùng cho kết nối này.

## 3. Mental Model
Hãy tưởng tượng việc kết nối này như **"Mở một két sắt bằng thẻ từ"**:
- `ssh`: Là hành động bạn đi tới két sắt.
- `-i "my-key.pem"`: Là chiếc thẻ từ (Private Key) bạn cầm trên tay.
- `ubuntu`: Là tên của ngăn chứa bên trong két sắt mà bạn có quyền vào.
- `54.209.227.108`: Là địa chỉ nhà của ngân hàng chứa cái két sắt đó.
Chỉ khi thẻ từ khớp với ổ khóa đã được cài đặt sẵn bên trong két, cửa mới mở ra cho bạn.

## 4. Where it fits
Nằm ở bước truy cập trực tiếp vào hạ tầng sau khi đã khởi tạo Instance:
`AWS Console (Launch EC2) -> Get Public IP & Key -> Run SSH Command -> Remote Shell Access`

## 5. When to use
- Khi bạn cần đăng nhập vào EC2 Instance lần đầu tiên sau khi tạo.
- Khi server đã tắt tính năng đăng nhập bằng mật khẩu (mặc định của AWS Ubuntu image).
- Khi bạn quản lý nhiều server khác nhau với các cặp key khác nhau.

## 6. When NOT to use
- Khi tệp `.pem` chưa được phân quyền đúng (`chmod 400`). SSH sẽ từ chối dùng key nếu nó "quá hở".
- Khi bạn đang đứng bên trong mạng nội bộ và đã cấu hình SSH Config (`~/.ssh/config`) để dùng bí danh (alias) cho ngắn gọn.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Bảo mật cực cao, không thể bị đoán mật khẩu. | Nếu mất tệp `.pem`, bạn bị khóa khỏi server vĩnh viễn. |
| Không cần gõ mật khẩu mỗi khi đăng nhập. | Cú pháp lệnh dài và khó nhớ đối với người mới. |
| Có thể tự động hóa trong các script. | Tệp key cần được lưu trữ và bảo vệ cực kỳ cẩn thận. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| **SSH Config File** | Lưu cấu hình vào `~/.ssh/config`, sau đó chỉ cần gõ `ssh myserver`. |
| **SSH Agent** | Dùng `ssh-add` để lưu key vào bộ nhớ, không cần tham số `-i` mỗi lần gõ. |
| **AWS Instance Connect** | Truy cập trực tiếp qua trình duyệt web trên AWS Console. |

## 9. How
```bash
# BƯỚC 1: Phải phân quyền cho key trước (bắt buộc)
chmod 400 my-key.pem

# BƯỚC 2: Thực hiện kết nối
ssh -i "my-key.pem" ubuntu@54.209.227.108

# Giải thích các thành phần:
# ssh: Lệnh thực thi
# -i: Tham số chỉ định identity file (khóa)
# "my-key.pem": Đường dẫn tới tệp khóa của bạn
# ubuntu: User mặc định trên Ubuntu (AWS)
# @: Ký tự phân cách
# 54.209.227.108: IP Public của server
```

## 10. Production concerns
### Scaling
Trong môi trường doanh nghiệp, thay vì chia sẻ tệp `.pem`, người ta dùng các giải pháp như `HashiCorp Vault` hoặc `AWS IAM Instance Connect` để cấp quyền truy cập tạm thời theo danh tính cá nhân.

### Failure
Nếu gặp lỗi "Permission denied (publickey)", hãy kiểm tra: 1. Đúng User chưa (ubuntu vs root vs ec2-user)? 2. Đúng IP chưa? 3. File key có đúng là bản đi kèm với server lúc tạo không?

### Monitoring
Log đăng nhập SSH được ghi lại tại `/var/log/auth.log`. Bạn có thể thấy các nỗ lực dùng sai key tại đây.

## 11. Common mistakes
- **Mistake**: Quên gõ `-i` và tệp key. SSH sẽ thử dùng mật khẩu hoặc key mặc định và thất bại.
  **Fix**: Luôn chỉ định rõ đường dẫn key nếu không dùng SSH Agent.

- **Mistake**: Sai tên User. Mỗi Distro có user mặc định khác nhau (Ubuntu là `ubuntu`, Amazon Linux là `ec2-user`, Debian là `admin`).
  **Fix**: Kiểm tra tài liệu của OS image bạn đang dùng.

## 12. Sample project
Tạo một bí danh (alias) trong tệp `.bashrc` hoặc `.zshrc`:
`alias my-server='ssh -i ~/keys/my-key.pem ubuntu@54.209.227.108'`
Từ nay bạn chỉ cần gõ `my-server` để kết nối.

## 13. Interview
### Core Q&A
1. **Q**: Chuyện gì xảy ra nếu bạn không `chmod 400` cho file `.pem`?
   **A**: Lệnh SSH sẽ báo lỗi "Permissions for 'my-key.pem' are too open" và yêu cầu bạn sửa quyền trước khi cho phép kết nối.
2. **Q**: Làm sao để SSH vào server nếu server nằm trong mạng nội bộ (Private Subnet)?
   **A**: Phải dùng một **Bastion Host** (Jump Server) hoặc thiết lập VPN/AWS Site-to-Site Connection.
3. **Q**: Tham số `-v` trong lệnh SSH dùng để làm gì?
   **A**: Để bật chế độ "verbose", giúp debug quá trình kết nối, xem nó bị kẹt ở bước nào (ví dụ: `ssh -v -i ...`).

### Scenario
**Tình huống**: Bạn đã gõ đúng lệnh nhưng bị "Connection timeout". Bạn xử lý thế nào?
**Trả lời**: Tôi sẽ kiểm tra Security Group của EC2 xem đã mở cổng 22 cho IP của máy tôi chưa. Nếu đã mở, tôi kiểm tra lại xem server có đang ở trạng thái "Running" không.

## 14. References
- AWS Docs: [Connect to your Linux instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html)

## 15. Real-world Code
- Một đoạn mã script tự động:
  ```bash
  KEY_PATH="./prod-key.pem"
  SERVER_IP="13.250.1.2"
  ssh -o StrictHostKeyChecking=no -i "$KEY_PATH" ubuntu@$SERVER_IP
  ```

## 16. Community
- Stack Overflow: Tag [ssh-key]
- Reddit: r/aws
