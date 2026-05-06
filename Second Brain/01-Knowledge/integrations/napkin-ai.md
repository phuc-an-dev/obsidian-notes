---
created: 2026-05-05
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/others"
  - "#topic/performance"
related:
  - "[[saas]]"
---

## 1. What
Napkin AI là một công cụ thiết kế được hỗ trợ bởi trí tuệ nhân tạo, tập trung vào việc biến văn bản (text) thành các sơ đồ, biểu đồ và hình ảnh minh họa chuyên nghiệp (visuals) một cách tức thì. Nó giúp người dùng không có kỹ năng thiết kế có thể tạo ra các nội dung trực quan để giải thích các ý tưởng phức tạp.

## 2. Why
Trước khi có Napkin AI:
- **Tốn thời gian**: Việc tự vẽ các sơ đồ trong PowerPoint, Canva hay Lucidchart mất rất nhiều công sức căn chỉnh.
- **Rào cản thiết kế**: Không phải ai cũng biết cách chọn màu sắc, icon hay bố cục sao cho chuyên nghiệp.
- **Mất tập trung**: Người viết phải liên tục chuyển đổi giữa việc tư duy nội dung và việc loay hoay với công cụ thiết kế.

## 3. Mental Model
Hãy tưởng tượng Napkin AI giống như một **"Họa sĩ phác thảo"** ngồi cạnh bạn. Bạn chỉ cần nói ý tưởng (nhập text), và người họa sĩ đó ngay lập tức vẽ ra một bản phác thảo sạch đẹp lên một tờ giấy ăn (napkin). Tuy nhiên, tờ giấy ăn này có thể biến thành file vector chuyên nghiệp để bạn đưa vào báo cáo chính thức.

## 4. Where it fits
Quy trình làm việc:
`Viết nội dung (Notion/Google Docs) -> Copy vào Napkin AI -> Chọn kiểu Visual -> Tùy chỉnh (Brand color/Icon) -> Export (PNG/SVG/PPTX)`

## 5. When to use
- Tạo sơ đồ quy trình (Flowchart) cho tài liệu hướng dẫn (Documentation).
- Thiết kế hình ảnh minh họa cho các bài đăng LinkedIn hoặc Blog để tăng tương tác.
- Tạo slide thuyết trình nhanh nhưng vẫn đảm bảo tính thẩm mỹ cao.
- Giải thích các hệ thống kiến trúc (System Architecture) ở mức độ high-level.

## 6. When NOT to use
- Cần các hình ảnh minh họa mang tính nghệ thuật cao hoặc đặc thù thương hiệu (Custom illustrations).
- Các sơ đồ kỹ thuật cực kỳ chi tiết và phức tạp (nên dùng Draw.io hoặc Miro).
- Khi không có kết nối internet (Napkin AI là công cụ chạy trên Web).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Tốc độ cực nhanh (vài giây cho một sơ đồ). | Khả năng sáng tạo bị giới hạn bởi các template của AI. |
| Xuất được file PPTX có thể chỉnh sửa từng element. | Đòi hỏi phải trả phí để bỏ watermark và dùng tính năng nâng cao. |
| Tự động chọn icon và màu sắc hài hòa. | Đôi khi AI hiểu sai ngữ cảnh của văn bản phức tạp. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Canva | Linh hoạt hơn nhưng tốn nhiều thao tác thủ công hơn. |
| Gamma.app | Tập trung vào việc tạo cả bản thuyết trình/website, Napkin AI tập trung vào visual riêng lẻ. |
| Lucidchart | Chuyên sâu về sơ đồ kỹ thuật, ít tính thẩm mỹ "tức thì" hơn. |

## 9. How
1. **Input**: Nhập đoạn văn bản hoặc dán một URL bài viết.
2. **Generate**: Click vào biểu tượng "Napkin" bên cạnh đoạn text muốn chuyển đổi.
3. **Select**: Chọn trong danh sách 30+ loại hình ảnh (Venn diagram, Cycle, Funnel, Matrix...).
4. **Customize**: Thay đổi font (700+ Google Fonts), bộ icon hoặc bảng màu thương hiệu.
5. **Export**: Tải về dưới định dạng PNG, PDF, SVG hoặc PPTX.

## 10. Production concerns
### Scaling
- Hỗ trợ Brand Kit: Cho phép lưu trữ bảng màu và font chữ của công ty để áp dụng đồng nhất cho mọi visual.

### Failure
- Nếu AI tạo ra sơ đồ không ưng ý, người dùng có thể dùng tính năng "Chat with AI" để yêu cầu sửa đổi cụ thể (ví dụ: "Thêm một bước vào quy trình này").

### Monitoring
- Quản lý credit: Các gói miễn phí thường giới hạn số lượng "Napkins" (lượt generate) mỗi tuần.

## 11. Common mistakes
- **Mistake**: Đưa quá nhiều văn bản vào một sơ đồ khiến nó bị rối và chữ quá nhỏ.
  **Fix**: Tóm tắt nội dung thành các từ khóa (bullet points) trước khi generate.

- **Mistake**: Giữ nguyên màu mặc định của AI mà không khớp với slide thuyết trình tổng thể.
  **Fix**: Sử dụng tính năng "Apply Brand Colors" để đồng bộ hóa.

## 12. Sample project
Thực hiện visual hóa quy trình "Onboarding một Software Engineer":
1. Nhập text: "Nhận máy tính -> Setup môi trường -> Đọc docs dự án -> Code fix bug đầu tiên -> Review code".
2. Yêu cầu AI tạo một "Horizontal Timeline".
3. Xuất file SVG để đưa vào Notion của team.

## 13. Interview
### Core Q&A
1. Q: Napkin AI có thể thay thế hoàn toàn một Designer không?
   A: Không. Nó thay thế các công việc đồ họa lặp đi lặp lại và đơn giản. Với các yêu cầu sáng tạo đột phá hoặc thiết kế UI/UX chuyên sâu, Designer vẫn là yếu tố quyết định.

2. Q: Tại sao tính năng export PPTX của Napkin AI lại quan trọng?
   A: Vì nó cho phép người dùng cuối (Manager/Client) có thể tự chỉnh sửa text hoặc màu sắc ngay trong PowerPoint mà không cần phải quay lại công cụ gốc.

### Scenario
**Tình huống**: Bạn có một buổi thuyết trình quan trọng vào sáng mai nhưng slide hiện tại toàn chữ. Bạn chỉ có 30 phút để chỉnh sửa.
**Giải quyết**: Sử dụng Napkin AI để quét qua các đoạn text chính, biến chúng thành các sơ đồ Venn hoặc Flowchart. Export toàn bộ sang PPTX và dán vào slide hiện tại. Việc này giúp slide trông chuyên nghiệp hơn hẳn chỉ trong 15-20 phút.

## 14. References
- Official Site: [napkin.ai](https://www.napkin.ai/)
- Documentation: [Napkin Help Center](https://help.napkin.ai/)

## 15. Real-world Code
N/A (Tool-based).

## 16. Community
- YouTube: Các video review về "AI tools for productivity".
- LinkedIn: Hashtag `#NapkinAI` thường có các mẫu thiết kế đẹp từ cộng đồng.
