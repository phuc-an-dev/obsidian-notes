---
created: 2026-05-04
tags:
  - "#type/tutorial"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/performance"
related:
  - "[[ec2]]"
  - "[[SSH]]"
---

## 1. What
Đây là tập hợp các lệnh cơ bản dùng để kiểm tra tài nguyên và thông tin hệ thống trên Linux:
- `free -h`: Xem dung lượng RAM (đã dùng, còn trống) dưới dạng dễ đọc (GB, MB).
- `nproc`: Xem số lượng nhân CPU (cores) khả dụng.
- `df -h`: Xem dung lượng và tình trạng sử dụng của các ổ đĩa.
- `curl ifconfig.me`: Lấy địa chỉ IP Public thực tế của server từ bên ngoài.

## 2. Why
Khi quản trị server, bạn cần biết máy chủ của mình "mạnh" đến đâu và có đang bị quá tải hay không.
- Kiểm tra RAM/CPU giúp phát hiện bottleneck về hiệu năng.
- Kiểm tra Disk (`df`) giúp tránh tình trạng server bị treo do đầy ổ cứng.
- Kiểm tra IP Public để xác nhận cấu hình mạng và kết nối internet đang hoạt động đúng.

## 3. Mental Model
Hãy tưởng tượng các lệnh này như **"Bảng đồng hồ trên xe ô tô"**:
- `free` và `nproc` giống như Kim xăng và Đồng hồ vòng tua máy (Máy khỏe không, còn sức chạy không?).
- `df` giống như Đồng hồ báo dung lượng khoang chứa đồ (Còn chỗ chứa hàng không?).
- `curl ifconfig.me` giống như việc nhìn ra biển hiệu bên đường để biết xe mình đang thực sự đứng ở đâu trên bản đồ thế giới.

## 4. Where it fits
Nằm ở tầng giám sát hệ thống (System Monitoring) cơ bản:
`User -> Execute Commands -> Linux Kernel / System Files (/proc) -> Console Output`

## 5. When to use
- Ngay sau khi SSH vào một server lạ để nắm bắt thông số.
- Khi ứng dụng chạy chậm hoặc báo lỗi "Out of Memory" / "No space left on device".
- Khi cần cấu hình các tham số phần mềm dựa trên phần cứng (ví dụ: đặt số lượng worker bằng số nhân CPU).

## 6. When NOT to use
- Khi bạn cần giám sát theo thời gian thực liên tục (nên dùng `top`, `htop` hoặc các giải pháp như `CloudWatch`, `Prometheus`).
- Khi cần phân tích sâu xem process nào đang chiếm tài nguyên (dùng `ps` hoặc `lsof`).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Cực kỳ nhanh, có sẵn trên hầu hết các distro Linux. | Chỉ cung cấp thông tin tại một thời điểm (snapshot). |
| Tham số `-h` (human-readable) giúp đọc số liệu dễ dàng. | Không cung cấp biểu đồ hay lịch sử tài nguyên. |
| Không cần cài đặt thêm thư viện ngoài. | Một số lệnh (như curl) yêu cầu kết nối internet. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| **htop / top** | Trực quan hơn, hiển thị process và cập nhật liên tục. |
| **du -sh** | Kiểm tra dung lượng của từng thư mục cụ thể (chi tiết hơn `df`). |
| **lscpu** | Hiển thị thông tin chi tiết về kiến trúc CPU (hơn chỉ là số nhân). |

## 9. How
```bash
# 1. Kiểm tra RAM
free -h
# Output mẫu: Mem: 2.0Gi 500Mi 1.0Gi ...

# 2. Kiểm tra số nhân CPU
nproc
# Output mẫu: 2

# 3. Kiểm tra ổ đĩa
df -h
# Output mẫu: /dev/xvda1 8.0G 2.5G 5.5G 32% /

# 4. Lấy IP Public
curl ifconfig.me
# Output mẫu: 54.209.227.108
```

## 10. Production concerns
### Scaling
Trong các hệ thống Auto Scaling, các thông số này được dùng làm "CloudWatch Metrics" để quyết định khi nào thì cần tăng thêm server (ví dụ: khi CPU > 70%).

### Failure
Lệnh `df -h` có thể bị treo (hang) nếu có một kết nối ổ đĩa mạng (NFS/EFS) bị mất kết nối. Cần cẩn trọng khi dùng trong script tự động.

### Monitoring
Nên thiết lập các cảnh báo (Alerts) tự động khi ổ đĩa đạt mức 90% thay vì đợi đến khi gõ lệnh thủ công.

## 11. Common mistakes
- **Mistake**: Nhìn vào cột "free" trong lệnh `free -h` và tưởng máy hết RAM.
  **Fix**: Linux dùng RAM trống để làm cache. Hãy nhìn vào cột **"available"** để biết lượng RAM thực tế ứng dụng có thể dùng.

- **Mistake**: Quên rằng `df -h` hiển thị theo phân vùng. Cần tìm dòng có Mount point là `/` để biết dung lượng đĩa chính.
  **Fix**: Tập trung vào dòng Root filesystem.

## 12. Sample project
Viết một script "Health Check" đơn giản gửi thông báo qua Telegram nếu dung lượng ổ đĩa còn dưới 1GB hoặc RAM khả dụng còn dưới 100MB.

## 13. Interview
### Core Q&A
1. **Q**: Làm sao để xem RAM theo đơn vị Megabyte thay vì Gigabyte?
   **A**: Dùng `free -m`.
2. **Q**: Sự khác biệt giữa `df` và `du`?
   **A**: `df` (disk free) báo cáo dung lượng của cả phân vùng hệ thống. `du` (disk usage) báo cáo dung lượng của một file hoặc thư mục cụ thể.
3. **Q**: Tại sao lệnh `curl ifconfig.me` lại khác với lệnh `ip addr`?
   **A**: `ip addr` cho biết IP nội bộ (Private IP) của card mạng. `ifconfig.me` cho biết IP Public mà thế giới bên ngoài nhìn thấy server của bạn qua NAT/Gateway.

### Scenario
**Tình huống**: Bạn chạy `df -h` thấy ổ cứng còn 50% trống, nhưng ứng dụng vẫn báo "No space left on device". Tại sao?
**Trả lời**: Có thể do phân vùng đó đã hết **Inodes** (số lượng tệp tin tối đa). Tôi sẽ kiểm tra bằng lệnh `df -i`.

## 14. References
- Man pages: `man free`, `man df`, `man nproc`.

## 15. Real-world Code
- Lệnh lấy số nhân để build ứng dụng nhanh hơn: `make -j $(nproc)`.

## 16. Community
- Reddit: r/linuxadmin
- Stack Overflow: Tag [linux-commands]
