---
created: 2026-05-07
tags:
  - "#type/tool"
  - "#status/draft"
  - "#lang/others"
  - "#topic/devops"
related:
  - "[[github-actions-ci.md]]"
  - "[[github-actions-workflow-protection.md]]"
  - "[[github-secrets.md]]"
  - "[[npm-audit-fix.md]]"
---

## 1. What
GitHub Dependabot là một công cụ tự động hóa được tích hợp sẵn vào GitHub, giúp phát hiện, báo cáo và tự động cập nhật các thư viện phụ thuộc (dependencies) trong dự án của bạn. Nó hoạt động bằng cách quét các file manifest (như `package.json`, `pom.xml`, `requirements.txt`) để tìm các phiên bản cũ hoặc có lỗ hổng bảo mật.

## 2. Why
Việc duy trì các thư viện luôn ở phiên bản mới nhất là một nhiệm vụ tẻ nhạt nhưng cực kỳ quan trọng:
- **Bảo mật**: Các thư viện cũ thường chứa các lỗ hổng đã được công bố (CVE). Dependabot giúp bạn vá lỗi ngay khi có bản cập nhật.
- **Tính năng**: Giúp dự án tiếp cận sớm với các tính năng mới và bản sửa lỗi từ nhà phát triển thư viện.
- **Giảm nợ kỹ thuật**: Ngăn chặn việc dự án bị tụt hậu quá xa so với sự phát triển của hệ sinh thái, giúp việc nâng cấp sau này ít đau đớn hơn.

## 3. Mental Model
Hãy tưởng tượng Dependabot giống như một **"Quản gia mẫn cán"** cho thư viện sách của bạn:
- Hàng ngày, quản gia sẽ kiểm tra danh sách các cuốn sách bạn đang có.
- Nếu ông ấy thấy một cuốn sách đã có phiên bản tái bản mới (Update) hoặc phát hiện cuốn sách đó có nội dung độc hại (Security alert).
- Ông ấy sẽ tự động mua cuốn sách mới về, đặt lên bàn và viết một tờ giấy nhắn (Pull Request): "Tôi đã thấy bản mới, bạn có muốn thay thế bản cũ không?". Bạn chỉ việc gật đầu (Merge) là xong.

## 4. Where it fits
GitHub Repository -> Manifest Files -> **Dependabot Service** -> Pull Requests -> CI/CD Pipeline -> Main Branch.

## 5. When to use
- Mọi dự án phần mềm sử dụng các trình quản lý gói (Package Managers).
- Khi muốn tuân thủ quy trình DevSecOps, đưa bảo mật vào sớm trong vòng đời phát triển.
- Các dự án mã nguồn mở cần duy trì sự ổn định và an toàn cho cộng đồng.

## 6. When NOT to use
- Các dự án cực kỳ cũ, nơi việc nâng cấp một thư viện có thể làm hỏng toàn bộ hệ thống do không có unit test đảm bảo.
- Khi dự án sử dụng các thư viện nội bộ (private) mà Dependabot không có quyền truy cập (trừ khi cấu hình thêm secrets).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Tự động hóa hoàn toàn việc theo dõi version. | Tạo ra quá nhiều Pull Requests gây nhiễu cho team (PR noise). |
| Vá lỗ hổng bảo mật cực nhanh. | Có thể gây lỗi ứng dụng (breaking changes) nếu merge mà không kiểm tra kỹ. |
| Dễ dàng cấu hình qua file YAML. | Đôi khi gây xung đột dependency giữa các PR khác nhau. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Renovate | Mạnh mẽ hơn, hỗ trợ nhiều nền tảng ngoài GitHub, cấu hình cực kỳ linh hoạt nhưng phức tạp hơn. |
| Snyk | Tập trung sâu vào bảo mật hơn là chỉ cập nhật version, có phí cho doanh nghiệp. |
| Manual Update | Thủ công hoàn toàn, tốn thời gian và dễ bỏ sót. |

## 9. How
Cấu hình Dependabot thông qua file `.github/dependabot.yml`:

```yaml
version: 2
updates:
  # Cập nhật cho npm
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "daily"
    open-pull-requests-limit: 10
    reviewers:
      - "octocat"

  # Cập nhật cho Maven (Java)
  - package-ecosystem: "maven"
    directory: "/"
    schedule:
      interval: "weekly"
    ignore:
      - dependency-name: "org.springframework:*"
        update-types: ["version-update:semver-major"]
```

## 10. Production concerns
### CI/CD Integration
Luôn chạy bộ test CI của bạn trên các PR do Dependabot tạo ra. Chỉ merge khi test pass hoàn toàn.

### Auto-merge
Sử dụng các GitHub Actions như `ahmadnassri/action-dependabot-auto-merge` để tự động merge các bản cập nhật nhỏ (patch/minor) nếu CI pass, giúp giảm tải công việc Review.

## 11. Common mistakes
- Mistake: Không có unit test nhưng lại tin tưởng hoàn toàn vào Dependabot.
  Fix: Luôn xây dựng bộ test vững chắc trước khi bật Dependabot.
- Mistake: Để Dependabot tạo quá nhiều PR cùng lúc làm nghẽn pipeline.
  Fix: Sử dụng `open-pull-requests-limit` để giới hạn số lượng PR mở.

## 12. Sample project
Thiết lập Dependabot cho một dự án Node.js:
1. Tạo file `.github/dependabot.yml`.
2. Cấu hình quét hàng tuần.
3. Quan sát Dependabot tạo PR cho các thư viện cũ như `lodash` hoặc `express`.
4. Thực hiện Review và Merge trực tiếp trên giao diện GitHub.

## 13. Interview
### Core Q&A
1. Q: Dependabot Security Alerts và Dependabot Version Updates khác nhau như thế nào?
   A: Security Alerts chỉ thông báo khi phát hiện lỗ hổng bảo mật nghiêm trọng. Version Updates chủ động tạo PR để cập nhật lên bản mới nhất bất kể có lỗi bảo mật hay không.

2. Q: Làm thế nào để Dependabot truy cập được vào private registry (ví dụ JFrog hoặc ECR)?
   A: Bạn cần cấu hình `registries` trong file `dependabot.yml` và cung cấp thông tin xác thực qua GitHub Secrets.

### Scenario
"Dự án của bạn có 50 thư viện và Dependabot tạo 20 PR mỗi ngày khiến team không thể làm việc khác. Bạn giải quyết thế nào?"
-> Trả lời:
1. Giảm tần suất quét từ `daily` xuống `weekly`.
2. Sử dụng `ignore` để bỏ qua các thư viện không quan trọng hoặc chỉ quan tâm đến `security updates`.
3. Nhóm các PR lại (Grouping) bằng tính năng `groups` trong Dependabot (mới).
4. Thiết lập auto-merge cho các bản patch update.

## 14. References
- GitHub Docs: [Configuring Dependabot](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuring-dependabot-version-updates)
- GitHub Docs: [Dependabot YAML options](https://docs.github.com/en/code-security/dependabot/dependabot-version-updates/configuration-options-for-the-dependabot.yml-file)

## 15. Real-world Code
Nghiên cứu file `dependabot.yml` của các dự án lớn như **Next.js** hoặc **VS Code** trên GitHub để xem cách họ quản lý hàng trăm dependencies.

## 16. Community
- GitHub Blog: Tin tức mới nhất về Dependabot.
- Reddit: r/github.
- Stack Overflow: Tag [github-dependabot].
