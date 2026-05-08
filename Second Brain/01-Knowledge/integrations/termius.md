---
created: 2026-05-06
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/networking"
related:
  - "[[ssh-tools-ec2-macos.md]]"
---

## 1. What
Termius là một giải pháp quản lý SSH (Secure Shell) hiện đại, đa nền tảng, cho phép người dùng kết nối và quản trị các máy chủ từ xa một cách an toàn. Ngoài tính năng SSH client cơ bản, nó còn tích hợp quản lý SFTP, port forwarding, và đồng bộ dữ liệu qua đám mây.

## 2. Why
Việc dùng Terminal truyền thống để quản lý hàng chục server với các Key Pair (.pem) khác nhau rất dễ gây nhầm lẫn và tốn thời gian. Termius giải quyết các vấn đề:
- **Tổ chức**: Nhóm các server theo dự án hoặc khách hàng.
- **Đồng bộ**: Truy cập danh sách server và key từ bất kỳ thiết bị nào (Laptop, Điện thoại, Tablet).
- **Tự động hóa**: Tự động điền mật khẩu hoặc nạp key khi kết nối.
- **Hợp tác**: Chia sẻ quyền truy cập server cho đồng nghiệp một cách an toàn (bản Team).

## 3. Mental Model
Hãy tưởng tượng Termius giống như một **"Danh bạ điện thoại thông minh"** dành cho server:
- Thay vì bạn phải nhớ số điện thoại (IP) và mang theo chìa khóa vật lý (PEM file) cho từng ngôi nhà.
- Termius lưu sẵn mọi thứ. Bạn chỉ cần nhấn vào tên "Server Production", và nó sẽ tự động dùng đúng chìa khóa để mở cửa cho bạn.
- Danh bạ này tự động cập nhật trên mọi thiết bị bạn có.

## 4. Where it fits
Vị trí trong quy trình làm việc:
`Lập trình viên -> Termius App -> Cloud Sync (Optional) -> SSH Tunnel -> Remote Server`

## 5. When to use
- Khi bạn phải quản lý nhiều máy chủ cloud (AWS EC2, DigitalOcean, v.v.).
- Khi thường xuyên phải làm việc trên nhiều thiết bị khác nhau và muốn giữ cấu hình SSH đồng nhất.
- Khi cần một giao diện SFTP trực quan để kéo thả file thay vì dùng lệnh `scp`.

## 6. When NOT to use
- Nếu bạn chỉ làm việc với duy nhất 1 server và ưu tiên sự tối giản tuyệt đối (dùng Terminal mặc định là đủ).
- Trong các môi trường yêu cầu bảo mật cực cao, nơi chính sách công ty cấm sử dụng các công cụ có tính năng đồng bộ Cloud bên thứ ba.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Giao diện cực đẹp, trực quan và dễ dùng. | Các tính năng tốt nhất (Sync, SFTP, Snippets) yêu cầu trả phí hàng tháng. |
| Đồng bộ đa thiết bị là tính năng độc bản. | Nặng hơn các client truyền thống như PuTTY hoặc Terminal. |
| Hỗ trợ Snippets để chạy nhanh các lệnh phổ biến. | Phụ thuộc vào hạ tầng cloud của Termius nếu dùng tính năng Sync. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Tabby | Miễn phí, mã nguồn mở, hỗ trợ nhiều plugin nhưng không có đồng bộ cloud mượt bằng. |
| iTerm2 | Miễn phí, cực mạnh trên macOS nhưng không có tính năng quản lý host chuyên sâu. |
| Royal TSX | Quản lý đa giao thức (RDP, SSH, VNC) cực mạnh cho Enterprise nhưng giao diện hơi cũ. |

## 9. How
Các bước thiết lập cơ bản:
1. **Add Host**: Nhập Label, IP (Address) và Username.
2. **Key Management**: Import file `.pem` hoặc `.pub` vào phần "Keys".
3. **Link Key**: Gán key vừa tạo vào Host.
4. **Connect**: Nhấn đúp vào Host để bắt đầu phiên làm việc.

Tính năng hữu ích: **Snippets**
Lưu lệnh: `tail -f /var/log/nginx/error.log`
Chỉ cần chọn snippet này khi đang ở trong server, Termius sẽ tự động gõ và chạy lệnh cho bạn.

## 10. Production concerns
### Encryption
Dữ liệu đồng bộ trên Cloud của Termius được mã hóa đầu cuối (End-to-End Encryption). Chỉ có thiết bị của bạn mới có chìa khóa để giải mã danh sách server và keys.

### Local Storage
Nếu không dùng Cloud, Termius lưu dữ liệu trong một database cục bộ được mã hóa. Hãy đảm bảo bạn nhớ "Master Password" của ứng dụng.

## 11. Common mistakes
- Mistake: Quên Master Password. Bạn sẽ mất quyền truy cập vào toàn bộ database keys đã lưu.
- Mistake: Để tính năng Cloud Sync bật trên các thiết bị không an toàn (như điện thoại không có mã khóa).

## 12. Sample project
Sử dụng Termius để thiết lập một SSH Tunnel (Port Forwarding):
1. Cấu hình để port 3306 của Database Server (nằm trong mạng nội bộ) được map về port 3307 của máy local.
2. Dùng TablePlus kết nối tới `localhost:3307` để quản trị database mà không cần mở port DB ra công chúng.

## 13. Interview
### Core Q&A
1. Q: Termius có an toàn không khi lưu trữ Private Key trên mây?
   A: Termius sử dụng mã hóa AES-256 đầu cuối. Chìa khóa giải mã được tạo ra từ Master Password của bạn và không bao giờ được gửi về server của Termius.

2. Q: Làm thế nào để di chuyển dữ liệu từ Terminal sang Termius?
   A: Termius hỗ trợ import file `~/.ssh/config`. Nó sẽ tự động nhận diện các host, user và đường dẫn key.

### Scenario
"Bạn đang đi du lịch và server Production gặp sự cố. Bạn không mang theo laptop. Termius giúp gì cho bạn?"
-> Trả lời: Nhờ tính năng Cloud Sync, tôi có thể mở app Termius trên điện thoại di động, toàn bộ danh sách server và keys đã có sẵn. Tôi có thể SSH vào server ngay lập tức để kiểm tra logs hoặc restart dịch vụ chỉ trong vài giây.

## 14. References
- Official Website: [termius.com](https://termius.com/)
- Documentation: [Termius Support](https://support.termius.com/hc/en-us)

## 15. Real-world Code
Termius cho phép xuất danh sách host ra định dạng JSON hoặc YAML để phục vụ mục đích backup thủ công.

## 16. Community
- Reddit: r/termius.
- Termius Blog: Các bài viết về mẹo quản trị server.
