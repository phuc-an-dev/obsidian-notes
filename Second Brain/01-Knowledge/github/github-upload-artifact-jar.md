---
created: 2026-05-05
tags:
  - "#type/pattern"
  - "#status/draft"
  - "#lang/java"
  - "#topic/devops"
related:
  - "[[github-manual-deployment]]"
  - "[[github-secret-injection]]"
---

## 1. What
Upload JAR Artifact là quá trình lưu trữ tệp tin `.jar` (kết quả của quá trình build Java) lên hệ thống lưu trữ tạm thời của GitHub Actions. Tệp tin này sau đó có thể được tải xuống bởi các Job khác trong cùng một workflow hoặc bởi người dùng để kiểm tra thủ công.

## 2. Why
Việc upload artifact mang lại các lợi ích quan trọng:
- **Build Once, Run Anywhere**: Đảm bảo tệp tin được deploy chính là tệp tin đã qua các bước kiểm tra (unit test, linting). Tránh việc phải build lại code ở mỗi job, gây tốn thời gian và rủi ro sai lệch version.
- **Tách biệt Job (Separation of Concerns)**: Chia workflow thành các Job nhỏ: `Build` (trên môi trường mạnh) và `Deploy` (trên môi trường có quyền truy cập server).
- **Lưu trữ bằng chứng (Audit Trail)**: Giữ lại bản build để có thể tải về kiểm tra nếu quá trình deploy gặp sự cố.

## 3. Mental Model
Hãy tưởng tượng quy trình này giống như một **"Cuộc đua tiếp sức"**:
- Vận động viên thứ nhất (Build Job) chạy và tạo ra một "Cây gậy" (JAR file).
- Anh ta không thể chạy tiếp vào khu vực cấm (Server), nên anh ta đặt cây gậy vào một **"Ngăn tủ chung"** (GitHub Artifact Storage).
- Vận động viên thứ hai (Deploy Job) đến ngăn tủ, lấy đúng cây gậy đó ra và tiếp tục chạy về đích (Deployment).

## 4. Where it fits
`Checkout -> JDK Setup -> Maven/Gradle Build -> Upload Artifact -> [Download Artifact in next job] -> Deploy`

## 5. When to use
- Các dự án Java (Spring Boot, Micronaut, Quarkus).
- Cần chuyển tệp build từ Job này sang Job khác.
- Muốn lưu lại bản build sau khi workflow kết thúc để QA tải về test.

## 6. When NOT to use
- Dự án nhỏ chỉ có 1 Job duy nhất: Có thể deploy trực tiếp sau khi build mà không cần upload.
- Lưu trữ lâu dài (tháng/năm): Artifact chỉ nên tồn tại ngắn hạn. Dùng GitHub Releases hoặc Docker Registry (GHCR) cho việc lưu trữ lâu dài.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Đảm bảo tính nhất quán của file deploy. | Tốn thời gian upload/download (đặc biệt với JAR nặng). |
| Giúp workflow sạch sẽ và dễ debug hơn. | Giới hạn dung lượng lưu trữ miễn phí của GitHub. |
| Hỗ trợ workflow phức tạp (Parallel jobs). | Cần quản lý thời gian hết hạn (Retention days). |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Docker Image | Tốt hơn cho môi trường Container (K8s), đóng gói cả OS. JAR artifact chỉ là file code. |
| GitHub Releases | Dùng cho phiên bản chính thức (Production release). |
| Maven/Artifactory | Dùng trong nội bộ doanh nghiệp lớn để quản lý dependency. |

## 9. How
### Step 1: Upload Artifact trong Build Job
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
      - name: Build with Maven
        run: mvn clean package -DskipTests
      
      - name: Upload JAR
        uses: actions/upload-artifact@v4
        with:
          name: my-app-jar
          path: target/*.jar
          retention-days: 1 # Chỉ giữ 1 ngày để tiết kiệm bộ nhớ
```

### Step 2: Download Artifact trong Deploy Job
```yaml
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - name: Download JAR
        uses: actions/download-artifact@v4
        with:
          name: my-app-jar
          path: deploy-folder
      
      - name: List files
        run: ls -R deploy-folder
```

## 10. Production concerns
### Scaling
- Với dự án Monolith lớn, file JAR có thể lên tới vài trăm MB. Nên nén (Zip) hoặc chỉ upload những gì thực sự cần thiết.

### Failure
- Nếu Job Build fail, Job Deploy sẽ không chạy (nhờ `needs: build`).
- Nếu artifact bị xóa sớm (hết hạn), Job Deploy sẽ báo lỗi "Artifact not found".

### Monitoring
- Xem dung lượng artifact đã sử dụng trong `Settings -> Billing and plans -> Actions`.

## 11. Common mistakes
- **Mistake**: Dùng sai version của action. `upload-artifact@v4` không tương thích với `download-artifact@v3`.
  **Fix**: Luôn dùng cùng một version (v4) cho cả upload và download.

- **Mistake**: Không chỉ định đúng `path`. Ví dụ: `path: target/app.jar` nhưng Maven lại tạo ra `target/app-1.0-SNAPSHOT.jar`.
  **Fix**: Dùng wildcard như `path: target/*.jar`.

## 12. Sample project
Tạo một Multi-stage workflow:
1. `Job 1`: Build Maven, Upload JAR.
2. `Job 2`: Chạy Integration Test trên JAR vừa tải về.
3. `Job 3`: Deploy JAR lên server thông qua SSH.

## 13. Interview
### Core Q&A
1. Q: Tại sao không build lại code ở Job Deploy cho nhanh?
   A: Vì việc build lại có thể tạo ra mã bytecode khác (do thay đổi nhỏ trong môi trường hoặc dependency). Việc dùng chung một artifact đảm bảo 100% tính nhất quán.

2. Q: Làm thế nào để truyền tệp tin giữa các Job khác nhau nhưng trong các Workflow khác nhau?
   A: Artifact mặc định chỉ nằm trong 1 workflow run. Để truyền giữa các workflow, cần dùng `actions/download-artifact` với `run_id` cụ thể hoặc dùng GitHub API.

### Scenario
**Tình huống**: Bạn có 5 Jobs chạy song song cần dùng chung tệp JAR để test trên các hệ điều hành khác nhau. Bạn sẽ làm gì?
**Giải quyết**: Build JAR ở một Job khởi đầu, `upload-artifact`. Cả 5 Job kia đều `needs` Job build này và đồng thời `download-artifact`.

## 14. References
- Official Docs: [Storing workflow data as artifacts](https://docs.github.com/en/actions/using-workflows/storing-workflow-data-as-artifacts)
- Action Repo: [actions/upload-artifact](https://github.com/actions/upload-artifact)

## 15. Real-world Code
N/A

## 16. Community
- Stack Overflow: Tag `github-actions-artifacts`
- Blog: `Optimizing GitHub Actions storage`
