---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/others"
  - "#topic/linux"
related:
  - "[[Common Ubuntu Commands.md]]"
---

## 1. What
Nano Editor là một trình soạn thảo văn bản dựa trên giao diện dòng lệnh (Terminal-based text editor) phổ biến trên hệ điều hành Ubuntu và các bản phân phối Linux khác. Nó được thiết kế để đơn giản, dễ sử dụng cho cả người mới bắt đầu và các chuyên gia khi cần chỉnh sửa nhanh các tệp tin cấu hình.

## 2. Why
Trước khi có các trình soạn thảo hiện đại, việc chỉnh sửa file trên server từ xa qua SSH thường rất khó khăn vì các trình soạn thảo như Vi/Vim có độ dốc học tập (learning curve) cao. Nano ra đời để cung cấp một giải pháp thay thế trực quan hơn, với các phím tắt hướng dẫn luôn hiển thị ở dưới cùng màn hình, giúp lập trình viên không cần nhớ quá nhiều lệnh phức tạp.

## 3. Mental Model
Hãy tưởng tượng Nano giống như một chiếc **"Sổ tay bỏ túi"** trong Terminal.
- Nó không có nhiều tính năng hào nhoáng như Word hay VS Code.
- Nó luôn nằm sẵn trong túi của bạn (hệ thống).
- Bạn chỉ cần mở ra, viết nhanh vài dòng rồi cất đi (lưu lại).
- Nó đơn giản đến mức ai cũng có thể mở và dùng được ngay mà không cần đọc hướng dẫn sử dụng dày cộp.

## 4. Where it fits
Vị trí trong quy trình làm việc:
`User -> Terminal -> SSH -> Nano Editor -> Configuration Files (.conf, .yaml, .txt)`

## 5. When to use
- Khi cần chỉnh sửa nhanh các file cấu hình hệ thống (ví dụ: `/etc/nginx/nginx.conf`).
- Khi viết các nội dung ngắn trong terminal như git commit message.
- Khi làm việc trên các server từ xa mà bạn không muốn hoặc không thể cài đặt các trình soạn thảo phức tạp hơn.

## 6. When NOT to use
- Khi bạn cần viết code cho một dự án lớn (nên dùng VS Code, IntelliJ hoặc Vim với các plugin hỗ trợ).
- Khi cần xử lý các file văn bản cực lớn (Vim hoặc Emacs xử lý bộ nhớ tốt hơn trong trường hợp này).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Cực kỳ dễ học và dễ sử dụng. | Thiếu các tính năng nâng cao (như split panes, macro). |
| Luôn có sẵn hướng dẫn phím tắt ở cuối màn hình. | Không hỗ trợ nhiều plugin mạnh mẽ như Vim. |
| Nhẹ và khởi động gần như ngay lập tức. | Khả năng tùy biến giao diện hạn chế. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Vim | Mạnh mẽ nhất, nhưng cực kỳ khó học đối với người mới. |
| Emacs | Một hệ điều hành bên trong trình soạn thảo, cực kỳ linh hoạt nhưng nặng nề. |
| Micro | Hiện đại hơn Nano, hỗ trợ dùng chuột và phím tắt giống Windows (Ctrl+C, Ctrl+V). |

## 9. How
Các phím tắt cơ bản trong Nano (Ký hiệu `^` tương đương phím `Ctrl`):

```bash
# Mở hoặc tạo một file
nano filename.txt

# Lưu file (Write Out)
Ctrl + O (Sau đó nhấn Enter để xác nhận)

# Thoát khỏi Nano
Ctrl + X

# Tìm kiếm văn bản
Ctrl + W

# Cắt một dòng (Cut text)
Ctrl + K

# Dán một dòng (Uncut text)
Ctrl + U

# Đi tới một dòng cụ thể
Ctrl + _ (Sau đó nhập số dòng)
```

## 10. Production concerns
### Permissions
Khi chỉnh sửa file hệ thống trên Production, luôn cần dùng `sudo nano` để có quyền ghi.

### Backups
Nano có tính năng tự động tạo file backup. Bạn có thể dùng tham số `-B` để tạo bản sao lưu của file trước khi chỉnh sửa: `nano -B config.conf`.

## 11. Common mistakes
- Mistake: Không biết cách thoát khỏi Nano.
  Fix: Nhìn xuống dưới cùng màn hình, lệnh thoát là `^X` (Ctrl + X). Nếu có thay đổi chưa lưu, nó sẽ hỏi bạn có muốn lưu không.

- Mistake: Quên dùng `sudo` khi sửa file quan trọng.
  Fix: Nếu đã lỡ sửa rất nhiều mà không có quyền lưu, bạn có thể lưu file ra một tên tạm khác, sau đó dùng lệnh `sudo mv` để đè lên file gốc.

## 12. Sample project
Sử dụng Nano để tạo một file script đơn giản:
1. Gõ `nano hello.sh`.
2. Viết nội dung: `echo "Hello Ubuntu 24.04"`.
3. Lưu lại bằng `Ctrl + O`.
4. Thoát bằng `Ctrl + X`.
5. Cấp quyền thực thi: `chmod +x hello.sh`.
6. Chạy thử: `./hello.sh`.

## 13. Interview
### Core Q&A
1. Q: Làm thế nào để bật tính năng hiển thị số dòng trong Nano?
   A: Bạn có thể dùng tham số `-l` khi mở file: `nano -l filename`. Hoặc cấu hình vĩnh viễn trong file `.nanorc`.

2. Q: Ý nghĩa của các ký hiệu `^G`, `^O`, `^X` ở phía dưới màn hình là gì?
   A: Ký hiệu `^` đại diện cho phím `Ctrl`. `^G` là Get Help, `^O` là Write Out (Lưu), `^X` là Exit (Thoát).

### Scenario
"Bạn đang sửa một file cấu hình quan trọng trên Production bằng Nano và đột nhiên mất kết nối internet. Dữ liệu của bạn có bị mất không?"
-> Trả lời: Nano thường tạo một file tạm (swap file) để lưu trạng thái. Tuy nhiên, nếu chưa nhấn `Ctrl+O` để lưu chính thức, các thay đổi gần nhất có thể không được ghi vào file gốc. Khi kết nối lại, bạn nên kiểm tra xem có file nào kết thúc bằng `.save` không.

## 14. References
- Man page: `man nano`
- Official Website: [nano-editor.org](https://www.nano-editor.org/)

## 15. Real-world Code
Tùy biến Nano qua file `~/.nanorc` để hỗ trợ syntax highlighting:
```text
set linenumbers
set softwrap
set tabsize 4
include "/usr/share/nano/*.nanorc"
```

## 16. Community
- GNU Nano mailing list.
- Stack Overflow: Tag [nano].
- Reddit: r/linux.
