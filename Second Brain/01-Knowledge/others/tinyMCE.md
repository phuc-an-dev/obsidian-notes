---
created: 2026-04-24
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/javascript"
  - "#topic/editor"
related:
  - "[[]]"
---

## 1. What
TinyMCE là một thư viện JavaScript WYSIWYG editor giúp chuyển đổi các thẻ `<textarea>` hoặc phần tử HTML thành trình soạn thảo văn bản phong phú. Nó cho phép người dùng định dạng nội dung trực quan và xuất ra dữ liệu dạng HTML sạch.

## 2. Why
Trước khi có các trình soạn thảo như TinyMCE, việc cho phép người dùng nhập nội dung có định dạng (bold, list, table) trên web là cực kỳ khó khăn vì phải tự quản lý DOM, xử lý sự kiện bôi đen và sinh mã HTML thủ công. TinyMCE ra đời để trừu tượng hóa các tác vụ đó, cung cấp giao diện chuẩn hóa và an toàn.

## 3. Mental Model
Hãy tưởng tượng TinyMCE là một lớp màng ngăn cách giữa người dùng (thích dùng chuột để bôi đen, nhấn nút in đậm) và trình duyệt (chỉ hiểu các thẻ HTML như `<b>`, `<ul>`). TinyMCE nhận các lệnh hành động từ người dùng và tự động dịch chúng thành các cấu trúc HTML tương ứng một cách ổn định.

## 4. Where it fits
Người dùng -> TinyMCE Editor -> HTML/Content -> Sanitizer -> Database -> Rendering Engine (Front-end).

## 5. When to use
- Khi hệ thống cần tính năng CMS (Content Management System).
- Khi yêu cầu người dùng soạn thảo nội dung có định dạng phức tạp như bài viết, email, hoặc tài liệu kỹ thuật.
- Khi cần sự ổn định và hỗ trợ plugin chuyên sâu (như kéo thả ảnh, quản lý bảng).

## 6. When NOT to use
- Khi chỉ cần nhập liệu văn bản thuần túy (dùng `<textarea>` hoặc `<input>` là đủ).
- Khi cần hiệu năng cực cao và dung lượng cực nhẹ (TinyMCE khá nặng do hệ thống plugin phong phú).
- Khi sử dụng các giải pháp như Markdown editor (như Toast UI) để giảm thiểu rủi ro bảo mật XSS.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Hỗ trợ plugin rất mạnh mẽ | Dung lượng thư viện lớn |
| Giao diện người dùng chuyên nghiệp | Khó tùy biến sâu về kiến trúc lõi |
| Cộng đồng lớn, ổn định | Cần config bảo mật cẩn thận |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| CKEditor | Đối thủ cạnh tranh trực tiếp, mạnh mẽ và phổ biến tương đương. |
| Quill.js | Nhẹ hơn, sử dụng Delta format thay vì HTML trực tiếp. |
| TipTap | Hiện đại hơn, xây dựng trên Prosemirror, hỗ trợ React tốt. |

## 9. How
```javascript
import tinymce from 'tinymce/tinymce';
import 'tinymce/themes/silver/theme';
import 'tinymce/icons/default/icons';

tinymce.init({
  selector: '#editor',
  plugins: 'link image table',
  toolbar: 'undo redo | bold italic | link image | table'
});
```

## 10. Production concerns
### Scaling
Nên sử dụng CDN để load tài nguyên của TinyMCE nhằm giảm tải cho server ứng dụng.

### Failure
Cần fallback về textarea mặc định nếu script load thất bại để không làm gián đoạn trải nghiệm người dùng.

### Monitoring
Theo dõi các lỗi trong quá trình render (thường do xung đột CSS) qua các công cụ logging front-end như Sentry.

## 11. Common mistakes
- Mistake: Lưu trực tiếp nội dung từ editor vào DB mà không qua bộ lọc.
  Fix: Sử dụng các thư viện như DOMPurify để làm sạch HTML trước khi lưu.

- Mistake: Load toàn bộ plugin mặc định gây chậm trang.
  Fix: Chỉ liệt kê chính xác các plugin cần dùng trong cấu hình init.

## 12. Sample project
Tạo một trang quản trị bài viết có tích hợp TinyMCE, yêu cầu người dùng upload ảnh, sau đó làm sạch HTML trước khi lưu vào cơ sở dữ liệu giả lập.

## 13. Interview
### Core Q&A
1. Q: Làm thế nào để xử lý bảo mật với TinyMCE?
   A: Luôn coi nội dung từ editor là dữ liệu không an toàn. Phải sử dụng bộ lọc HTML (Sanitizer) ở server-side hoặc client-side (DOMPurify) để loại bỏ các script tag hoặc attribute nguy hiểm (như onclick).

### Scenario
1. Q: Nếu khách hàng yêu cầu tích hợp trình soạn thảo vào ứng dụng React, bạn chọn giải pháp nào?
   A: Tôi sẽ dùng `@tinymce/tinymce-react` vì nó cung cấp wrapper chuẩn cho vòng đời của React, quản lý việc load script và unmount editor sạch sẽ hơn tự nhúng DOM.

## 14. References
- Official Docs: https://www.tiny.cloud/docs/
- GitHub Repo: https://github.com/tinymce/tinymce
- Spec / RFC: HTML5 standards
- Changelog: https://www.tiny.cloud/docs/tinymce/6/changelog/

## 15. Real-world Code
Kiểm tra các dashboard opensource như Strapi hoặc các CMS viết bằng React thường sử dụng TinyMCE hoặc CKEditor làm mặc định.

## 16. Community
- Reddit: r/javascript
- Stack Overflow: [tinymce] tag
- Blog: Tiny Cloud Blog
- Talk: Các talk liên quan đến "Web Content Editing" trên YouTube.
