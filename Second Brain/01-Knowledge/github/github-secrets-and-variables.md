---
created: 2026-05-05
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/others"
  - "#topic/error-handling"
related:
  - "[[github-secrets]]"
---

## 1. What
Đây là tài liệu phân biệt giữa GitHub Secrets (Biến bảo mật) và GitHub Variables (Biến cấu hình). Cả hai đều được dùng để truyền tham số vào GitHub Actions nhưng có mức độ bảo mật và cách hiển thị khác nhau.

## 2. Why
Trước khi có sự phân tách này, người dùng thường phải lưu cả những thông tin không nhạy cảm (như tên App, URL API public) vào Secrets. Điều này gây khó khăn cho việc quản lý vì Secrets không thể xem lại giá trị (write-only). GitHub Variables ra đời để giải quyết bài toán lưu trữ các cấu hình "non-sensitive" một cách minh bạch.

## 3. Mental Model
Hãy tưởng tượng dự án của bạn là một nhà hàng.
- **Secrets**: Là công thức nấu ăn gia truyền (chỉ đầu bếp biết, không ai được xem).
- **Variables**: Là menu món ăn hoặc địa chỉ nhà hàng (ai cũng có thể xem, chỉnh sửa công khai để khách hàng biết).

## 4. Where it fits
Settings -> Security -> Secrets and variables -> Actions.
Trong workflow, chúng được truy cập qua context: `{{ secrets.NAME }}` và `{{ vars.NAME }}`.

## 5. When to use
- **Dùng Secrets**: API Keys, Password, SSH Keys, Token.
- **Dùng Variables**: Tên môi trường (staging/prod), Port hiệu hành, URL endpoint công khai, Flag bật/tắt tính năng (feature toggles).

## 6. When NOT to use
- Không dùng Variables cho bất kỳ dữ liệu nào mà bạn không muốn contributor xem được.
- Không dùng Secrets cho các biến cần debug nhanh vì giá trị bị mask trong log.

## 7. Trade-offs
| Đặc điểm | GitHub Secrets | GitHub Variables |
|----------|----------------|------------------|
| Bảo mật | Được mã hóa mạnh | Không mã hóa (Plain text) |
| Hiển thị | Chỉ có dấu `***` trong logs | Hiển thị rõ ràng trong logs |
| Truy cập | Write-only (Không thể xem lại) | Read-write (Có thể xem và sửa) |
| Mục đích | Bảo mật thông tin | Cấu hình ứng dụng |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Hardcoded code | Rất tệ cho bảo mật và khả năng tái sử dụng |
| Matrix Strategy | Dùng để định nghĩa biến trực tiếp trong YAML cho các job chạy song song |
| Environment Files | Lưu biến trong file `.env` (phải cẩn thận với gitignore) |

## 9. How
```yaml
name: Deploy App
on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Use Variables and Secrets
        run: |
          echo "Deploying to: ${{ vars.DEPLOY_REGION }}"
          echo "App Name: ${{ vars.APP_NAME }}"
          # Secret sẽ hiển thị là *** trong logs
          curl -H "Authorization: Bearer ${{ secrets.API_TOKEN }}" ${{ vars.API_URL }}
```

## 10. Production concerns
### Scaling
Sử dụng Organization Variables để quản lý các cấu hình chung cho hàng trăm repository (ví dụ: `COMPANY_NAME`).

### Failure
Nếu nhầm lẫn đưa secret vào variables, thông tin nhạy cảm sẽ bị log ra console và có thể bị hacker thu thập. Luôn kiểm tra kỹ loại biến trước khi lưu.

### Monitoring
Sử dụng "Configuration as code" (ví dụ dùng Terraform) để quản lý cả Secrets và Variables, giúp theo dõi lịch sử thay đổi (Audit trail).

## 11. Common mistakes
- Mistake: Lưu `DATABASE_PASSWORD` vào Variables vì lười không muốn copy-paste nhiều lần.
  Fix: Luôn ưu tiên Secrets cho các thông tin có thể gây nguy hại nếu lộ lọt.

- Mistake: Quên không cập nhật `vars` khi chuyển từ môi trường Test sang Production.
  Fix: Sử dụng GitHub Environments để định nghĩa `vars` riêng cho từng môi trường.

## 12. Sample project
Thiết kế một hệ thống deploy đa khu vực (multi-region). Sử dụng `vars.TARGET_REGION` để xác định nơi deploy và `secrets.AWS_ACCESS_KEY` để thực hiện lệnh. Thử thay đổi `vars` và quan sát workflow tự động thay đổi hành vi mà không cần sửa code.

## 13. Interview
### Core Q&A
1. Q: Khi nào bạn nên chọn Variables thay vì Secrets?
   A: Khi dữ liệu đó không cần bảo mật (không phải key/pass) và bạn muốn đồng đội có thể nhìn thấy giá trị đó để dễ dàng debug hoặc thay đổi cấu hình mà không cần hỏi người giữ key.
2. Q: Có thể chuyển đổi từ Variable sang Secret được không?
   A: Không có nút chuyển đổi trực tiếp. Bạn phải xóa biến ở Variables và tạo mới ở mục Secrets (và ngược lại).

### Scenario
Bạn đang cấu hình một ứng dụng React trên GitHub Actions. Bạn có 2 biến: `API_URL` (public) và `FIREBASE_API_KEY` (private). Bạn sẽ lưu chúng như thế nào?
Trả lời: Tôi sẽ lưu `API_URL` vào GitHub Variables vì nó là thông tin công khai, giúp team dễ dàng kiểm tra xem đang trỏ đúng về backend nào. Còn `FIREBASE_API_KEY` tôi bắt buộc phải lưu vào GitHub Secrets để tránh bị đánh cắp và tiêu tốn quota hoặc truy cập trái phép vào dữ liệu người dùng.

## 14. References
- Official Docs: https://docs.github.com/en/actions/learn-github-actions/variables
- GitHub Repo: https://github.com/actions/starter-workflows
- Spec / RFC: N/A
- Changelog: GitHub Actions: Variables are now generally available (2023).

## 15. Real-world Code
https://github.com/github/docs (Chính repository tài liệu của GitHub cũng sử dụng kết hợp cả hai loại biến này)

## 16. Community
- Reddit: r/GitHub
- Stack Overflow: Tag #github-actions #environment-variables
- Blog: GitHub Engineering Blog
- Talk: Modern CI/CD practices with GitHub Actions.
