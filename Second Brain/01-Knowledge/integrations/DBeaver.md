---
created: 2026-05-06
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/others"
  - "#topic/database"
related:
  - "[[railway-postgresql.md]]"
  - "[[TablePlus.md]]"
---

## 1. What
DBeaver là một công cụ quản trị cơ sở dữ liệu (Database Management Tool) mã nguồn mở, đa nền tảng và đa năng. Nó hỗ trợ hầu hết các hệ quản trị CSDL phổ biến hiện nay như MySQL, PostgreSQL, SQL Server, Oracle, MongoDB, Redis và thậm chí cả các tệp tin CSV.

## 2. Why
Lập trình viên thường phải làm việc với nhiều loại database khác nhau cùng lúc. Việc cài đặt mỗi loại một công cụ quản lý riêng (như pgAdmin cho Postgre, Workbench cho MySQL) rất tốn tài nguyên và khó làm quen. DBeaver giải quyết vấn đề này bằng một giao diện duy nhất "Tất cả trong một", cực kỳ mạnh mẽ và hoàn toàn miễn phí.

## 3. Mental Model
Hãy tưởng tượng DBeaver giống như một **"Con dao quân thụy Thụy Sĩ"** dành cho dữ liệu:
- Bạn có thể mở bất kỳ loại "hộp" dữ liệu nào bằng công cụ này.
- Dù đó là hộp SQL (Quan hệ) hay NoSQL (Phi cấu trúc), DBeaver đều có đúng "lưỡi dao" (Driver) để xử lý.
- Nó mạnh mẽ, bền bỉ và là vật bất ly thân của mọi "nhà thám hiểm" dữ liệu.

## 4. Where it fits
Vị trí trong quy trình:
`Developer PC -> DBeaver (JDBC Driver) -> Database Server (Local/Remote/Cloud)`

DBeaver sử dụng trình điều khiển JDBC để kết nối với các server, đảm bảo tính tương thích rộng rãi.

## 5. When to use
- Khi bạn cần một công cụ miễn phí nhưng có đầy đủ tính năng cao cấp cho công việc hàng ngày.
- Khi cần so sánh dữ liệu giữa hai database khác nhau (ví dụ từ MySQL sang PostgreSQL).
- Khi cần vẽ sơ đồ quan hệ thực thể (ER Diagram) từ một database có sẵn.
- Khi làm việc với các hệ quản trị CSDL ít phổ biến hoặc đặc thù.

## 6. When NOT to use
- Nếu bạn ưu tiên sự mượt mà, tối giản và tốc độ khởi động nhanh trên macOS (TablePlus sẽ tốt hơn).
- Khi bạn làm việc trong các môi trường doanh nghiệp yêu cầu hỗ trợ kỹ thuật 24/7 (có thể cân nhắc bản DBeaver Enterprise có phí).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Hoàn toàn miễn phí (bản Community) và mã nguồn mở. | Giao diện viết trên Java/Eclipse nên đôi khi cảm giác hơi nặng nề và "cũ kỹ". |
| Tính năng cực kỳ đồ sộ: ERD, Data Transfer, Mock Data. | Tốn nhiều RAM hơn các công cụ Native. |
| Hỗ trợ NoSQL (MongoDB, Cassandra) rất tốt ở bản trả phí. | Cú pháp gợi ý code (IntelliSense) đôi khi hơi chậm. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| TablePlus | Native, cực nhanh, đẹp, nhưng bản miễn phí giới hạn số tab/connection. |
| DataGrip | Cực mạnh về thông minh (JetBrains), nhưng tốn phí và rất nặng. |
| Navicat | Tiêu chuẩn ngành lâu đời, giao diện tốt, nhưng giá rất đắt. |

## 9. How
Các tính năng "quyền năng" trong DBeaver:

### Database Navigator
Quản lý cây thư mục các connection. Bạn có thể kéo thả để di chuyển bảng giữa các database.

### SQL Editor
Hỗ trợ viết truy vấn với khả năng tự động hoàn thành, định dạng code (Format SQL) và giải thích kế hoạch thực thi (Explain Plan) để tối ưu query.

### Data Viewer
Cho phép sửa dữ liệu trực tiếp trên bảng như một file Excel. Đặc biệt có tính năng **"Value Panel"** để xem các chuỗi JSON dài một cách dễ dàng.

### ER Diagrams
Chỉ cần chuột phải vào Database -> View Diagram. DBeaver sẽ tự động vẽ sơ đồ quan hệ giữa các bảng giúp bạn hiểu cấu trúc hệ thống cực nhanh.

## 10. Production concerns
### SSH Tunneling
Luôn sử dụng tính năng SSH Tunnel tích hợp trong DBeaver để kết nối tới database Production (thường nằm trong mạng nội bộ). Không bao giờ mở port DB (3306, 5432) ra public.

### Confirmation
Bật tính năng "Confirm data changes" và "Transaction mode = Manual" cho các connection Production để tránh việc vô tình chạy lệnh `DELETE` hoặc `UPDATE` nhầm mà không thể quay đầu.

## 11. Common mistakes
- Mistake: Để chế độ tự động commit (Auto-commit) khi đang thao tác trên database khách hàng.
- Mistake: Tải quá nhiều driver không cần thiết làm phình dung lượng ứng dụng.

## 12. Sample project
Sử dụng DBeaver để di chuyển dữ liệu:
1. Kết nối vào DB SQLite ở máy local.
2. Kết nối vào DB PostgreSQL trên Cloud (Railway/AWS).
3. Sử dụng tính năng "Export Data" từ SQLite trỏ thẳng đích đến là PostgreSQL. DBeaver sẽ tự tạo bảng và map dữ liệu cho bạn.

## 13. Interview
### Core Q&A
1. Q: DBeaver kết nối với Database thông qua cơ chế nào?
   A: Sử dụng JDBC (Java Database Connectivity) drivers. DBeaver sẽ tự động gợi ý tải driver phù hợp khi bạn tạo connection mới.

2. Q: Làm thế nào để xem câu lệnh SQL mà DBeaver thực hiện khi bạn sửa dữ liệu trên giao diện UI?
   A: Trước khi nhấn "Save", bạn nhấn nút "Script" ở thanh công cụ phía dưới, DBeaver sẽ hiển thị chính xác câu lệnh `UPDATE` hoặc `INSERT` sắp được chạy.

### Scenario
"Database Production của bạn bị chậm, bạn dùng DBeaver để làm gì?"
-> Trả lời: Tôi sẽ sử dụng công cụ "Execution Plan" (phím tắt Ctrl+Shift+E) để xem database đang quét dữ liệu như thế nào. Nếu thấy "Full Table Scan", tôi biết mình cần bổ sung Index cho các cột trong câu lệnh `WHERE`.

## 14. References
- Official Site: [dbeaver.io](https://dbeaver.io/)
- Wiki: [DBeaver Wiki on GitHub](https://github.com/dbeaver/dbeaver/wiki)

## 15. Real-world Code
DBeaver lưu trữ cấu hình connection trong file `.json`. Bạn có thể tìm thấy chúng trong thư mục cấu hình của user để thực hiện backup hoặc migrate sang máy mới.

## 16. Community
- GitHub: [dbeaver/dbeaver](https://github.com/dbeaver/dbeaver)
- Reddit: r/dbeaver.
