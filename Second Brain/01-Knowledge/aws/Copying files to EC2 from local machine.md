---
created: 2026-05-07
tags:
  - "#type/tutorial"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/compute"
related:
  - "[[ec2.md]]"
  - "[[aws-ec2-key-pairs.md]]"
  - "[[ssh-tools-ec2-macos.md]]"
---

## 1. What
Đây là các phương pháp và câu lệnh để truyền tải dữ liệu (files/folders) từ máy tính cá nhân (local machine) lên máy chủ ảo Amazon EC2 thông qua giao thức SSH. Các công cụ phổ biến nhất bao gồm `scp` (Secure Copy), `rsync` và SFTP.

## 2. Why
Trong quá trình vận hành server, bạn thường xuyên phải:
- Upload mã nguồn ứng dụng (nếu không dùng CI/CD).
- Đẩy các file cấu hình (`.env`, `nginx.conf`).
- Chuyển các file dữ liệu lớn, SSL certificates lên server.
Việc nắm vững các lệnh này giúp bạn thao tác nhanh chóng và an toàn mà không cần cài đặt các phần mềm giao diện phức tạp.

## 3. Mental Model
Hãy tưởng tượng EC2 server là một **căn hộ ở xa**.
- SSH là chiếc **chìa khóa** (Key Pair) để mở cửa.
- `scp` giống như việc bạn tự tay mang một món đồ đến và đặt vào phòng.
- `rsync` giống như một **người vận chuyển thông minh**: Anh ta kiểm tra xem món đồ đó đã có ở căn hộ chưa, nếu có rồi và giống hệt thì anh ta không mang đi nữa, nếu chỉ khác một chút thì anh ta chỉ mang phần linh kiện thay thế đến (tiết kiệm sức lực/băng thông).

## 4. Where it fits
Local Machine -> **SSH Protocol (Port 22)** -> Security Group -> **EC2 Filesystem**.

## 5. When to use
- Khi cần upload nhanh 1-2 file cấu hình.
- Khi làm việc với các hệ thống không có giao diện đồ họa (headless servers).
- Khi muốn tự động hóa việc upload qua các script shell.

## 6. When NOT to use
- Triển khai ứng dụng quy mô lớn (nên dùng CI/CD như GitHub Actions, AWS CodeDeploy).
- Khi số lượng file quá lớn và thay đổi liên tục (nên dùng S3 làm trung gian hoặc EFS).
- Khi server không mở port 22 (SSH).

## 7. Trade-offs
| Method | Pros | Cons |
|------|------|------|
| **SCP** | Đơn giản, cài sẵn trên hầu hết các OS. | Không hỗ trợ tiếp tục (resume) nếu bị ngắt mạng, copy lại toàn bộ file dù chỉ đổi 1 bit. |
| **RSYNC** | Cực nhanh (chỉ gửi phần thay đổi), hỗ trợ resume, nén dữ liệu khi gửi. | Cú pháp phức tạp hơn, cần cài đặt trên cả máy gửi và máy nhận. |
| **SFTP** | Giao diện tương tác giống FTP nhưng bảo mật. | Chậm hơn cho việc truyền tải hàng loạt. |

## 1. How

### Sử dụng SCP (Phổ biến nhất)
Copy 1 file:
```bash
scp -i "key-pair.pem" local-file.txt ec2-user@ec2-11-22-33-44.compute-1.amazonaws.com:/home/ec2-user/
```

Copy cả thư mục (dùng `-r`):
```bash
scp -i "key-pair.pem" -r ./my-folder ec2-user@ec2-11-22-33-44.compute-1.amazonaws.com:/home/ec2-user/
```

### Sử dụng RSYNC (Khuyên dùng cho file lớn/thư mục)
```bash
rsync -avz -e "ssh -i key-pair.pem" ./local-folder/ ec2-user@ec2-11-22-33-44.compute-1.amazonaws.com:/home/ec2-user/remote-folder
```
- `-a`: Archive mode (giữ nguyên quyền, chủ sở hữu).
- `-v`: Verbose (hiển thị chi tiết).
- `-z`: Compress (nén dữ liệu khi truyền).

## 9. Production concerns
### Security
Luôn giới hạn IP được phép SSH trong Security Group để tránh kẻ xấu tấn công Brute-force khi bạn đang mở cổng để copy file.

### Permissions
File sau khi upload lên thường thuộc về user `ec2-user` hoặc `ubuntu`. Nếu ứng dụng của bạn chạy dưới user khác (như `www-data`), bạn cần chạy thêm lệnh `chown` trên server sau khi upload.

## 10. Common mistakes
- Mistake: Quên tham số `-i` để chỉ định file private key (`.pem`).
  Fix: Luôn đi kèm `-i path/to/key.pem`.

- Mistake: Sai đường dẫn đích trên server dẫn đến lỗi "Permission denied".
  Fix: Đảm bảo upload vào thư mục mà user có quyền ghi (thường là `/home/user_name/`).

## 11. Sample project
Viết một script `deploy.sh` đơn giản:
1. Nén thư mục mã nguồn thành `app.tar.gz`.
2. Dùng `scp` đẩy file nén lên EC2.
3. Dùng `ssh` để giải nén và restart service trên EC2.

## 12. Interview
### Core Q&A
1. Q: Làm thế nào để copy file từ EC2 về máy local?
   A: Chỉ cần đảo ngược vị trí trong lệnh `scp`: `scp -i key.pem user@remote:/path/to/file ./local-path/`.

2. Q: Tại sao dùng `rsync` lại tốt hơn `scp` cho các thư mục lớn?
   A: Vì `rsync` sử dụng thuật toán delta-transfer, nó chỉ gửi những phần khác biệt giữa file local và file remote, giúp tiết kiệm băng thông và thời gian đáng kể.

### Scenario
"Bạn đang upload một file 10GB lên EC2 bằng `scp` và mạng bị ngắt ở 90%. Bạn làm gì tiếp theo?"
-> Trả lời: Tôi sẽ chuyển sang dùng `rsync`. `rsync` sẽ kiểm tra phần dữ liệu đã có trên server và chỉ upload tiếp 10% còn lại thay vì phải bắt đầu lại từ đầu như `scp`.

## 13. References
- Linux Manual: `man scp`, `man rsync`.
- AWS Docs: [Transfer files to Linux instances](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/AccessingInstancesLinux.html#AccessingInstancesLinuxSCP)

## 14. Real-world Code
Sử dụng các công cụ GUI như **FileZilla** hoặc **Cyberduck** kết nối qua giao thức SFTP bằng file `.pem` để quản lý file trực quan hơn nếu không muốn dùng command line.

## 15. Community
- Stack Overflow: Tag [scp] [rsync] [amazon-ec2].
- Reddit: r/aws - thảo luận về các công cụ quản lý file server hiệu quả.
