---
created: 2026-05-07
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/others"
  - "#topic/security"
related:
  - "[[SSL and TLS.md]]"
---

## 1. What
CSRF (Cross-Site Request Forgery) là một lỗ hổng bảo mật web cho phép kẻ tấn công lừa trình duyệt của người dùng thực hiện một hành động không mong muốn trên một website khác mà người dùng đó đã xác thực (đã đăng nhập).

## 2. Why
Trình duyệt có cơ chế tự động gửi kèm các cookie xác thực (như Session ID) mỗi khi gửi yêu cầu đến server đích. Kẻ tấn công lợi dụng đặc điểm này để "mượn danh" người dùng. Nếu không có biện pháp phòng vệ, kẻ tấn công có thể thay đổi mật khẩu, chuyển tiền, hoặc thay đổi email của nạn nhân mà nạn nhân không hề hay biết.

## 3. Mental Model
Hãy tưởng tượng bạn có một **con dấu cá nhân** (Session Cookie) luôn được để ở quầy giao dịch ngân hàng.
- Bạn đang đứng ở quầy ngân hàng (Website A) để giao dịch.
- Cùng lúc đó, bạn nhận được một tờ rơi (Website B của kẻ tấn công) và mở ra xem.
- Tờ rơi này có "ma thuật": Khi bạn chạm vào một hình ảnh, nó sẽ điều khiển tay bạn ký một lệnh chuyển tiền và đóng con dấu cá nhân của bạn vào đó.
- Ngân hàng thấy con dấu thật của bạn nên thực hiện lệnh ngay lập tức mà không biết rằng bạn bị điều khiển.

## 4. Where it fits
Attacker Site -> Malicious Request (GET/POST) -> User Browser (with Cookies) -> Target Site Server (Vulnerable).

## 5. When to use
N/A (Đây là một lỗ hổng, không phải tính năng). Tuy nhiên, các kỹ thuật phòng chống CSRF phải được sử dụng cho tất cả các State-changing requests (POST, PUT, DELETE, PATCH).

## 6. When NOT to use
N/A. Tuy nhiên, các yêu cầu chỉ đọc (GET) không cần thiết phải áp dụng chống CSRF (theo quy chuẩn RESTful, GET không được làm thay đổi dữ liệu).

## 7. Trade-offs
| Pros (của việc phòng chống) | Cons (của việc phòng chống) |
|------|------|
| Bảo vệ người dùng khỏi các hành động trái phép. | Tăng độ phức tạp của code (quản lý token). |
| Ngăn chặn tổn thất tài chính và dữ liệu. | Có thể gây lỗi nếu người dùng mở nhiều tab hoặc session hết hạn. |
| Đáp ứng các tiêu chuẩn bảo mật (OWASP Top 10). | N/A |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Anti-CSRF Token | Phổ biến nhất, an toàn nhất, yêu cầu server kiểm tra token duy nhất cho mỗi form. |
| SameSite Cookie Attribute | Đơn giản, cấu hình ở mức trình duyệt, nhưng không hỗ trợ các trình duyệt quá cũ. |
| Double Submit Cookie | Không yêu cầu lưu state ở server, nhưng kém an toàn hơn token truyền thống. |
| Custom Request Headers | (Ví dụ: `X-Requested-With`) Hiệu quả cho các API gọi qua AJAX. |

## 9. How
Ví dụ về cơ chế Anti-CSRF Token trong Spring Security:
1. Server tạo một chuỗi ngẫu nhiên (Token) và gửi về cho client (thường ẩn trong form).
2. Khi client gửi request POST, phải gửi kèm token này.
3. Server so sánh token nhận được với token đang lưu trong session.

```html
<!-- Form trong HTML -->
<form action="/transfer-money" method="POST">
    <input type="hidden" name="_csrf" value="a1b2c3d4e5f6g7h8" />
    <input type="number" name="amount" />
    <button type="submit">Chuyển tiền</button>
</form>
```

Cấu hình SameSite cho Cookie (Set-Cookie header):
```text
Set-Cookie: session_id=abc123; SameSite=Strict; Secure; HttpOnly
```

## 10. Production concerns
### Scaling
Nếu dùng Anti-CSRF Token lưu trong session, bạn cần đảm bảo Session Replication hoặc Sticky Sessions khi chạy nhiều instance server. Nếu không, dùng cơ chế Stateless Token (như Signed CSRF Token).

### Failure
Nếu token bị mất hoặc hết hạn, người dùng sẽ nhận lỗi 403 Forbidden. Cần có trang thông báo lỗi thân thiện và hướng dẫn refresh trang.

### Monitoring
Log lại các trường hợp thiếu CSRF token hoặc token không hợp lệ để phát hiện các cuộc tấn công đang diễn ra.

## 11. Common mistakes
- Mistake: Chỉ dùng Cookie để xác thực mà không có biện pháp chống CSRF.
  Fix: Luôn sử dụng Anti-CSRF Token hoặc SameSite cookies.

- Mistake: Cho phép các hành động thay đổi dữ liệu (như xóa tài khoản) thực hiện qua phương thức GET.
  Fix: Chỉ sử dụng POST/PUT/DELETE cho các hành động thay đổi trạng thái.

## 12. Sample project
Xây dựng một ứng dụng web đơn giản:
1. Một trang "Kẻ tấn công" chứa một ảnh ẩn (`<img src="target-site.com/delete-account">`).
2. Một trang "Nạn nhân" có chức năng xóa tài khoản.
3. Thử nghiệm việc bật/tắt CSRF protection để thấy sự khác biệt.

## 13. Interview
### Core Q&A
1. Q: Tại sao SameSite cookie có thể giúp chống CSRF?
   A: Vì khi trình duyệt gửi yêu cầu từ trang của kẻ tấn công sang trang mục tiêu, thuộc tính `SameSite=Lax/Strict` sẽ ngăn trình duyệt gửi kèm cookie xác thực, khiến yêu cầu trở nên vô danh (unauthenticated).

2. Q: Anti-CSRF Token có cần thiết cho các ứng dụng hoàn toàn dùng JWT không?
   A: Nếu JWT được lưu trong Cookie, vẫn CẦN CSRF protection. Nếu JWT được lưu trong `localStorage` và gửi qua Header `Authorization`, ứng dụng tự nhiên miễn nhiễm với CSRF vì trình duyệt không tự động gửi header đó.

### Scenario
"Một trang web dùng API và AJAX hoàn toàn. Tôi có cần Anti-CSRF Token không?"
-> Trả lời: Nếu bạn dùng Cookie để lưu session/token, bạn vẫn cần. Một cách phổ biến cho ứng dụng AJAX là yêu cầu một Custom Header (như `X-XSRF-TOKEN`). Kẻ tấn công không thể dễ dàng thực hiện yêu cầu Cross-site với custom headers do chính sách CORS của trình duyệt.

## 14. References
- OWASP: [Cross-Site Request Forgery (CSRF) Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- MDN: [SameSite cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite)

## 15. Real-world Code
Nghiên cứu cách Spring Security hoặc Django tự động chèn CSRF token vào các form HTML.

## 16. Community
- YouTube: "CSRF Explained" - Computerphile.
- Stack Overflow: Tag [csrf].
- PortSwigger Web Security Academy: CSRF labs.
