---
created: 2026-05-05
tags:
  - "#type/tutorial"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/security"
related:
  - "[[ec2]]"
---

## 1. What
Checklist Backdoor AWS là tập hợp các câu lệnh shell dùng để kiểm tra và phát hiện các dấu vết xâm nhập, các điểm truy cập trái phép hoặc các tiến trình độc hại đang chạy trên các thực thể EC2 (Linux).

## 2. Why
Khi một kẻ tấn công chiếm được quyền truy cập vào server, chúng thường thiết lập các backdoor để duy trì sự hiện diện (persistence). Việc kiểm tra định kỳ giúp phát hiện sớm các thay đổi bất thường mà các công cụ giám sát tự động có thể bỏ lỡ.

## 3. Mental Model
Hãy tưởng tượng bạn đang thực hiện một đợt tổng vệ sinh và kiểm tra an ninh cho ngôi nhà của mình. Bạn sẽ kiểm tra xem có ai lạ đang ở trong nhà không (User), các ổ khóa có bị thay đổi không (SSH Keys), có ai đặt lịch đột nhập không (Cron jobs), và có các thiết bị nghe lén nào đang hoạt động không (Network connections).

## 4. Where it fits
Security Audit -> Incident Response -> Post-Exploitation Detection. 
Đây là bước đầu tiên trong quá trình điều tra số (Digital Forensics) khi nghi ngờ hệ thống bị thỏa hiệp.

## 5. When to use
- Ngay sau khi phát hiện các dấu hiệu bất thường (CPU cao, log lạ).
- Định kỳ hằng tháng để audit bảo mật.
- Khi bàn giao hệ thống từ bên thứ ba hoặc nhân viên cũ.

## 6. When NOT to use
- Không nên dùng làm giải pháp bảo mật duy nhất; đây chỉ là kiểm tra thủ công.
- Khi hệ thống đã bị chiếm quyền root hoàn toàn và kẻ tấn công sử dụng rootkit để che giấu file/tiến trình (lúc này các lệnh cat/ps sẽ trả về kết quả giả).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Nhanh chóng, không cần cài đặt thêm tool. | Dễ bị bypass bởi attacker chuyên nghiệp sử dụng rootkit. |
| Sử dụng các lệnh tiêu chuẩn của Linux. | Tạo ra nhiều kết quả gây nhiễu (false positive). |
| Giúp hiểu sâu về cấu trúc hệ thống. | Tốn thời gian nếu phải thực hiện thủ công trên nhiều server. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| AWS Inspector | Tự động hóa, quét được nhiều lỗ hổng nhưng có chi phí. |
| Lynis | Tool audit bảo mật mã nguồn mở, chuyên sâu hơn nhiều. |
| Wazuh | Giải pháp XDR/SIEM toàn diện, giám sát thời gian thực. |

## 9. How
Dưới đây là các lệnh kiểm tra cơ bản:

### Check user lạ
```bash
cat /etc/passwd | grep -v nologin | grep -v false
```

### Check SSH authorized keys lạ
```bash
cat ~/.ssh/authorized_keys
sudo cat /root/.ssh/authorized_keys
```

### Check cron job lạ
```bash
crontab -l
sudo crontab -l
cat /etc/crontab
ls /etc/cron.d/
```

### Check process lạ đang chạy
```bash
ps aux | grep -v "systemd\|sshd\|java\|node\|nginx"
```

### Check network connection lạ
```bash
sudo netstat -tulpn
# Hoặc dùng ss
sudo ss -tulpn
```

### Check service lạ
```bash
sudo systemctl list-units --type=service --state=running
```

### Check file mới được tạo/sửa gần đây (trong 7 ngày qua)
```bash
sudo find / -mtime -7 -type f -not -path "/proc/*" \
  -not -path "/sys/*" 2>/dev/null | head -50
```

## 10. Production concerns
### Scaling
Trên quy mô lớn, nên đẩy các script này vào AWS Systems Manager (SSM) Run Command để thực thi đồng loạt trên hàng nghìn instance.

### Failure
Nếu các lệnh cơ bản bị thay thế (ví dụ lệnh ps bị attacker sửa đổi), kết quả sẽ không đáng tin cậy. Nên sử dụng các static binary từ nguồn tin cậy.

### Monitoring
Nên chuyển kết quả từ các lệnh này về CloudWatch Logs hoặc một hệ thống log tập trung để phân tích alert.

## 11. Common mistakes
- Mistake: Chỉ kiểm tra user hiện tại mà quên kiểm tra root user.
  Fix: Luôn sử dụng sudo để kiểm tra thư mục /root và các file hệ thống.

- Mistake: Bỏ qua các cron job trong /etc/cron.d/.
  Fix: Kiểm tra tất cả các vị trí lưu trữ định kỳ của hệ thống Linux.

## 12. Sample project
Xây dựng một script bash tự động chạy các lệnh trên và gửi báo cáo về Slack qua Webhook khi phát hiện có User mới hoặc SSH Key mới được thêm vào.

## 13. Interview
### Core Q&A
1. Q: Tại sao cần kiểm tra cả /etc/passwd và /etc/shadow?
   A: /etc/passwd chứa thông tin user cơ bản, trong khi /etc/shadow chứa hash mật khẩu. Kiểm tra cả hai giúp xác định xem có user nào được tạo mà không có mật khẩu hoặc có mật khẩu yếu không.

2. Q: Làm thế nào để biết một process lạ là độc hại hay hợp lệ?
   A: Kiểm tra đường dẫn thực thi của nó (ls -l /proc/PID/exe), kiểm tra các file nó đang mở (lsof -p PID) và check network connection mà nó đang khởi tạo.

3. Q: SSH Backdoor thường được đặt ở đâu nhất?
   A: Thường nằm trong file authorized_keys của user root hoặc các user có quyền sudo.

### Scenario
Hệ thống báo CPU tăng vọt lên 100% trên EC2. Bạn sử dụng `top` thấy một process tên `kworker` nhưng chiếm 90% CPU. Bạn sẽ làm gì?
Trả lời: Kiểm tra path của process đó. Thông thường `kworker` là của kernel, nếu nó chạy từ `/tmp` hoặc `/var/tmp` thì chắc chắn là malware (thường là miner). Sau đó dùng `lsof` để xem nó kết nối đi đâu.

## 14. References
- Official Docs: AWS Security Best Practices
- GitHub Repo: payload-all-the-things
- Spec / RFC: MITRE ATT&CK Framework
- Changelog: Linux Security Modules (LSM)

## 15. Real-world Code
Tham khảo các script audit của Lynis trên GitHub để thấy cách họ kiểm tra backdoor chuyên nghiệp.

## 16. Community
- Reddit: r/aws, r/cybersecurity
- Stack Overflow: Security tags
- Blog: AWS Security Blog
- Talk: DEF CON Cloud Village talks
