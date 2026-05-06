---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/security"
related:
  - "[[github-environments.md]]"
  - "[[github-secrets.md]]"
---

## 1. What
GitHub Environment Secrets là các biến bí mật (Secrets) được định nghĩa và giới hạn phạm vi sử dụng trong một Environment cụ thể trên GitHub Actions. Chúng cho phép lưu trữ các thông tin nhạy cảm như API Keys, Passwords hoặc Certificates mà chỉ những công việc (Jobs) được gán vào Environment đó mới có quyền truy cập.

## 2. Why
Trong các dự án lớn, việc sử dụng chung một tập hợp Secrets cho mọi môi trường (Development, Staging, Production) là cực kỳ nguy hiểm. Environment Secrets ra đời để:
- **Tách biệt môi trường**: Đảm bảo Job chạy ở Staging không bao giờ vô tình truy cập được Database của Production.
- **Bảo mật tăng cường**: Kết hợp với "Environment protection rules", một secret chỉ được giải mã khi Job đó đã được phê duyệt (Approved) bởi người quản lý.
- **Tuân thủ (Compliance)**: Đáp ứng các tiêu chuẩn bảo mật về việc quản lý truy cập đặc quyền.

## 3. Mental Model
Hãy tưởng tượng Environment Secrets giống như những **"Chiếc hộp đen trong từng căn phòng riêng biệt"**:
- Bạn có một tòa nhà (Repository) với nhiều phòng (Environments).
- Trong phòng Production có một chiếc hộp đen chứa vàng (Production Secret).
- Chỉ những người được phép vào phòng Production (Job gán environment: production) mới có chìa khóa để mở chiếc hộp đó.
- Người ở phòng Staging (Job gán environment: staging) dù có đứng ngay cạnh cũng không thể nhìn thấy hay mở được chiếc hộp của phòng Production.

## 4. Where it fits
Vị trí trong cấu trúc bảo mật GitHub:
`Organization Secrets -> Repository Secrets -> Environment Secrets (Độ ưu tiên cao nhất)`

## 5. When to use
- Khi bạn có các thông tin bí mật khác nhau cho từng môi trường triển khai.
- Khi cần áp dụng quy trình Manual Approval cho việc sử dụng secret trên Production.
- Khi muốn giới hạn bí mật chỉ được dùng bởi các nhánh cụ thể thông qua cấu hình của Environment.

## 6. When NOT to use
- Khi một bí mật được dùng chung cho toàn bộ dự án trên mọi nhánh và mọi môi trường (nên dùng Repository Secrets để tránh lặp lại).
- Khi bạn sử dụng GitHub cá nhân miễn phí cho các repository riêng tư (không hỗ trợ Environment protection).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Độ an toàn cao nhất, cô lập rủi ro theo môi trường. | Quản lý phức tạp hơn khi số lượng môi trường và secret tăng lên. |
| Có thể dùng cùng một tên biến cho các giá trị khác nhau. | Khó khăn trong việc đồng bộ secret giữa các môi trường nếu không có tool hỗ trợ. |
| Tích hợp sẵn cơ chế phê duyệt (Approval). | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Repository Secrets | Dễ setup hơn nhưng không có tính cô lập và bảo vệ theo môi trường. |
| HashiCorp Vault / AWS Secrets Manager | Chuyên nghiệp hơn, hỗ trợ xoay vòng key tự động, nhưng yêu cầu tích hợp phức tạp hơn. |

## 9. How
Cách truy xuất Environment Secret trong workflow:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: production # Bắt buộc phải khai báo để lấy được Environment Secrets
    
    steps:
    - name: Use Secret
      run: |
        echo "Connecting to DB..."
        # GitHub sẽ tự động lấy biến DB_URL từ Environment 'production'
        connect_db.sh ${{ secrets.DB_URL }}
```

Lưu ý quan trọng: Nếu bạn có cả Repository Secret và Environment Secret cùng tên `DB_URL`, GitHub Actions sẽ ưu tiên lấy giá trị từ **Environment Secret**.

## 10. Production concerns
### Secret Masking
GitHub tự động ẩn (mask) các secret trong logs (hiển thị thành `***`). Tuy nhiên, cần tránh in secret ra logs theo các cách gián tiếp (như mã hóa base64 rồi in).

### Reviewers
Luôn thiết lập ít nhất 1 Reviewer cho môi trường Production để kiểm soát việc kích hoạt các Jobs có quyền truy cập vào Production Secrets.

## 11. Common mistakes
- Mistake: Quên khai báo dòng `environment: <name>` trong Job. Kết quả là `${{ secrets.MY_SECRET }}` sẽ trả về rỗng hoặc lấy giá trị của Repository Secret.
  Fix: Luôn khai báo environment name ở cấp độ job.

- Mistake: In secret ra log để debug.
  Fix: Sử dụng các công cụ debug của GitHub Actions hoặc kiểm tra giá trị secret ở máy local trước khi push.

## 12. Sample project
Xây dựng workflow deploy đa môi trường:
1. Environment `staging`: Chứa secret `APP_KEY` của bản test.
2. Environment `production`: Chứa secret `APP_KEY` của bản chính thức và yêu cầu Tech Lead approve.
3. Job deploy dùng chung một file YAML nhưng thay đổi `environment` dựa trên nhánh.

## 13. Interview
### Core Q&A
1. Q: Nếu một Repository Secret và một Environment Secret có cùng tên, GitHub sẽ chọn cái nào?
   A: GitHub sẽ ưu tiên Environment Secret.

2. Q: Làm sao để bảo vệ Environment Secrets khỏi các Pull Request từ các nhánh lạ?
   A: Trong cài đặt Environment, chọn "Deployment branches" và chỉ cho phép nhánh `main` hoặc `tags` cụ thể.

### Scenario
"Hacker chiếm được quyền push code lên một nhánh feature. Họ có thể đánh cắp Production Secrets không?"
-> Trả lời: Nếu hệ thống được cấu hình đúng với GitHub Environments, hacker không thể lấy được Production Secrets. Vì:
1. Production Secrets chỉ được cấp cho job có gán `environment: production`.
2. Environment `production` đã được giới hạn chỉ chạy từ nhánh `main`.
3. Job chạy trên Environment `production` yêu cầu phê duyệt thủ công từ người quản lý.

## 14. References
- Official Docs: [Environment secrets](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment#environment-secrets)
- Security Guide: [Encrypted secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)

## 15. Real-world Code
Tìm kiếm các file `.github/workflows` trong các dự án Enterprise trên GitHub để thấy cách họ phân bổ job theo environment.

## 16. Community
- Reddit: r/GitHubActions
- GitHub Security Advisory.
- Stack Overflow: Tag [github-actions-secrets].
