---
created: 2026-05-04
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/performance"
related:
  - "[[ec2]]"
---

## 1. What
`chmod 400` là một lệnh trong hệ điều hành Linux/Unix dùng để thay đổi quyền truy cập của một tệp tin. Cụ thể, nó thiết lập quyền **chỉ đọc (read-only)** cho chủ sở hữu (owner) và **không có quyền gì** cho nhóm (group) cũng như những người dùng khác (others).

## 2. Why
Trong bảo mật hệ thống, nguyên tắc "quyền hạn tối thiểu" (least privilege) là cực kỳ quan trọng. Có những tệp tin chứa thông tin nhạy cảm (như private keys) mà nếu để các user khác đọc được hoặc chính chủ sở hữu lỡ tay ghi đè/xóa mất thì sẽ gây hậu quả nghiêm trọng. `chmod 400` đảm bảo tệp tin được "đóng băng" ở trạng thái an toàn nhất.

## 3. Mental Model
Hãy tưởng tượng `chmod 400` như một cái **"Nhật ký có khóa"**. 
- Bạn là người giữ chìa khóa duy nhất (owner).
- Bạn có thể mở ra đọc nội dung bên trong (read).
- Tuy nhiên, bạn đã cố ý dán kín các trang lại để chính bạn cũng không thể viết thêm vào (no write).
- Những người khác trong nhà (group/others) thậm chí còn không được phép nhìn thấy bìa cuốn sách đó.

## 4. Where it fits
Nó là một phần của hệ thống quản lý quyền hạn (Filesystem Permissions):
`User -> Shell -> chmod 400 -> Linux Kernel -> File Metadata`

## 5. When to use
- Khi làm việc với các tệp SSH Private Key (ví dụ: `id_rsa`, `key.pem`).
- Khi lưu trữ các tệp cấu hình chứa secret (như mật khẩu DB) mà không cần thay đổi thường xuyên.
- Khi tạo các bản sao lưu (backups) mà bạn muốn đảm bảo không bị chỉnh sửa nhầm.

## 6. When NOT to use
- Không dùng cho thư mục (directories) vì quyền "x" (execute) là bắt buộc để "vào" thư mục. Nếu dùng 400 cho thư mục, bạn sẽ không thể truy cập nội dung bên trong.
- Không dùng cho các tệp tin mà ứng dụng cần ghi dữ liệu liên tục (như log files).
- Không dùng cho các script cần được thực thi (cần quyền 500 hoặc 700).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Bảo mật tối đa cho dữ liệu nhạy cảm. | Gây phiền hà khi muốn chỉnh sửa (phải chmod lại). |
| Ngăn chặn việc vô tình xóa hoặc ghi đè file. | Dễ gây lỗi "Permission denied" cho các tiến trình cần ghi. |
| Đáp ứng tiêu chuẩn bảo mật của các công cụ như SSH. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `chmod 600` | Chủ sở hữu có thể đọc và ghi (phổ biến cho cấu hình cá nhân). |
| `chmod 440` | Cho phép cả nhóm (group) được đọc dữ liệu. |
| `chattr +i` | (Immutable bit) Khóa file hoàn toàn, ngay cả root cũng không xóa được nếu không gỡ bit. |

## 9. How
```bash
# Thiết lập quyền cho file key của AWS EC2
chmod 400 my-key.pem

# Kiểm tra lại quyền hạn
ls -l my-key.pem
# Kết quả: -r-------- 1 user user ... my-key.pem

# Giải thích số 400:
# 4 (Owner): Read (4) + Write (0) + Execute (0) = 4
# 0 (Group): Read (0) + Write (0) + Execute (0) = 0
# 0 (Others): Read (0) + Write (0) + Execute (0) = 0
```

## 10. Production concerns
### Scaling
Trong các hệ thống tự động hóa (CI/CD), việc quên `chmod 400` cho các key có thể làm gãy luồng deployment vì SSH sẽ từ chối kết nối nếu key quá "hở" (too open).

### Failure
Nếu tệp tin được sở hữu bởi `root` nhưng bạn chạy ứng dụng bằng user thường, `chmod 400` sẽ khiến ứng dụng không thể đọc được file đó.

### Monitoring
Sử dụng các công cụ audit hệ thống để phát hiện các file nhạy cảm có quyền quá lỏng lẻo (ví dụ tìm các file có quyền 777).

## 11. Common mistakes
- **Mistake**: Chạy `chmod 400` trên thư mục.
  **Fix**: Luôn dùng `chmod 500` hoặc `700` cho thư mục nếu muốn hạn chế quyền, để vẫn có thể `ls` hoặc `cd` vào được.

- **Mistake**: Quên `sudo` khi file thuộc sở hữu của `root`.
  **Fix**: Dùng `sudo chmod 400 <file>` để thực hiện thay đổi.

## 12. Sample project
Tạo một script bash tự động khởi tạo một server mới. Script này sẽ tải private key từ một kho bí mật, lưu xuống đĩa và ngay lập tức thực hiện `chmod 400` để đảm bảo lệnh `ssh` tiếp theo không bị lỗi bảo mật.

## 13. Interview
### Core Q&A
1. **Q**: Số 4 trong `400` đại diện cho điều gì?
   **A**: Trong hệ bát phân (octal), 4 đại diện cho quyền Read (đọc). 2 là Write, 1 là Execute.
2. **Q**: Tại sao lệnh `ssh` lại yêu cầu key phải có quyền `400` hoặc `600`?
   **A**: Để đảm bảo rằng không ai khác ngoài chủ sở hữu có thể đọc được private key, tránh rò rỉ thông tin danh tính.
3. **Q**: Làm thế nào để cấp quyền cho phép bạn vừa đọc vừa ghi nhưng người khác không làm được gì?
   **A**: Dùng `chmod 600`.

### Scenario
**Tình huống**: Bạn thực hiện `chmod 400` cho một script `.sh` và sau đó cố gắng chạy nó bằng lệnh `./script.sh` nhưng thất bại. Tại sao?
**Trả lời**: Vì `chmod 400` chỉ cấp quyền đọc. Để chạy một script trực tiếp, bạn cần quyền thực thi (Execute - số 1). Tôi nên dùng `chmod 500` hoặc chạy bằng cách gọi trình thông dịch: `bash script.sh`.

## 14. References
- Man page: `man chmod`
- Linux Documentation: [File Permissions](https://www.kernel.org/doc/html/latest/admin-guide/index.html)

## 15. Real-world Code
- Lệnh `ssh-keygen` thường tạo file với quyền mặc định là 600, nhưng các dịch vụ như AWS khuyên dùng 400 cho các tệp `.pem` tải về.

## 16. Community
- Stack Overflow: Tag [chmod], [linux-permissions]
- Reddit: r/linuxadmin
