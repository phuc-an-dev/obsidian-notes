---
created: 2026-05-06
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/http"
related:
  - "[[aws-ec2-instance.md]]"
  - "[[aws-ec2-security-groups.md]]"
  - "[[AWS Inbound Rules.md]]"
---

## 1. What
SSH (Secure Shell) tools trên macOS là các phần mềm hoặc giao diện dòng lệnh cho phép người dùng thiết lập kết nối mã hóa an toàn đến các máy ảo EC2 trên AWS. Trên macOS, người dùng có thể sử dụng từ công cụ mặc định (Terminal) đến các ứng dụng bên thứ ba có giao diện đồ họa mạnh mẽ (GUI).

## 2. Why
Việc quản lý server EC2 yêu cầu thao tác trực tiếp qua dòng lệnh. Mặc dù Terminal mặc định của macOS rất mạnh mẽ, nhưng khi số lượng server tăng lên, việc ghi nhớ IP, Key Pair (.pem) và User trở nên khó khăn. Các công cụ SSH chuyên dụng giúp quản lý danh sách server, lưu trữ key an toàn và hỗ trợ đa nhiệm tốt hơn.

## 3. Mental Model
Hãy tưởng tượng các công cụ SSH giống như những **"Chùm chìa khóa thông minh"**.
- Terminal mặc định là một chiếc chìa khóa vạn năng đơn giản: bạn phải tự nhớ ổ khóa nào dùng chìa nào.
- Các công cụ như Termius hay Tabby là những chiếc ví đựng chìa khóa cao cấp: chúng ghi nhãn từng chìa, tự động tra vào ổ khi bạn chọn và có thể đồng bộ giữa các thiết bị.

## 4. Where it fits
Vị trí trong luồng làm việc:
`macOS User -> SSH Tool -> Internet (Port 22) -> AWS Security Group -> EC2 Instance`

## 5. When to use
- Khi cần cấu hình, cài đặt phần mềm hoặc kiểm tra log trực tiếp trên EC2.
- Terminal (Native): Dùng cho các tác vụ nhanh, đơn giản hoặc khi không muốn cài thêm phần mềm.
- Termius/Tabby: Dùng khi quản lý hàng chục server, cần chia nhóm hoặc đồng bộ cấu hình giữa các máy tính.

## 6. When NOT to use
- Khi có thể sử dụng AWS Systems Manager (SSM) Session Manager để truy cập không cần mở Port 22 (Bảo mật hơn).
- Khi chỉ cần thực hiện các tác vụ tự động hóa hàng loạt (nên dùng Ansible hoặc Terraform thay vì SSH thủ công).

## 7. Trade-offs
| Tool | Pros | Cons |
|------|------|------|
| **Terminal (Built-in)** | Sẵn có, cực kỳ nhẹ, ổn định tuyệt đối. | Khó quản lý nhiều Profile/Key Pair, giao diện đơn điệu. |
| **Termius** | Giao diện đẹp, đồng bộ Cloud, quản lý Key cực tốt. | Bản đầy đủ (Sync) tốn phí hàng tháng. |
| **Tabby** | Open-source, hỗ trợ nhiều Tab, Plugin phong phú. | Nặng hơn Terminal vì viết trên Electron. |
| **iTerm2** | Tùy biến cực mạnh, hỗ trợ Split Panes, Hotkeys. | Chỉ dành cho macOS, vẫn là trình giả lập Terminal. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| AWS Instance Connect | Truy cập trực tiếp từ trình duyệt trên Console AWS, không cần app ngoài. |
| VS Code Remote SSH | Cho phép sửa code trực tiếp trên EC2 bằng giao diện VS Code quen thuộc. |

## 9. How
Cách kết nối cơ bản từ Terminal macOS:

```bash
# 1. Cấp quyền đọc cho file key pair
chmod 400 my-key.pem

# 2. Kết nối SSH
ssh -i "my-key.pem" ec2-user@ec2-11-22-33-44.compute-1.amazonaws.com
```

Cấu hình file `~/.ssh/config` để kết nối nhanh hơn:
```text
Host dev-server
    HostName 11.22.33.44
    User ec2-user
    IdentityFile ~/keys/my-key.pem
```
Sau đó chỉ cần gõ: `ssh dev-server`.

## 10. Production concerns
### Security
Luôn bảo vệ file `.pem`. Không bao giờ commit file key pair lên GitHub. Sử dụng SSH Agent Forwarding nếu cần kết nối từ server này sang server khác.

### Availability
Khi IP public của EC2 thay đổi (nếu không dùng Elastic IP), bạn phải cập nhật lại cấu hình trong tool SSH.

### Monitoring
Sử dụng tính năng "Keep-alive" trong các tool SSH để tránh bị ngắt kết nối khi không thao tác trong thời gian ngắn.

## 11. Common mistakes
- Mistake: Để quyền file `.pem` quá lỏng lẻo (ví dụ 777). SSH sẽ từ chối kết nối.
  Fix: Luôn dùng `chmod 400 path/to/key.pem`.

- Mistake: Quên mở Port 22 trong AWS Security Group cho IP của mình.
  Fix: Kiểm tra Inbound Rules trong Security Group trên AWS Console.

## 12. Sample project
1. Khởi tạo một EC2 instance trên AWS.
2. Tải file `.pem` về macOS.
3. Cài đặt iTerm2 và cấu hình Profile để SSH vào server đó chỉ bằng một cú click hoặc phím tắt.

## 13. Interview
### Core Q&A
1. Q: Tại sao macOS là nền tảng tốt cho việc SSH vào EC2?
   A: Vì macOS dựa trên Unix, có sẵn OpenSSH client tương đồng với môi trường Linux của EC2, giúp các lệnh shell hoạt động nhất quán.

2. Q: SSH Agent là gì?
   A: Là một chương trình chạy ngầm giúp lưu trữ các private key đã được giải mã, giúp bạn không phải nhập passphrase hoặc chỉ định file `-i` mỗi lần kết nối.

### Scenario
"Bạn mất file .pem nhưng cần SSH vào EC2 gấp, bạn xử lý thế nào?"
-> Trả lời: Nếu không có AWS Systems Manager (SSM) được cài sẵn, cách duy nhất là tạo một AMI từ instance đó, sau đó launch một instance mới từ AMI với Key Pair mới.

## 14. References
- SSH Config Manual: `man ssh_config`
- Termius Official: [termius.com](https://termius.com/)
- Tabby GitHub: [github.com/Eugeny/tabby](https://github.com/Eugeny/tabby)

## 15. Real-world Code
Nhiều DevOps Engineer sử dụng công cụ `sshuttle` để tạo VPN giả lập qua SSH hoặc `mosh` (Mobile Shell) để duy trì kết nối khi mạng chập chờn trên macOS.

## 16. Community
- Reddit: r/macsysadmin
- Stack Overflow: Tag [ssh] [macos]
- iTerm2 Community
