---
created: 2026-05-04
tags:
  - "#type/tutorial"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/performance"
related:
  - "[[SSH]]"
  - "[[ec2]]"
---

## 1. What
Đây là tổ hợp hai lệnh cơ bản trong hệ điều hành dựa trên Debian (như Ubuntu) để quản lý phần mềm:
- `sudo apt update`: Cập nhật danh sách các gói phần mềm và phiên bản mới nhất từ kho lưu trữ (repositories).
- `sudo apt upgrade -y`: Thực hiện tải về và cài đặt các bản cập nhật thực tế. Tham số `-y` tự động trả lời "Yes" cho các câu hỏi xác nhận.

## 2. Why
Hệ điều hành và phần mềm luôn có các lỗ hổng bảo mật hoặc lỗi logic được phát hiện theo thời gian. Việc chạy lệnh này thường xuyên giúp hệ thống luôn ở trạng thái an toàn nhất, ổn định nhất và có được các tính năng mới nhất từ nhà phát triển.

## 3. Mental Model
Hãy tưởng tượng việc này như **"Cập nhật thực đơn và Đi chợ"**:
- `apt update` giống như việc bạn gọi điện cho nhà hàng để hỏi xem hôm nay có món gì mới không (Cập nhật danh sách món). Lúc này bạn chưa có đồ ăn trong tay.
- `apt upgrade` giống như việc bạn thực sự đi mua những món mới đó về và nấu (Cài đặt thực tế).

## 4. Where it fits
Nằm ở bước đầu tiên sau khi khởi tạo server hoặc định kỳ bảo trì:
`Launch Server -> SSH Login -> Update/Upgrade -> Install Specific Apps`

## 5. When to use
- Ngay sau khi bạn vừa mới tạo một EC2 Instance hoặc máy ảo mới.
- Định kỳ hàng tuần hoặc hàng tháng để vá lỗi bảo mật.
- Trước khi cài đặt một phần mềm lớn (như Docker, Nginx, Java).

## 6. When NOT to use
- Khi server đang chạy các dịch vụ cực kỳ nhạy cảm và bạn chưa kiểm tra tính tương thích của bản cập nhật mới trên môi trường Staging (có rủi ro làm gãy hệ thống).
- Khi đường truyền internet đang rất yếu hoặc chập chờn.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Đảm bảo an toàn bảo mật tối đa. | Có thể gây xung đột phiên bản với các phần mềm cũ. |
| Hệ thống chạy ổn định hơn. | Tốn băng thông và dung lượng ổ đĩa. |
| Quy trình tự động hóa dễ dàng. | Đôi khi yêu cầu khởi động lại (reboot) server. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| **unattended-upgrades** | Tự động cập nhật bảo mật ngầm mà không cần gõ lệnh thủ công. |
| **apt-get** | Lệnh cũ hơn của apt, hiện nay khuyến nghị dùng `apt` cho người dùng cuối. |
| **pacman/yum/dnf** | Các trình quản lý gói cho các distro Linux khác (Arch, CentOS). |

## 9. How
```bash
# Cách chạy an toàn nhất
sudo apt update && sudo apt upgrade -y

# Nếu muốn xem danh sách các gói sẽ được nâng cấp trước
sudo apt list --upgradable
```

## 10. Production concerns
### Scaling
Trong môi trường có hàng trăm server, người ta dùng các công cụ như `Ansible` hoặc `Systems Manager` của AWS để đẩy lệnh update đồng loạt thay vì SSH vào từng máy.

### Failure
Nếu lệnh bị ngắt giữa chừng do mất mạng, database của gói phần mềm có thể bị khóa (locked). Cách sửa: `sudo dpkg --configure -a`.

### Monitoring
Theo dõi các thông báo bảo mật từ các kênh tin tức của Linux Distro bạn đang dùng.

## 11. Common mistakes
- **Mistake**: Nghĩ rằng `apt update` đã là cài đặt xong phần mềm mới.
  **Fix**: Nhớ rằng `update` chỉ lấy thông tin, `upgrade` mới là thực thi cài đặt.

- **Mistake**: Quên `sudo` dẫn đến lỗi quyền hạn.
  **Fix**: Luôn dùng `sudo` cho các thao tác quản lý hệ thống.

## 12. Sample project
Viết một script Bash đơn giản chạy bằng Cronjob vào lúc 3 giờ sáng mỗi ngày để tự động chạy lệnh này và ghi kết quả vào file log.

## 13. Interview
### Core Q&A
1. **Q**: Tại sao dùng `&&` giữa hai lệnh?
   **A**: Để đảm bảo lệnh `upgrade` chỉ chạy nếu lệnh `update` kết thúc thành công.
2. **Q**: Sự khác biệt giữa `apt upgrade` và `apt dist-upgrade`?
   **A**: `upgrade` chỉ nâng cấp gói hiện có, không xóa gói nào. `dist-upgrade` thông minh hơn, có thể thêm hoặc xóa gói để giải quyết các phụ thuộc phức tạp của phiên bản mới.
3. **Q**: Làm sao để biết một gói có bản cập nhật mà không cần update toàn bộ?
   **A**: `sudo apt update` sau đó `apt list --upgradable | grep <tên-gói>`.

### Scenario
**Tình huống**: Bạn chạy lệnh upgrade và nhận được thông báo "Keep the local version or install the maintainer's version" cho một file config. Bạn chọn gì?
**Trả lời**: Nếu tôi đã tùy chỉnh file đó rất nhiều (như nginx.conf), tôi sẽ giữ lại bản local. Sau đó tôi sẽ dùng lệnh `diff` để xem bản mới có gì quan trọng cần bổ sung thủ công hay không.

## 14. References
- Manual: `man apt`
- Ubuntu Documentation: [Package Management](https://help.ubuntu.com/lts/serverguide/package-management.html)

## 15. Real-world Code
- Luôn là dòng đầu tiên trong các file `Dockerfile` hoặc `Cloud-init` script.

## 16. Community
- Stack Overflow: Tag [apt], [ubuntu]
- Reddit: r/linux4noobs
