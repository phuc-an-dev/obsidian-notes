---
created: 2026-05-06
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/others"
  - "#topic/http"
related:
  - "[[RESTful API.md]]"
---

## 1. What
Postman là một nền tảng API (API Platform) cho phép các lập trình viên xây dựng, kiểm thử, quản lý và tài liệu hóa các API. Nó cung cấp giao diện đồ họa (GUI) mạnh mẽ để thực hiện các HTTP requests (GET, POST, PUT, DELETE) và kiểm tra các phản hồi (Response) từ server một cách nhanh chóng.

## 2. Why
Trước khi có Postman, lập trình viên phải dùng các lệnh cURL phức tạp trong terminal hoặc viết code test tạm bợ để kiểm tra API. Điều này rất khó để quản lý các header, body phức tạp hoặc chia sẻ kết quả cho team. Postman ra đời để đơn giản hóa quy trình này, giúp tăng tốc độ phát triển và cải thiện sự phối hợp giữa Frontend và Backend.

## 3. Mental Model
Hãy tưởng tượng Postman giống như một **"Phòng thí nghiệm API"**.
- Bạn là nhà khoa học.
- Các API là các mẫu vật cần thí nghiệm.
- Postman cung cấp đầy đủ các dụng cụ (Header, Body, Auth, Scripts) để bạn đưa các kích thích (Request) vào mẫu vật và quan sát phản ứng (Response) của chúng trong một môi trường được kiểm soát.

## 4. Where it fits
Vị trí trong quy trình phát triển:
`Frontend Developer / Backend Developer -> Postman -> API Endpoint (Server)`

## 5. When to use
- Khi bắt đầu thiết kế và xây dựng API mới để kiểm tra tính đúng đắn của logic.
- Khi cần viết các bộ test tự động (Automated Testing) cho API.
- Khi cần viết tài liệu (Documentation) cho API để bàn giao cho các bên liên quan.
- Khi cần giả lập (Mock) các phản hồi của API khi backend chưa hoàn thiện.

## 6. When NOT to use
- Khi chỉ cần thực hiện các lệnh gọi API đơn giản và nhanh chóng một lần duy nhất (có thể dùng cURL hoặc các extension trình duyệt nhẹ hơn).
- Đối với các bài kiểm tra hiệu năng (Performance/Load Testing) ở quy mô cực lớn (nên dùng JMeter hoặc k6).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Giao diện trực quan, cực kỳ dễ sử dụng. | Chiếm dụng nhiều tài nguyên RAM (do viết trên Electron). |
| Hỗ trợ Scripts (JavaScript) mạnh mẽ cho kiểm thử. | Phiên bản Cloud đôi khi gặp vấn đề về bảo mật nếu lưu thông tin nhạy cảm. |
| Khả năng đồng bộ và làm việc nhóm tuyệt vời. | Một số tính năng nâng cao (Mock, Monitor) bị giới hạn ở bản miễn phí. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Insomnia | Nhẹ hơn, tập trung vào sự đơn giản và trải nghiệm người dùng. |
| Thunder Client | Extension trực tiếp trong VS Code, rất tiện lợi khi đang code. |
| Bruno | Open-source, lưu dữ liệu trực tiếp vào git của project (Git-friendly). |

## 9. How
Các tính năng cốt lõi cần nắm vững:

### Environment Variables
Dùng để quản lý các biến thay đổi theo môi trường (Dev, Staging, Prod):
`{{base_url}}/api/v1/users`

### Tests Script (JavaScript)
Viết test case ngay sau khi nhận phản hồi:
```javascript
pm.test("Status code is 200", function () {
    pm.response.to.have.status(200);
});

pm.test("Response body contains user_id", function () {
    var jsonData = pm.response.json();
    pm.expect(jsonData).to.have.property("user_id");
});
```

### Pre-request Scripts
Dùng để xử lý dữ liệu trước khi gửi request (ví dụ: tạo hash, lấy timestamp).

## 10. Production concerns
### Security
Tuyệt đối không lưu các thông tin nhạy cảm như Production API Keys, Passwords vào biến "Initial Value" (vì nó sẽ đồng bộ lên cloud của Postman). Luôn dùng "Current Value" cho các thông tin này.

### Monitoring
Postman hỗ trợ tính năng "Monitors" để tự động chạy các bộ test định kỳ và thông báo nếu API Production gặp sự cố.

## 11. Common mistakes
- Mistake: Hardcode URL vào từng request.
  Fix: Luôn sử dụng Environment Variables để dễ dàng chuyển đổi giữa các môi trường.

- Mistake: Không sử dụng tính năng "Collection" để phân nhóm API.
  Fix: Sắp xếp API theo module hoặc feature trong các Collections để dễ quản lý.

## 12. Sample project
Tạo một Collection "User Management":
1. Request "Login" để lấy Bearer Token.
2. Dùng Script để tự động lưu Token vào biến môi trường.
3. Các request sau (Get Profile, Update Info) sẽ tự động lấy Token đó để thực hiện Authentication.

## 13. Interview
### Core Q&A
1. Q: "Collection Runner" trong Postman dùng để làm gì?
   A: Dùng để chạy toàn bộ các request trong một Collection theo thứ tự, hỗ trợ việc kiểm thử hồi quy (Regression Testing) hoặc chạy test với nhiều bộ dữ liệu (Data-driven testing).

2. Q: Làm thế nào để chia sẻ API Documentation từ Postman?
   A: Sử dụng tính năng "Publish Docs". Postman sẽ tạo ra một URL công khai hoặc nội bộ chứa toàn bộ thông tin về endpoint, tham số và ví dụ response.

### Scenario
"Backend thay đổi cấu trúc JSON response làm hỏng Frontend. Làm sao Postman giúp bạn phát hiện sớm việc này?"
-> Trả lời: Tôi sẽ viết các "Schema Validation tests" trong Postman. Mỗi khi Backend deploy, tôi chạy Collection Runner. Nếu cấu trúc JSON trả về không khớp với Schema định nghĩa, các test case sẽ fail ngay lập tức, giúp phát hiện lỗi trước khi đến tay Frontend.

## 14. References
- Official Site: [Postman.com](https://www.postman.com/)
- Documentation: [Postman Learning Center](https://learning.postman.com/)

## 15. Real-world Code
Postman hỗ trợ xuất (Export) Collection dưới dạng file JSON. Nhiều dự án open-source đính kèm file này trong thư mục `docs/` để lập trình viên khác có thể import và test ngay.

## 16. Community
- Postman Community Forum
- Reddit: r/postman
- Stack Overflow: Tag [postman]
