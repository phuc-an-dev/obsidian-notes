---
created: 2026-05-05
tags:
  - "#type/pattern"
  - "#status/draft"
  - "#lang/others"
  - "#topic/devops"
related:
  - "[[github-secrets]]"
  - "[[github-manual-deployment]]"
  - "[[github-actions-workflow-protection]]"
---

## 1. What
Inject Secret là quá trình đưa các giá trị nhạy cảm đã được lưu trữ trong GitHub Secrets vào môi trường thực thi của ứng dụng trong quá trình Deploy. Quá trình này thường thông qua việc gán Secret vào Biến môi trường (Environment Variables) hoặc ghi trực tiếp vào các tệp cấu hình (Config files) trước khi ứng dụng khởi chạy.

## 2. Why
Việc "bơm" (inject) secret đúng cách đảm bảo:
- **Tách biệt dữ liệu**: Code chỉ chứa các placeholder (tên biến), còn giá trị thật nằm an toàn trong kho lưu trữ của GitHub.
- **Tính linh hoạt**: Cùng một code nhưng có thể chạy ở nhiều môi trường (Dev, Staging, Prod) chỉ bằng cách thay đổi giá trị inject.
- **Bảo mật**: Secret chỉ tồn tại trong bộ nhớ của Runner trong lúc thực thi và bị xóa sạch sau khi job kết thúc.

## 3. Mental Model
Hãy tưởng tượng ứng dụng của bạn là một **"Máy pha cà phê"** có các ngăn trống (Environment Variables).
- Bạn không để sẵn hạt cà phê hay đường (Secrets) trong máy khi vận chuyển.
- Khi máy bắt đầu hoạt động (Deploy), GitHub Actions sẽ đóng vai trò người phục vụ, lấy nguyên liệu từ kho bảo mật và đổ vào các ngăn tương ứng để máy có thể pha được cà phê.

## 4. Where it fits
`GitHub Secrets Storage -> Workflow YAML -> Runner Environment -> Application Process`

## 5. When to use
- Cần truyền API Key cho ứng dụng Frontend (thông qua Build-time variables).
- Cần truyền Database Connection String cho ứng dụng Backend.
- Cần tạo file `.env` hoặc `config.json` động trong quá trình build image Docker.
- Cần truyền SSH Private Key để thực hiện lệnh `rsync` hoặc `ssh` tới server.

## 6. When NOT to use
- Truyền các tham số không nhạy cảm (nên dùng GitHub Variables).
- Đừng inject secret vào các bước không cần thiết (ví dụ: bước Linting hoặc Unit Test không cần DB).
- Tránh inject secret vào các script shell phức tạp khó kiểm soát đầu ra log.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Dễ dàng tự động hóa quy trình Deploy. | Nếu workflow bị sửa đổi trái phép, secret có thể bị "leak". |
| Hỗ trợ che dấu (masking) tự động trong log. | Khó debug nếu việc inject bị lỗi (do giá trị bị che). |
| Quản lý tập trung, dễ dàng xoay vòng (rotate) key. | Phụ thuộc vào tính ổn định của GitHub Service. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| OIDC (OpenID Connect) | Không cần inject key dài hạn, tự động lấy token ngắn hạn (khuyến khích cho AWS/Azure/GCP). |
| Secret Managers (Vault/AWS) | Inject trực tiếp từ bên thứ ba vào ứng dụng lúc runtime thay vì lúc deploy. |

## 9. How
### Cách 1: Inject qua Environment Variables (Phổ biến nhất)
```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Start Application
        run: npm start
        env:
          DATABASE_URL: ${{ secrets.PROD_DATABASE_URL }}
          API_KEY: ${{ secrets.PROD_API_KEY }}
```

### Cách 2: Inject vào file `.env` trước khi build
```yaml
      - name: Create .env file
        run: |
          echo "DB_PASSWORD=${{ secrets.DB_PASSWORD }}" >> .env
          echo "SECRET_KEY=${{ secrets.SECRET_KEY }}" >> .env
```

### Cách 3: Sử dụng cho Docker Build
```yaml
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          build-args: |
            "API_TOKEN=${{ secrets.API_TOKEN }}"
```

## 10. Production concerns
### Scaling
Sử dụng **Environment-level Secrets** để đảm bảo khi deploy lên Production, workflow sẽ tự động lấy secret của Production mà không cần sửa code YAML.

### Failure
Nếu một secret bị thiếu hoặc rỗng, ứng dụng có thể crash khi khởi động. Nên có bước kiểm tra (validation) trước khi deploy:
```bash
if [ -z "$DATABASE_URL" ]; then echo "Error: DATABASE_URL is missing"; exit 1; fi
```

### Monitoring
GitHub tự động **Masking** các giá trị inject trong console log. Tuy nhiên, nếu bạn in secret dưới dạng JSON hoặc Base64, GitHub có thể không nhận diện được để che.

## 11. Common mistakes
- **Mistake**: Quên gán secret vào phần `env:` của job hoặc step, dẫn đến ứng dụng nhận giá trị `undefined`.
  **Fix**: Luôn kiểm tra scope của biến `env`. Nếu khai báo ở cấp Job, mọi Step đều thấy. Nếu ở cấp Step, chỉ Step đó thấy.

- **Mistake**: Inject secret vào ứng dụng Client-side (React/Vue) và tưởng rằng nó an toàn.
  **Fix**: Secret inject lúc build-time cho Frontend sẽ bị lộ trong mã nguồn trình duyệt. Chỉ inject các key công khai hoặc dùng Backend Proxy.

## 12. Sample project
Tạo một workflow deploy ứng dụng Node.js lên Heroku/AWS, sử dụng GitHub Environments để quản lý `DB_PASSWORD` khác nhau cho Staging và Production.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa việc inject secret qua `env` và gán trực tiếp vào tham số lệnh (CLI arguments)?
   A: Inject qua `env` an toàn hơn vì các biến môi trường thường không bị lưu lại trong lịch sử lệnh (command history) của hệ điều hành, trong khi CLI arguments có thể bị log lại bởi các công cụ giám sát tiến trình (như `ps -ef`).

2. Q: Làm thế nào để inject một file secret (ví dụ file `.p12` hoặc `.json` chứa private key)?
   A: Mã hóa file đó sang Base64, lưu chuỗi đó vào GitHub Secrets. Khi deploy, dùng lệnh `echo "${{ secrets.BASE64_FILE }}" | base64 -d > secret_file.json` để giải mã lại thành file vật lý.

### Scenario
**Tình huống**: Bạn cần inject 50 secrets vào ứng dụng. Việc viết 50 dòng trong file YAML quá dài và khó bảo trì. Bạn xử lý thế nào?
**Giải quyết**: Sử dụng một file JSON duy nhất chứa toàn bộ secrets (mã hóa JSON này vào 1 GitHub Secret) hoặc sử dụng Action chuyên dụng như `aws-actions/aws-secretsmanager-get-parameters` để fetch hàng loạt từ Secret Manager bên ngoài.

## 14. References
- Official Docs: [Using secrets in GitHub Actions](https://docs.github.com/en/actions/security-guides/encrypted-secrets#using-secrets-in-github-actions)
- Blog: [Best practices for secret management in CI/CD](https://github.blog/2022-04-12-git-security-best-practices-for-software-artifacts-and-secret-management/)

## 15. Real-world Code
N/A

## 16. Community
- Stack Overflow: Tag `github-actions-secrets`
- Talk: "CI/CD Security: How to keep your secrets secret" by GitHub.
