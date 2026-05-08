---
created: 2026-05-07
tags:
  - "#type/tool"
  - "#status/draft"
  - "#lang/javascript"
  - "#topic/security"
related:
  - "[[dependabot-github.md]]"
  - "[[typescript.md]]"
  - "[[nextjs.md]]"
---

## 1. What
`npm audit fix` là một câu lệnh CLI của npm (Node Package Manager) được sử dụng để tự động cập nhật các thư viện phụ thuộc (dependencies) có lỗ hổng bảo mật lên phiên bản an toàn hơn. Nó dựa trên kết quả quét từ lệnh `npm audit`, đối chiếu với dữ liệu từ GitHub Advisory Database.

## 2. Why
Các dự án JavaScript thường sử dụng hàng trăm, thậm chí hàng ngàn package trung gian. Việc theo dõi thủ công các lỗ hổng bảo mật trong mớ hỗn độn này là không khả thi. `npm audit fix` giúp:
- **Tiết kiệm thời gian**: Tự động tìm và áp dụng các bản vá lỗi (patches).
- **Nâng cao bảo mật**: Đảm bảo dự án không sử dụng các thư viện có lỗi đã được công bố (CVE).
- **Dễ dàng bảo trì**: Giúp `package-lock.json` luôn ở trạng thái lành mạnh nhất có thể.

## 3. Mental Model
Hãy tưởng tượng dự án của bạn là một **"Ngôi nhà được xây từ nhiều linh kiện lắp ghép"**.
- `npm audit` giống như một **"Nhà kiểm định xây dựng"** đến kiểm tra xem có linh kiện nào (thư viện) bị lỗi thời hoặc có nguy cơ cháy nổ (lỗ hổng bảo mật) không.
- `npm audit fix` giống như một **"Đội thợ sửa chữa"** đi theo ngay sau đó. Họ cầm sẵn các linh kiện mới an toàn hơn và thay thế ngay những chỗ bị lỗi mà không làm thay đổi cấu trúc chính của ngôi nhà.

## 4. Where it fits
Local Machine -> **`npm install`** -> `package-lock.json` -> **`npm audit` (Scan)** -> **`npm audit fix` (Repair)** -> CI/CD.

## 5. When to use
- Ngay sau khi chạy `npm install` và nhận được cảnh báo về lỗ hổng bảo mật.
- Định kỳ hằng tuần hoặc hằng tháng để làm sạch codebase.
- Trước khi thực hiện một bản release Production quan trọng.

## 6. When NOT to use
- Khi bạn đang ở giữa một đợt refactor code lớn và không muốn thay đổi thêm bất kỳ version thư viện nào.
- Khi dự án không có bộ Unit Test tốt. Việc tự động cập nhật version có thể gây ra "breaking changes" âm thầm mà bạn không biết.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Khắc phục lỗ hổng bảo mật chỉ với 1 câu lệnh. | Có thể gây xung đột dependency nếu các bản vá yêu cầu version node cao hơn. |
| Chỉ cập nhật các bản patch/minor (an toàn). | Đôi khi không thể sửa hết mọi lỗi (phải dùng `--force`). |
| Cập nhật file `package-lock.json` một cách chuẩn xác. | Việc dùng `--force` có rủi ro làm hỏng ứng dụng rất cao. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `yarn audit` | Tương đương nhưng dành cho yarn. |
| `pnpm audit` | Dành cho pnpm, thường nhanh và tiết kiệm ổ cứng hơn. |
| Snyk | Công cụ chuyên sâu hơn, có giao diện web và hỗ trợ nhiều ngôn ngữ. |
| Dependabot | Tự động hoàn toàn trên môi trường GitHub, tạo Pull Request cho bạn. |

## 9. How
Lệnh kiểm tra các lỗ hổng:
```bash
npm audit
```

Lệnh tự động sửa các lỗ hổng (chỉ các thay đổi không gây break):
```bash
npm audit fix
```

Lệnh cưỡng chế sửa lỗi (bao gồm cả cập nhật version lớn - Major, có rủi ro cao):
```bash
npm audit fix --force
```

Chỉ xem trước những gì sẽ thay đổi mà không thực hiện sửa:
```bash
npm audit fix --dry-run
```

## 10. Production concerns
### CI/CD Integration
Bạn có thể cấu hình để CI build thất bại nếu có lỗ hổng bảo mật nghiêm trọng:
```bash
npm audit --audit-level=high
```

### Immutable Lockfile
Trên Production, bạn nên dùng `npm ci` để cài đặt thay vì `npm install`. Đừng bao giờ chạy `npm audit fix` trực tiếp trên server Production; hãy thực hiện ở local, test kỹ, commit `package-lock.json` rồi mới deploy.

## 11. Common mistakes
- Mistake: Chạy `npm audit fix --force` mà không đọc kỹ log xem nó sẽ cập nhật những gì.
  Fix: Luôn chạy `npm audit fix` trước, nếu vẫn còn lỗi thì mới xem xét từng cái một hoặc dùng `--dry-run` trước khi dùng `--force`.
- Mistake: Quên commit file `package-lock.json` sau khi fix.
  Fix: Luôn commit cả hai file manifest sau mỗi lần thay đổi dependency.

## 12. Sample project
Tạo một project Node.js cũ sử dụng `express` phiên bản 4.15.0 (có nhiều lỗi bảo mật). Chạy `npm audit` để thấy danh sách lỗi, sau đó dùng `npm audit fix` để đưa nó lên phiên bản an toàn nhất hiện tại (ví dụ 4.18.2).

## 13. Interview
### Core Q&A
1. Q: `npm audit fix` có cập nhật các thư viện lên phiên bản Major mới nhất không?
   A: Mặc định là không. Nó chỉ cập nhật các bản patch hoặc minor mà nó tin là tương thích ngược. Để cập nhật Major, bạn phải dùng thêm cờ `--force`.

2. Q: Tại sao đôi khi chạy `npm audit fix` xong vẫn còn báo lỗi bảo mật?
   A: Vì bản vá (fix) có thể chưa tồn tại cho phiên bản node/npm bạn đang dùng, hoặc bản vá yêu cầu một sự thay đổi lớn (breaking change) mà lệnh tự động không dám thực hiện.

### Scenario
"Dự án của bạn báo có 10 lỗ hổng bảo mật 'Critical' nhưng `npm audit fix` không sửa được cái nào. Bạn làm gì?"
-> Trả lời:
1. Đọc log chi tiết của `npm audit` để biết chính xác thư viện nào bị lỗi.
2. Kiểm tra xem có thể nâng cấp thủ công thư viện đó trong `package.json` hay không.
3. Nếu thư viện đó là dependency của một dependency khác (transitive), tôi sẽ dùng tính năng `overrides` trong `package.json` (npm 8+) để ép buộc sử dụng phiên bản an toàn.

## 14. References
- npm Documentation: [npm-audit](https://docs.npmjs.com/cli/v10/commands/npm-audit)
- GitHub Advisory Database: https://github.com/advisories

## 15. Real-world Code
Nghiên cứu mục `overrides` trong `package.json` của các dự án lớn để thấy cách họ xử lý các lỗ hổng cứng đầu:
```json
"overrides": {
  "graceful-fs": "^4.2.11"
}
```

## 16. Community
- Reddit: r/node.
- Stack Overflow: Tag [npm-audit].
- npm Blog: Các bài viết về bảo mật hệ sinh thái JavaScript.
