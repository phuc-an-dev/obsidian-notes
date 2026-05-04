---
created: 2026-04-30
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/ci-cd"
related:
  - "[[Git vs GitHub]]"
  - "[[Pull Request in Git]]"
---

## 1. What
GitHub Actions là một nền tảng tích hợp liên tục và triển khai liên tục (CI/CD) cho phép bạn tự động hóa quy trình xây dựng (build), kiểm thử (test) và triển khai (deploy) phần mềm ngay trên GitHub. Bạn có thể tạo ra các luồng công việc (workflows) để thực hiện các hành động cụ thể khi có sự kiện (events) xảy ra trong repository của mình.

## 2. Why
Trước khi có GitHub Actions, lập trình viên thường phải sử dụng các công cụ bên thứ ba như Jenkins, Travis CI hay CircleCI để thiết lập CI/CD. Việc cấu hình các công cụ này đòi hỏi nhiều công sức bảo trì và đôi khi khó tích hợp sâu với GitHub. GitHub Actions ra đời để mang lại trải nghiệm tự động hóa mượt mà, "all-in-one" ngay tại nơi lưu trữ code.

## 3. Mental Model
Hãy tưởng tượng GitHub Actions giống như một "Người quản gia tận tụy" cho repository của bạn.
- **Sự kiện (Event)**: Giống như tiếng chuông cửa (có ai đó push code hoặc mở Pull Request).
- **Workflow**: Cuốn sổ tay hướng dẫn cho người quản gia (file YAML).
- **Jobs/Steps**: Các đầu việc cụ thể người quản gia phải làm (Ví dụ: 1. Lau nhà, 2. Rửa bát, 3. Nấu cơm).
Khi có chuông cửa reo, người quản gia sẽ tự động mở sổ tay ra và thực hiện chính xác các bước đã được ghi lại mà không cần bạn phải nhắc nhở.

## 4. Where it fits
GitHub Actions đóng vai trò là "Cầu nối tự động" giữa code và môi trường vận hành:
Code Push -> GitHub Actions (Build/Test/Lint) -> Artifact Generation -> Deployment (AWS/Heroku/Vercel).

## 5. When to use
- Tự động chạy Unit Test và Integration Test mỗi khi có Pull Request.
- Tự động kiểm tra chất lượng code (Linting) và quét lỗ hổng bảo mật.
- Tự động triển khai ứng dụng lên môi trường Production sau khi merge vào nhánh chính.
- Tự động tạo release và đẩy các gói thư viện lên NPM, Maven, Docker Hub.

## 6. When NOT to use
- Các dự án cực kỳ nhỏ hoặc mang tính chất lưu trữ cá nhân không cần quy trình kiểm thử phức tạp.
- Các tác vụ đòi hỏi tài nguyên phần cứng cực lớn hoặc thời gian chạy quá lâu (trên 6 tiếng - giới hạn của GitHub-hosted runners).
- Khi doanh nghiệp có quy định bảo mật nghiêm ngặt yêu cầu mọi công cụ CI/CD phải chạy hoàn toàn trong mạng nội bộ (on-premise).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Tích hợp cực sâu với hệ sinh thái GitHub | Giới hạn về dung lượng và thời gian chạy cho tài khoản miễn phí |
| Cấu hình bằng YAML dễ đọc, dễ quản lý | Vendor lock-in (khó chuyển sang nền tảng khác) |
| Chợ ứng dụng (Marketplace) khổng lồ với các Actions sẵn có | Đôi khi khó debug các lỗi môi trường phức tạp |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Jenkins | Mạnh mẽ, tùy biến cực cao nhưng yêu cầu tự cài đặt và bảo trì server riêng. |
| GitLab CI/CD | Đối thủ trực tiếp, tích hợp sâu với GitLab, có cơ chế quản lý biến môi trường rất tốt. |
| CircleCI | Tốc độ nhanh, cấu hình tối ưu nhưng chi phí có thể cao hơn. |

## 9. How
Ví dụ một file `.github/workflows/main.yml` cơ bản:
```yaml
name: CI/CD Pipeline

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  build_and_test:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Set up Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '20'

    - name: Install dependencies
      run: npm install

    - name: Run Tests
      run: npm test
```

## 10. Production concerns
### Scaling
Bạn có thể sử dụng "Self-hosted runners" (máy chủ riêng của bạn) thay vì dùng máy chủ của GitHub để tăng tốc độ build và tùy chỉnh phần cứng.

### Failure
Nếu một bước (step) bị lỗi, toàn bộ job sẽ dừng lại và báo đỏ. Cần cấu hình thông báo (qua Slack/Email) để team xử lý kịp thời.

### Monitoring
GitHub cung cấp giao diện trực quan để theo dõi lịch sử chạy, log chi tiết từng bước và biểu đồ thời gian thực thi của Workflows.

## 11. Common mistakes
- Mistake: Lưu trữ API Keys, mật khẩu trực tiếp trong file YAML.
  Fix: Luôn sử dụng "GitHub Secrets" và gọi chúng qua biến môi trường `${{ secrets.MY_SECRET }}`.

- Mistake: Không giới hạn quyền hạn của `GITHUB_TOKEN`.
  Fix: Luôn tuân thủ nguyên tắc đặc quyền tối thiểu (least privilege) khi cấu hình permissions cho workflow.

## 12. Sample project
Thiết lập một workflow tự động đẩy Docker Image lên Docker Hub mỗi khi bạn đánh tag (git tag) cho phiên bản mới của ứng dụng.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa `Job` và `Step` trong GitHub Actions là gì?
   A: `Step` là một tác vụ nhỏ nhất chạy tuần tự trong cùng một máy ảo. `Job` là tập hợp của nhiều bước, các Job có thể chạy song song trên các máy ảo khác nhau trừ khi được cấu hình phụ thuộc (`needs`).

2. Q: "Action" trong GitHub Actions thực chất là gì?
   A: Action là một thành phần có thể tái sử dụng, thực hiện một tác vụ cụ thể (như checkout code, setup java). Nó có thể là một Docker container hoặc một script JavaScript được đóng gói lại để người dùng khác dễ dàng gọi trong workflow.

### Scenario
Tình huống: Làm thế nào để đảm bảo một Job chỉ chạy sau khi một Job khác đã thành công?
Trả lời: Sử dụng thuộc tính `needs` trong định nghĩa Job. Ví dụ: `deploy_job: needs: build_job`. Điều này tạo ra một chuỗi phụ thuộc, đảm bảo tính an toàn cho quy trình CI/CD.

## 14. References
- GitHub Actions Documentation: https://docs.github.com/en/actions
- GitHub Marketplace: https://github.com/marketplace?type=actions
- Workflow syntax: https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions

## 15. Real-world Code
Hầu hết các dự án open-source trên GitHub hiện nay đều có thư mục `.github/workflows/` chứa các cấu hình CI/CD thực tế.

## 16. Community
- Reddit: r/github
- GitHub Community Forum: https://github.community/c/code-to-cloud/github-actions/41
- Lab thực hành của GitHub: GitHub Skills.
