---
created: 2026-05-07
tags:
  - "#type/pattern"
  - "#status/draft"
  - "#lang/others"
  - "#topic/devops"
related:
  - "[[github-actions-ci.md]]"
  - "[[@SpringBootTest in Spring Boot.md]]"
  - "[[MySQL on EC2 vs AWS RDS.md]]"
---

## 1. What
GitHub Actions Service Containers là một tính năng cho phép bạn khởi chạy các container phụ trợ (services) như MySQL, Redis, hoặc Postgres ngay bên cạnh job chính trong quy trình CI/CD. Cụ thể, MySQL service container cung cấp một instance cơ sở dữ liệu MySQL tạm thời để phục vụ cho việc chạy integration tests.

## 2. Why
Khi chạy automated tests trên GitHub Actions, ứng dụng của bạn thường cần kết nối tới một database thật để kiểm tra tính đúng đắn của các câu lệnh SQL và logic nghiệp vụ. Sử dụng Service Containers giúp:
- **Tự động hóa**: Database được tạo ra và xóa đi cùng với vòng đời của job.
- **Cô lập**: Mỗi job có một database instance riêng sạch sẽ, không bị ảnh hưởng bởi các job khác.
- **Tiện lợi**: Không cần cài đặt MySQL thủ công lên runner bằng `apt-get`.

## 3. Mental Model
Hãy tưởng tượng job chính của bạn (nơi chạy code Java/Node.js) là một **"Người thợ xây"**.
- MySQL service container giống như một **"Xe trộn bê tông"** đỗ ngay sát công trường.
- Người thợ xây chỉ cần với tay ra (gọi qua `localhost`) là có ngay bê tông (dữ liệu) để dùng.
- Khi thợ xây làm xong việc và ra về, chiếc xe trộn cũng tự động rời đi, trả lại mặt bằng sạch sẽ.

## 4. Where it fits
GitHub Runner -> **Main Job (Steps)** --(Network)---> **MySQL Service Container**.
Trong GitHub Actions, các service containers được nối với nhau qua cùng một Docker network.

## 5. When to use
- Chạy integration tests yêu cầu database thật.
- Kiểm tra tính tương thích của các bản script migrate database (Flyway, Liquibase).
- Khi bạn muốn môi trường test giống hệt production (cùng version MySQL).

## 6. When NOT to use
- **Unit Test thuần túy**: Nên dùng Mocking để đạt tốc độ cao nhất.
- Khi đã sử dụng **Testcontainers** trong mã nguồn (Testcontainers cũng khởi chạy Docker nhưng linh hoạt hơn trong việc quản lý cấu hình từ code).
- Các ứng dụng cực kỳ nhỏ mà H2 Database (In-memory) là đủ.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Dễ cấu hình qua file YAML đơn giản. | Làm tăng thời gian khởi động Job (chờ tải image và database sẵn sàng). |
| Luôn bắt đầu với database sạch. | Khó tùy chỉnh cấu hình database phức tạp so với Testcontainers. |
| Không tốn thêm chi phí (nằm trong tài nguyên runner). | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| H2 / SQLite | Nhanh hơn nhưng không hỗ trợ các tính năng đặc thù của MySQL (JSON, Stored Procs). |
| Testcontainers | Mạnh mẽ hơn, quản lý từ code Java/Node, nhưng yêu cầu runner hỗ trợ Docker. |
| External Dev DB | DB dùng chung, dễ bị xung đột dữ liệu giữa các lần chạy test song song. |

## 9. How
Cấu hình MySQL Service trong file `.github/workflows/main.yml`:

```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: root
          MYSQL_DATABASE: test_db
        ports:
          - 3306:3306
        options: >-
          --health-cmd="mysqladmin ping"
          --health-interval=10s
          --health-timeout=5s
          --health-retries=3

    steps:
      - uses: actions/checkout@v3
      - name: Run tests
        env:
          DB_URL: jdbc:mysql://localhost:3306/test_db
        run: ./mvnw test
```

## 10. Production concerns
### Health Checks
Luôn sử dụng `options` với `health-cmd` để đảm bảo MySQL đã khởi động xong trước khi Job chính bắt đầu chạy code test. Nếu không, bạn sẽ gặp lỗi "Connection Refused".

### Port Mapping
Mặc định GitHub Actions sẽ map port vào host runner. Nếu bạn chạy job trên container (`container: ...`), bạn nên gọi service qua hostname (ví dụ: `mysql:3306`) thay vì `localhost`.

## 11. Common mistakes
- Mistake: Quên cấu hình biến môi trường `MYSQL_ROOT_PASSWORD`, dẫn đến container mysql bị thoát ngay lập tức.
  Fix: Luôn khai báo `env` cho service.

- Mistake: Chạy test ngay khi container vừa start mà chưa kịp sẵn sàng nhận kết nối.
  Fix: Sử dụng `health-cmd` hoặc thêm một step `sleep 10`.

## 12. Sample project
Xây dựng workflow cho ứng dụng Spring Boot:
1. Định nghĩa service `mysql:8.0`.
2. Truyền các biến `SPRING_DATASOURCE_URL`, `USERNAME`, `PASSWORD` vào bước chạy test.
3. Sử dụng Flyway để tự động tạo table trên service container này trước khi test logic.

## 13. Interview
### Core Q&A
1. Q: Làm thế nào để kết nối tới MySQL service từ job chính?
   A: Nếu job chạy trực tiếp trên host runner (`ubuntu-latest`), kết nối qua `localhost:3306`. Nếu job chạy trong một container khác, kết nối qua hostname của service (ví dụ `mysql:3306`).

2. Q: Có thể chạy nhiều service container cùng lúc không?
   A: Có. Bạn có thể định nghĩa thêm Redis, Postgres, MongoDB trong cùng mục `services`.

### Scenario
"Workflow của bạn thi thoảng bị lỗi ngẫu nhiên khi kết nối tới DB ở những giây đầu tiên. Bạn xử lý thế nào?"
-> Trả lời: Tôi sẽ thêm phần `options` cho service để định nghĩa `healthcheck`. GitHub Actions sẽ tự động chờ cho đến khi lệnh `health-cmd` trả về thành công thì mới bắt đầu các `steps` của job.

## 14. References
- GitHub Docs: [About service containers](https://docs.github.com/en/actions/using-containerized-services/about-service-containers)
- GitHub Docs: [Creating MySQL service containers](https://docs.github.com/en/actions/using-containerized-services/creating-mysql-service-containers)

## 15. Real-world Code
Nghiên cứu các file YAML của các dự án Open Source lớn như **PrestaShop** hoặc **Laravel** để xem cách họ cấu hình ma trận test với nhiều phiên bản database khác nhau.

## 16. Community
- GitHub Community Forum: Actions category.
- Stack Overflow: Tag [github-actions] [mysql].
