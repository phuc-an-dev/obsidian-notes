---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/ci-cd"
related:
  - "[[github-actions-workflow-protection.md]]"
  - "[[github-secrets.md]]"
  - "[[github-actions-cd.md]]"
  - "[[github-actions-triggers.md]]"
  - "[[GitHub Actions MySQL Service Container.md]]"
  - "[[dependabot-github.md]]"
---

## 1. What
GitHub Actions CI (thường được định nghĩa trong file `ci.yml`) là một hệ thống tự động hóa tích hợp liên tục (Continuous Integration) tích hợp sẵn trong GitHub. Nó cho phép lập trình viên tự động hóa các quy trình như build, test, lint và package mã nguồn ngay khi có sự kiện thay đổi code (push, pull request).

## 2. Why
Trước khi có CI, việc kiểm tra code phụ thuộc vào ý thức của lập trình viên (tự chạy test dưới máy local) hoặc một máy chủ Jenkins riêng biệt khó quản lý. Điều này dẫn đến tình trạng "broken build" thường xuyên. GitHub CI ra đời để đảm bảo code luôn ở trạng thái ổn định, giảm thiểu lỗi human error và cung cấp phản hồi nhanh chóng cho team về chất lượng code.

## 3. Mental Model
Hãy tưởng tượng GitHub CI giống như một **"Trạm Kiểm Định Tự Động"** ở cửa ngõ của một nhà máy. Mỗi khi bạn mang một kiện hàng (Code) đến, trạm này sẽ tự động:
1. Mở kiện hàng ra (Checkout).
2. Kiểm tra xem các bộ phận có khớp nhau không (Build).
3. Thử vận hành xem có hỏng hóc gì không (Test).
Nếu mọi thứ đạt chuẩn, kiện hàng mới được phép nhập kho (Merge).

## 4. Where it fits
Vị trí trong luồng phát triển:
`Local Development -> Git Push -> GitHub CI (ci.yml) -> Review/Merge -> CD (Deployment)`

File này phải được đặt tại đường dẫn: `.github/workflows/ci.yml`.

## 5. When to use
- Mọi dự án phần mềm có từ 2 thành viên trở lên để đảm bảo tính đồng nhất.
- Khi muốn tự động hóa các tác vụ lặp đi lặp lại như chạy Unit Test, Integration Test.
- Khi cần kiểm tra tính tương thích của code trên nhiều hệ điều hành hoặc phiên bản ngôn ngữ khác nhau (Matrix build).

## 6. When NOT to use
- Các repository cá nhân chỉ dùng để lưu trữ file tĩnh, không có logic thực thi.
- Các project quá nhỏ, chạy một lần rồi bỏ (PoC cực nhanh).
- Khi project có các yêu cầu bảo mật đặc thù mà GitHub Runners (cloud) không thể đáp ứng (trường hợp này nên dùng Self-hosted Runners).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Miễn phí cho các repo công khai (Public). | Giới hạn số phút sử dụng hàng tháng cho repo tư nhân (Private). |
| Tích hợp cực sâu với GitHub (giao diện, PR, Checks API). | Cú pháp YAML đôi khi khó debug nếu workflow phức tạp. |
| Chạy trên Cloud, không tốn công bảo trì server. | Phụ thuộc hoàn toàn vào uptime của GitHub. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Jenkins | Tự quản lý hoàn toàn, mạnh mẽ nhưng tốn công bảo trì và setup server. |
| GitLab CI | Rất mạnh, tích hợp tốt nếu bạn dùng GitLab làm nơi lưu code. |
| CircleCI | Tốc độ build nhanh, nhưng tốn phí và là bên thứ 3. |

## 9. How
Ví dụ một file `ci.yml` cơ bản cho project Node.js:

```yaml
name: Node.js CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Use Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'
        cache: 'npm'

    - name: Install dependencies
      run: npm ci

    - name: Run Lint
      run: npm run lint

    - name: Run Tests
      run: npm test
```

## 10. Production concerns
### Scaling
Sử dụng `matrix` để chạy song song trên nhiều phiên bản:
```yaml
strategy:
  matrix:
    node: [18, 20, 22]
```

### Failure
Sử dụng `continue-on-error: true` cho các bước không quá quan trọng hoặc `timeout-minutes` để tránh treo workflow gây tốn phút build.

### Monitoring
Sử dụng GitHub Slack App để thông báo trạng thái build trực tiếp vào kênh làm việc của team.

## 11. Common mistakes
- Mistake: Quên sử dụng `actions/checkout`. Nếu không có bước này, máy ảo sẽ không có code để chạy.
  Fix: Luôn đặt `uses: actions/checkout@v4` là step đầu tiên.

- Mistake: Cài đặt dependencies bằng `npm install` thay vì `npm ci`.
  Fix: Dùng `npm ci` trong môi trường CI để đảm bảo đúng phiên bản trong `package-lock.json` và tốc độ nhanh hơn.

## 12. Sample project
Tạo một repository Spring Boot, thiết lập CI workflow để:
1. Chạy trên Ubuntu.
2. Setup Java 17 (Temurin).
3. Cache Maven/Gradle dependencies.
4. Chạy `mvn clean verify`.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa `run` và `uses` trong GitHub Actions là gì?
   A: `run` dùng để chạy các lệnh shell trực tiếp trên máy ảo. `uses` dùng để gọi các Action đã được đóng gói sẵn (reusable actions) từ GitHub Marketplace hoặc local.

2. Q: Làm thế nào để truyền dữ liệu giữa các Jobs khác nhau?
   A: Sử dụng `artifacts` (upload/download) hoặc `outputs` nếu chỉ là các giá trị string đơn giản.

### Scenario
"Workflow CI của bạn chạy rất chậm (10-15 phút), bạn làm gì để tối ưu?"
-> Trả lời: 
1. Sử dụng caching cho dependencies (Maven, NPM).
2. Chia nhỏ Jobs để chạy song song (Parallelism).
3. Chỉ chạy những step cần thiết dựa trên các file thay đổi (Paths filter).

## 14. References
- Official Docs: [GitHub Actions Documentation](https://docs.github.com/en/actions)
- GitHub Marketplace: [Actions Gallery](https://github.com/marketplace?type=actions)

## 15. Real-world Code
- [React GitHub Workflows](https://github.com/facebook/react/tree/main/.github/workflows)
- [Spring Boot Workflows](https://github.com/spring-projects/spring-boot/tree/main/.github/workflows)

## 16. Community
- Reddit: r/GitHubActions
- Stack Overflow: Tag [github-actions]
- Blog: GitHub Engineering Blog
