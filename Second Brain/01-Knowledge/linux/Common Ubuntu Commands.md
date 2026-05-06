---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/others"
  - "#topic/linux"
related:
  - "[[Docker in Ubuntu.md]]"
---

## 1. What
Common Ubuntu Commands là tập hợp các lệnh giao diện dòng lệnh (CLI) cơ bản và thiết yếu nhất để quản trị hệ điều hành Ubuntu 24.04 (Noble Numbat). Các lệnh này giúp người dùng tương tác với hệ thống tập tin, quản lý gói phần mềm, kiểm soát tiến trình và cấu hình mạng.

## 2. Why
Sử dụng dòng lệnh là kỹ năng bắt buộc đối với mọi lập trình viên và kỹ sư DevOps vì tính hiệu quả, khả năng tự động hóa và sự cần thiết khi làm việc với các máy chủ không có giao diện đồ họa (Headless servers). Ubuntu 24.04 giới thiệu một số cải tiến về bảo mật và quản lý gói, do đó việc nắm vững các lệnh này giúp tối ưu hóa hiệu suất làm việc.

## 3. Mental Model
Hãy tưởng tượng Ubuntu CLI giống như một **"Bàn điều khiển trung tâm"** của một con tàu vũ trụ.
- Thay vì dùng chuột để nhấn các nút ảo trên màn hình (GUI), bạn gửi các mật lệnh trực tiếp vào hệ thống.
- Các lệnh ngắn gọn nhưng cực kỳ mạnh mẽ, cho phép bạn điều khiển mọi ngóc ngách của con tàu từ động cơ (Kernel) đến kho lương (File system).

## 4. Where it fits
Vị trí trong hệ thống:
`User -> Shell (Bash/Zsh) -> Ubuntu CLI Commands -> Linux Kernel -> Hardware`

## 5. When to use
- Khi quản trị máy chủ từ xa qua SSH.
- Khi cần thực hiện các tác vụ lặp đi lặp lại một cách nhanh chóng.
- Khi cài đặt môi trường lập trình (JDK, Node.js, Docker).
- Khi xử lý sự cố hệ thống (Troubleshooting).

## 6. When NOT to use
- Khi bạn là người dùng phổ thông chỉ cần các tác vụ văn phòng cơ bản và đã có sẵn GUI thân thiện.
- Khi thực hiện các lệnh nguy hiểm (như xóa dữ liệu) mà chưa hiểu rõ hậu quả (nên dùng GUI hoặc backup trước).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Tốc độ thực hiện cực nhanh. | Khó nhớ đối với người mới bắt đầu. |
| Tiết kiệm tài nguyên hệ thống (không cần RAM cho GUI). | Một sai sót nhỏ có thể làm hỏng toàn bộ hệ thống (đặc biệt với sudo). |
| Dễ dàng tự động hóa bằng script. | Giao diện khô khan, khó quan sát tổng thể trực quan. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Ubuntu Desktop GUI | Dễ dùng nhưng chậm và không hỗ trợ tự động hóa tốt. |
| Webmin / Cockpit | Giao diện web quản trị server, trực quan nhưng vẫn cần CLI cho các tác vụ sâu. |

## 9. How
Dưới đây là các nhóm lệnh phổ biến nhất trên Ubuntu 24.04:

### Quản lý gói (APT)
```bash
sudo apt update             # Cập nhật danh sách gói từ repository
sudo apt upgrade            # Nâng cấp các gói đã cài đặt lên phiên bản mới
sudo apt install <package>  # Cài đặt một gói mới
sudo apt remove <package>   # Gỡ bỏ một gói
sudo apt autoremove         # Xóa các gói phụ thuộc không còn dùng đến
```

### Hệ thống tập tin (File System)
```bash
ls -la                      # Liệt kê file/thư mục (bao gồm file ẩn)
cd <path>                   # Di chuyển giữa các thư mục
pwd                         # Hiển thị đường dẫn thư mục hiện hành
mkdir -p <dir>              # Tạo thư mục (bao gồm cả thư mục cha nếu chưa có)
rm -rf <path>               # Xóa file hoặc thư mục (cẩn thận!)
cp -r <src> <dest>          # Sao chép file/thư mục
mv <src> <dest>             # Di chuyển hoặc đổi tên file/thư mục
```

### Quản lý hệ thống & Tiến trình
```bash
top                         # Theo dõi tài nguyên hệ thống thời gian thực
htop                        # Phiên bản trực quan hơn của top (cần cài đặt)
df -h                       # Kiểm tra dung lượng ổ đĩa
free -h                     # Kiểm tra dung lượng RAM
ps aux                      # Liệt kê tất cả tiến trình đang chạy
sudo systemctl status <svc> # Kiểm tra trạng thái của một service (nginx, docker)
```

### Mạng (Networking)
```bash
ip addr                     # Xem địa chỉ IP của các interface
ping <host>                 # Kiểm tra kết nối tới một host
netstat -tuln               # Liệt kê các port đang lắng nghe (listening)
curl -I <url>               # Kiểm tra header của một URL
```

## 10. Production concerns
### Security
Luôn hạn chế sử dụng tài khoản `root` trực tiếp. Hãy dùng `sudo` để thực hiện các lệnh đặc quyền và chỉ cấp quyền tối thiểu cần thiết.

### Automation
Kết hợp các lệnh vào file `.sh` (Shell Script) để tự động hóa việc cấu hình server sau khi khởi tạo (Provisioning).

### Logs
Sử dụng `journalctl -u <service_name> -f` để theo dõi log của các dịch vụ trong thời gian thực trên Ubuntu hiện đại.

## 11. Common mistakes
- Mistake: Chạy lệnh `rm -rf /` hoặc các đường dẫn quan trọng.
  Fix: Luôn kiểm tra kỹ đường dẫn và cân nhắc dùng `-i` để xác nhận trước khi xóa.

- Mistake: Không chạy `apt update` trước khi `apt install`.
  Fix: Luôn cập nhật danh sách gói để tránh lỗi "Package not found" hoặc cài bản cũ.

## 12. Sample project
Thực hiện kịch bản:
1. Tạo thư mục `my_project`.
2. Tạo file `index.html` bên trong.
3. Cài đặt `nginx`.
4. Di chuyển file `index.html` vào thư mục của nginx (`/var/www/html`).
5. Kiểm tra trạng thái nginx và truy cập qua IP.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa `apt` và `apt-get` là gì?
   A: `apt` là công cụ mới hơn, thân thiện với người dùng hơn (có thanh tiến trình, màu sắc) và gộp các lệnh phổ biến nhất. `apt-get` là công cụ truyền thống, ổn định hơn cho việc viết script.

2. Q: Lệnh `sudo !!` dùng để làm gì?
   A: Dùng để thực hiện lại lệnh vừa gõ trước đó với quyền `sudo` (rất hữu ích khi bạn quên gõ sudo).

### Scenario
"Server của bạn đang bị chậm, bạn dùng các lệnh nào để tìm nguyên nhân?"
-> Trả lời:
1. `top` hoặc `htop` để tìm tiến trình chiếm nhiều CPU/RAM nhất.
2. `df -h` để xem có phân vùng nào bị đầy không.
3. `iostat` (nếu có) để kiểm tra độ trễ đọc/ghi ổ đĩa.
4. `journalctl -p err` để xem các log lỗi hệ thống gần nhất.

## 14. References
- Ubuntu Manpage: [manpages.ubuntu.com](https://manpages.ubuntu.com/)
- Linux Command: [linuxcommand.org](http://linuxcommand.org/)

## 15. Real-world Code
Nhiều sysadmin sử dụng `alias` trong `.bashrc` để rút ngắn các lệnh dài:
`alias update='sudo apt update && sudo apt upgrade -y'`

## 16. Community
- Ask Ubuntu: [askubuntu.com](https://askubuntu.com/)
- Ubuntu Forums: [ubuntuforums.org]
- Reddit: r/ubuntu
