---
created: 2026-05-04
tags:
  - "#type/tutorial"
  - "#status/draft"
  - "#lang/others"
  - "#topic/performance"
related:
  - "[[aws-cli-configuration]]"
---

## 1. What
**Cấu hình biến môi trường trên macOS** là quá trình thiết lập các cặp key-value (ví dụ: `JAVA_HOME=/path/to/jdk`) trong hệ thống để các ứng dụng và dòng lệnh có thể truy cập thông tin cấu hình toàn cục. Trên các bản macOS hiện đại, việc này chủ yếu được thực hiện thông qua shell **zsh**.

## 2. Why
Nhiều công cụ phát triển (như Java, Python, AWS CLI, Node.js) yêu cầu các biến môi trường để hoạt động đúng. Ví dụ, biến `PATH` giúp hệ điều hành biết nơi tìm các tệp thực thi khi bạn gõ một lệnh trong terminal. Nếu không cấu hình đúng, bạn sẽ thường xuyên gặp lỗi "command not found".

## 3. Mental Model
Hãy tưởng tượng biến môi trường như một danh sách **"Tên hiệu (Nicknames)"** mà bạn đặt cho các đồ vật trong nhà. Thay vì nói "Hãy lấy cho tôi cái tua vít nằm ở ngăn kéo thứ 3, tủ gỗ màu nâu, phòng kho", bạn chỉ cần nói "Lấy cho tôi **Cái Tua Vít (Variable Name)**". Hệ điều hành sẽ tra cứu trong sổ tay địa chỉ (Environment Config) để tìm đúng vị trí đó cho bạn.

## 4. Where it fits
Nó nằm ở tầng cấu hình người dùng của hệ điều hành:
`User Login -> Shell Startup (.zshrc) -> Load Environment Variables -> Command Execution`

## 5. When to use
- Khi cài đặt các SDK mới (Java, Go, Flutter).
- Khi cần lưu các API Key hoặc Secret (cho môi trường local) mà không muốn ghi trực tiếp vào code.
- Khi muốn tùy chỉnh giao diện terminal hoặc các bí danh (alias) cho lệnh dài.

## 6. When NOT to use
- Khi lưu trữ các bí mật cực kỳ nhạy cảm của production (nên dùng Secret Manager).
- Khi các cấu hình chỉ mang tính chất tạm thời trong một phiên làm việc (chỉ cần `export` trực tiếp trong terminal).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Quản lý tập trung các cấu hình phần mềm. | Dễ gây nhầm lẫn nếu cấu hình sai tệp (ví dụ `.bash_profile` thay vì `.zshrc`). |
| Tự động nạp mỗi khi mở Terminal. | Phải chạy lệnh `source` hoặc mở lại Terminal để có hiệu lực. |
| Code sạch hơn, tách biệt config và logic. | Có thể gây xung đột nếu nhiều phần mềm cùng sửa một biến (như `PATH`). |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| **/etc/paths** | Cấu hình cho toàn bộ hệ thống (system-wide), yêu cầu quyền root. |
| **GUI Apps (LaunchAgent)** | Dùng cho các ứng dụng có giao diện, phức tạp hơn. |
| **Direnv** | Tự động nạp biến môi trường khi bạn `cd` vào một thư mục cụ thể. |

## 9. How
Các bước cấu hình cho macOS (dùng Zsh):
1. Mở Terminal.
2. Mở tệp cấu hình: `nano ~/.zshrc` (hoặc `vi`, `code`).
3. Thêm biến môi trường vào cuối file:
   ```zsh
   # Định nghĩa biến mới
   export MY_API_KEY="secret_value_here"
   
   # Thêm một đường dẫn vào PATH
   export PATH="/usr/local/mysql/bin:$PATH"
   ```
4. Lưu file và thoát (Ctrl+O, Enter, Ctrl+X đối với nano).
5. Áp dụng thay đổi ngay lập tức: `source ~/.zshrc`.
6. Kiểm tra: `echo $MY_API_KEY`.

## 10. Production concerns
### Scaling
Sử dụng các tệp `.env` kết hợp với thư viện (như `dotenv`) để quản lý biến môi trường theo từng dự án cụ thể thay vì nhồi nhét tất cả vào `.zshrc` của hệ thống.

### Failure
Nếu file `.zshrc` bị lỗi cú pháp, bạn có thể không chạy được bất kỳ lệnh nào (kể cả `ls` hay `vi`). Cách cứu vãn: dùng đường dẫn tuyệt đối `/bin/vi ~/.zshrc` để sửa lại.

### Monitoring
Sử dụng lệnh `printenv` để liệt kê toàn bộ các biến đang có hiệu lực trong phiên làm việc hiện tại.

## 11. Common mistakes
- **Mistake**: Quên từ khóa `export` (biến sẽ chỉ có tác dụng trong shell hiện tại, không truyền xuống các tiến trình con).
  **Fix**: Luôn dùng `export NAME="VALUE"`.

- **Mistake**: Ghi đè biến `PATH` thay vì nối thêm: `export PATH="/new/path"`.
  **Fix**: Luôn nối thêm biến cũ: `export PATH="/new/path:$PATH"`.

## 12. Sample project
Cấu hình `JAVA_HOME` cho macOS:
1. Tìm đường dẫn JDK: `/usr/libexec/java_home`.
2. Thêm vào `.zshrc`: `export JAVA_HOME=$(/usr/libexec/java_home)`.
3. Thêm vào PATH: `export PATH="$JAVA_HOME/bin:$PATH"`.

## 13. Interview
### Core Q&A
1. **Q**: Sự khác biệt giữa `.zshrc` và `.zprofile` là gì?
   **A**: `.zprofile` chạy khi login (thường là 1 lần), `.zshrc` chạy mỗi khi bạn mở một cửa sổ terminal mới (interactive shell). Thông thường ta dùng `.zshrc`.
2. **Q**: Làm sao để xóa một biến môi trường đang tồn tại?
   **A**: Dùng lệnh `unset VARIABLE_NAME`.
3. **Q**: Tại sao cần nối thêm `$PATH` vào cuối lệnh export PATH?
   **A**: Để giữ lại các đường dẫn cũ của hệ thống (như `/bin`, `/usr/bin`), nếu không bạn sẽ không thể chạy được các lệnh cơ bản.

### Scenario
**Tình huống**: Bạn đã thêm biến vào `.zshrc` nhưng khi `echo` ra thì không thấy gì. Tại sao?
**Trả lời**: Có 2 khả năng: 1. Tôi chưa chạy lệnh `source ~/.zshrc` để nạp lại cấu hình. 2. Tôi đang dùng một shell khác (như bash) chứ không phải zsh.

## 14. References
- Zsh Guide: [Zsh Startup Files](https://zsh.sourceforge.io/Intro/intro_3.html)

## 15. Real-world Code
- Các file setup môi trường chuyên nghiệp thường có đoạn: `[[ -s "$HOME/.sdkman/bin/sdkman-init.sh" ]] && source "$HOME/.sdkman/bin/sdkman-init.sh"`

## 16. Community
- Reddit: r/macsysadmin
- Stack Overflow: Tag [macos], [environment-variables]
