---
created: 2026-05-06
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/others"
  - "#topic/database"
related:
  - "[[DBeaver.md]]"
  - "[[railway-postgresql.md]]"
---

## 1. What
TablePlus là một công cụ quản lý cơ sở dữ liệu (Database GUI Tool) hiện đại, được xây dựng theo hướng native (thuần thục cho từng OS). Nó nổi tiếng với giao diện sạch sẽ, tốc độ cực nhanh và trải nghiệm người dùng (UX) tối giản nhưng hiệu quả, hỗ trợ nhiều loại CSDL như MySQL, PostgreSQL, Redis, Cassandra, v.v.

## 2. Why
Các công cụ quản trị DB cũ thường rất nặng nề và phức tạp. TablePlus ra đời để mang lại một hơi thở mới:
- **Tốc độ**: Vì là ứng dụng Native (không dùng Java/Electron), nó khởi động và phản hồi gần như tức thì.
- **An toàn**: Mọi thay đổi dữ liệu đều không được áp dụng ngay mà được giữ ở chế độ "Pending", giúp bạn kiểm tra lại trước khi nhấn Save.
- **Thẩm mỹ**: Giao diện được thiết kế tỉ mỉ, giúp lập trình viên không bị mệt mỏi khi phải nhìn vào dữ liệu hàng giờ liền.

## 3. Mental Model
Hãy tưởng tượng TablePlus giống như một **"Chiếc xe thể thao hạng sang"**:
- Nó không có quá nhiều nút bấm rắc rối như xe tải (DBeaver).
- Nhưng mọi nút bấm đều nằm đúng chỗ, mượt mà và cực kỳ nhanh.
- Bạn ngồi vào là muốn lái ngay, và việc "lái" qua các bảng dữ liệu trở nên thú vị hơn bao giờ hết.

## 4. Where it fits
Vị trí trong hệ thống:
`Lập trình viên (macOS/Windows) -> TablePlus (Native App) -> SSH Tunnel -> Database Server`

## 5. When to use
- Khi bạn dùng macOS hoặc Windows và muốn một trải nghiệm làm việc với database mượt mà nhất.
- Khi bạn cần thao tác nhanh trên nhiều connection cùng lúc nhờ tính năng Workspace.
- Khi cần một công cụ bảo mật tốt với tính năng SSH Tunneling dễ cấu hình.

## 6. When NOT to use
- Nếu bạn cần một công cụ hoàn toàn miễn phí không giới hạn (Bản miễn phí của TablePlus giới hạn 2 connection/2 workspace/2 tabs).
- Khi cần các tính năng mô hình hóa dữ liệu (Data Modeling) cực sâu mà DBeaver làm tốt hơn.
- Trên hệ điều hành Linux (bản Linux của TablePlus vẫn đang trong giai đoạn hoàn thiện, chưa tốt bằng các bản khác).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Hiệu năng Native cực cao, chiếm rất ít RAM. | Mô hình Freemium (giới hạn tính năng ở bản free). |
| Tính năng "Safe Mode" và "Pending Changes" cực kỳ an toàn. | Bản Linux còn sơ khai. |
| Hỗ trợ phím tắt thông minh kiểu Sublime Text/VS Code. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| DBeaver | Miễn phí hoàn toàn, tính năng đồ sộ hơn nhưng nặng nề hơn. |
| DataGrip | Thông minh hơn về SQL, nhưng tốn phí và rất nặng (Java-based). |
| Sequel Ace | Miễn phí, chuyên dụng cho MySQL trên macOS. |

## 9. How
Các tính năng làm nên tên tuổi của TablePlus:

### Pending Changes
Khi bạn sửa dữ liệu trên bảng, ô đó sẽ đổi màu vàng. Dữ liệu chưa được lưu vào DB cho đến khi bạn nhấn `Cmd + S`. Bạn có thể xem lại list các câu lệnh sắp chạy.

### Advanced Filters
Sử dụng các bộ lọc nhanh (Quick Filter) ở cuối bảng để tìm kiếm dữ liệu mà không cần viết lệnh `SELECT ... WHERE`.

### Plugin Hệ sinh thái
Hỗ trợ cài thêm plugin để mở rộng tính năng, ví dụ: Plugin tạo mã nguồn cho các ngôn ngữ (Go, PHP, Java) từ cấu hình bảng.

### Dark Mode & Themes
Cung cấp nhiều giao diện đẹp mắt, phù hợp với sở thích cá nhân của lập trình viên.

## 10. Production concerns
### Workspace Management
Tách biệt các kết nối Production vào một Workspace riêng và đặt màu sắc (Color tag) là **Màu Đỏ** để luôn nhắc nhở bản thân đang thao tác trên dữ liệu thật.

### Encryption
TablePlus sử dụng các thư viện TLS/SSL và SSH chuẩn nhất để đảm bảo dữ liệu trên đường truyền được bảo vệ tuyệt đối.

## 11. Common mistakes
- Mistake: Sử dụng bản "thuốc" (crack) không rõ nguồn gốc cho một công cụ quản lý dữ liệu nhạy cảm.
- Mistake: Quên nhấn Save khi đã sửa xong rất nhiều dữ liệu (do thói quen từ các tool auto-save).

## 12. Sample project
Tối ưu quy trình làm việc:
1. Tạo một connection tới Postgres local.
2. Sử dụng tính năng "Import SQL" để nạp file dump 1GB.
3. Sử dụng `Cmd + P` để nhảy nhanh tới bảng `users` và dùng Quick Filter để tìm user có email cụ thể.

## 13. Interview
### Core Q&A
1. Q: Tại sao TablePlus lại nhanh hơn các đối thủ?
   A: Vì nó là ứng dụng Native. Nó được viết bằng ngôn ngữ gốc của OS (Swift cho macOS, C++ cho Windows) thay vì chạy trên máy ảo Java hoặc Web-view.

2. Q: Tính năng "Pending Changes" giải quyết vấn đề gì?
   A: Giảm thiểu rủi ro con người. Nó cho phép bạn review lại tất cả các thay đổi dưới dạng mã SQL trước khi thực hiện commit cuối cùng vào database.

### Scenario
"Bạn cần copy dữ liệu từ một hàng (row) sang một hàng mới nhưng chỉ đổi ID. TablePlus giúp gì?"
-> Trả lời: Tôi sử dụng tính năng **"Duplicate"** (Cmd + D). TablePlus sẽ tạo một hàng mới giống hệt, tôi chỉ cần sửa ID và nhấn Save. Cực kỳ nhanh và tránh sai sót khi gõ lại.

## 14. References
- Official Website: [tableplus.com](https://tableplus.com/)
- Blog: [TablePlus Blog](https://tableplus.com/blog)

## 15. Real-world Code
TablePlus hỗ trợ xuất dữ liệu ra nhiều định dạng: SQL, CSV, JSON, Markdown. Tính năng "Copy as Markdown" rất hữu ích khi bạn muốn đưa ví dụ dữ liệu vào trong các tài liệu note này.

## 16. Community
- Twitter: @TablePlus.
- GitHub: [TablePlus/TablePlus-Queries](https://github.com/TablePlus/TablePlus-Queries) (nơi chia sẻ các query hữu ích).
