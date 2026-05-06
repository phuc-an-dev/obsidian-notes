---
created: 2026-05-06
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/others"
  - "#topic/http"
related:
  - "[[Postman.md]]"
  - "[[RESTful API.md]]"
---

## 1. What
Bruno là một ứng dụng API Client mã nguồn mở (Open-source), nhanh và nhẹ, được thiết kế để thay thế cho Postman và Insomnia. Điểm khác biệt lớn nhất của Bruno là nó lưu trữ các bộ sưu tập API (Collections) trực tiếp dưới dạng các file văn bản thuần túy (.bru) trong thư mục dự án của bạn, thay vì lưu trữ trên Cloud của bên thứ ba.

## 2. Why
Trong những năm gần đây, Postman và Insomnia dần chuyển sang mô hình Cloud-first, ép buộc người dùng đăng nhập và đồng bộ dữ liệu lên máy chủ của họ. Điều này gây lo ngại về bảo mật dữ liệu nhạy cảm và khó khăn khi làm việc nhóm qua Git. Bruno ra đời với triết lý "Local-first" và "Git-friendly", cho phép các API requests được quản lý, commit và merge giống như mã nguồn của ứng dụng.

## 3. Mental Model
Hãy tưởng tượng Bruno giống như một **"Thư mục hồ sơ trong suốt"**.
- Postman giống như một két sắt đặt tại văn phòng của người khác (Cloud): bạn phải đăng nhập mới mở được và không biết họ làm gì bên trong.
- Bruno là một thư mục hồ sơ nằm ngay trên bàn làm việc của bạn (Local Storage). Bạn có thể tự do sao chép, gửi cho đồng nghiệp hoặc cất vào kho lưu trữ chung (Git) của công ty mà không cần thông qua bất kỳ trung gian nào.

## 4. Where it fits
Vị trí trong luồng phát triển:
`Code -> Bruno (.bru files) -> Git (Push/Pull) -> Team Collaboration`

## 5. When to use
- Khi bạn muốn quản lý API Collections cùng với mã nguồn dự án trong cùng một repository Git.
- Khi làm việc trong các dự án có yêu cầu bảo mật cao, không được phép lưu thông tin API lên các nền tảng Cloud bên thứ ba.
- Khi cần một API client nhẹ, khởi động nhanh và không yêu cầu đăng nhập.
- Khi muốn thực hiện các bài test API bằng script JavaScript một cách đơn giản.

## 6. When NOT to use
- Khi team của bạn đã quá quen thuộc và phụ thuộc vào các tính năng cộng tác đặc thù của Postman (như Team Workspaces, Postman Flows).
- Khi bạn cần các báo cáo kiểm thử (Testing Reports) cực kỳ chi tiết và tích hợp sâu mà Bruno chưa hỗ trợ mạnh mẽ bằng Postman.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Hoàn toàn miễn phí và mã nguồn mở. | Hệ sinh thái plugin và cộng đồng vẫn đang trong giai đoạn phát triển. |
| Dễ dàng quản lý phiên bản và giải quyết xung đột (Conflict) qua Git. | Giao diện tối giản, có thể thiếu một số tính năng đồ họa cao cấp của Postman. |
| Không yêu cầu đăng nhập, đảm bảo quyền riêng tư dữ liệu. | Một số tính năng nâng cao vẫn đang được hoàn thiện. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Postman | Mạnh mẽ nhất nhưng nặng nề và ép buộc dùng Cloud. |
| Insomnia | Giao diện đẹp, dễ dùng nhưng cũng đang chuyển dần sang hướng Cloud-only. |
| Thunder Client | Tiện lợi trong VS Code nhưng là closed-source và có giới hạn tính năng. |

## 9. How
Bruno sử dụng định dạng ngôn ngữ Bru (một dạng DSL đơn giản) để lưu trữ request:

```text
# Ví dụ file get-user.bru
meta {
  name: Get User Profile
  type: http
  seq: 1
}

get {
  url: {{base_url}}/users/1
}

headers {
  Authorization: Bearer {{token}}
}

tests {
  test("Status code is 200", function() {
    expect(res.getStatus()).to.equal(200);
  });
}
```

Bạn có thể cài đặt Bruno CLI để chạy test trong CI/CD:
```bash
npm install -g @usebruno/cli
bru run
```

## 10. Production concerns
### Security
Vì các file `.bru` nằm trong repo Git, hãy cực kỳ cẩn thận để không commit các file môi trường chứa bí mật (như `.env`). Hãy đưa các file chứa sensitive data vào `.gitignore`.

### Scalability
Bruno quản lý hàng trăm request rất tốt nhờ cấu trúc thư mục phân cấp rõ ràng trên ổ cứng. Việc tìm kiếm và thay đổi hàng loạt (Find & Replace) trở nên cực kỳ dễ dàng bằng các công cụ code editor (như VS Code).

## 11. Common mistakes
- Mistake: Quên commit các file `.bru` mới tạo lên Git khiến đồng nghiệp không thấy.
  Fix: Tập thói quen coi file API request là một phần của source code.

- Mistake: Để lộ Token trong phần `Initial Value` của biến môi trường.
  Fix: Luôn sử dụng cơ chế biến môi trường local (file không commit) cho các thông tin nhạy cảm.

## 12. Sample project
Tạo một thư mục `api-docs/` trong dự án Backend của bạn, mở Bruno và trỏ vào thư mục đó. Tạo các request CRUD cho một Resource, viết test đơn giản để kiểm tra mã trạng thái trả về, sau đó commit toàn bộ thư mục đó lên Git.

## 13. Interview
### Core Q&A
1. Q: Tại sao Bruno được gọi là "Git-friendly"?
   A: Vì Bruno lưu mỗi request dưới dạng một file văn bản riêng biệt với định dạng dễ đọc. Khi nhiều người cùng sửa đổi, Git có thể dễ dàng so sánh và giải quyết xung đột (Conflict Resolution), điều mà các file JSON khổng lồ của Postman làm rất tệ.

2. Q: Bruno CLI có thể tích hợp vào GitHub Actions không?
   A: Có, Bruno cung cấp gói `@usebruno/cli` cho phép chạy các bộ sưu tập API trực tiếp từ dòng lệnh, giúp kiểm tra tính đúng đắn của API ngay trong quy trình CI/CD.

### Scenario
"Dự án của bạn yêu cầu không được dùng các công cụ lưu trữ dữ liệu trên Cloud nước ngoài. Bạn đề xuất công cụ nào để test API?"
-> Trả lời: Tôi đề xuất Bruno. Vì nó là công cụ local-first, toàn bộ dữ liệu request và môi trường đều nằm trên máy của lập trình viên hoặc server của công ty thông qua Git, đảm bảo dữ liệu không bao giờ rời khỏi hạ tầng nội bộ.

## 14. References
- Official Site: [usebruno.com](https://www.usebruno.com/)
- Documentation: [Bruno Docs](https://docs.usebruno.com/)
- GitHub: [github.com/usebruno/bruno](https://github.com/usebruno/bruno)

## 15. Real-world Code
Mở bất kỳ project nào trên GitHub có thư mục chứa các file `.bru` để xem cách họ tổ chức request theo module.

## 16. Community
- Discord: Bruno Community
- Reddit: r/usebruno
- Twitter: @use_bruno
