---
created: 2026-05-07
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/java"
  - "#topic/performance"
related:
  - "[[tdd.md]]"
  - "[[Docker Daemon.md]]"
  - "[[Docker Engine.md]]"
---

## 1. What
Testcontainers Cloud là một dịch vụ được quản lý (managed service) giúp chạy các thư viện **Testcontainers** mà không cần phải có Docker Daemon cục bộ trên máy của nhà phát triển hoặc trên các server CI/CD. Thay vì khởi chạy các container trực tiếp trên máy bạn, Testcontainers Cloud sẽ khởi tạo chúng trên một hạ tầng đám mây được tối ưu hóa và kết nối ngược lại với ứng dụng test của bạn.

## 2. Why
Việc sử dụng Testcontainers truyền thống đòi hỏi máy phải chạy Docker, điều này dẫn đến một số vấn đề:
- **Tốn tài nguyên**: Khởi chạy nhiều container (DB, Message Broker) cùng lúc làm máy dev bị chậm và nóng.
- **Phức tạp trên CI/CD**: Các pipeline như GitHub Actions hoặc Jenkins yêu cầu cấu hình Docker-in-Docker (DinD) hoặc quyền root phức tạp và kém an toàn.
- **Tốc độ**: Testcontainers Cloud khởi chạy container cực nhanh nhờ hạ tầng được warm-up sẵn, giúp giảm thời gian chạy bộ test tích hợp (Integration tests).

## 3. Mental Model
Hãy tưởng tượng Testcontainers truyền thống giống như bạn **tự mua thực phẩm và tự nấu ăn tại bếp nhà mình** (Máy bạn chạy Docker).
- Bếp nhà bạn bị chật, nóng và tốn điện.
- Testcontainers Cloud giống như việc bạn **đặt món qua một nhà bếp công nghiệp (Cloud)**.
- Bạn chỉ cần đưa yêu cầu (Code test), nhà bếp công nghiệp sẽ chế biến món ăn (khởi chạy Container) và gửi kết quả về bàn ăn của bạn qua một đường ống chuyên dụng. Bạn không cần lo về việc lau dọn bếp hay tốn diện tích nhà.

## 4. Where it fits
Unit Tests (JUnit/Kotest) -> Testcontainers Library -> **Testcontainers Cloud Desktop App / Agent** -> **Remote Cloud Infrastructure** -> Containers Started.

## 5. When to use
- Khi máy tính cá nhân của bạn không đủ cấu hình mạnh để chạy Docker ổn định.
- Khi làm việc trong các tổ chức có chính sách bảo mật khắt khe không cho phép chạy Docker Desktop cục bộ.
- Khi muốn tăng tốc các pipeline CI/CD mà không muốn đau đầu với cấu hình Docker sockets.
- Khi cần chia sẻ chung một môi trường testing chuẩn hóa cho toàn bộ team.

## 6. When NOT to use
- Khi dự án của bạn chỉ có 1-2 container đơn giản và máy local vẫn đáp ứng tốt.
- Khi bạn làm việc trong môi trường hoàn toàn offline (Air-gapped) không có internet.
- Khi chi phí của dịch vụ cloud vượt quá ngân sách cho phép của dự án.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Giải phóng RAM/CPU cho máy local. | Phụ thuộc vào kết nối internet để giao tiếp với cloud. |
| Setup CI/CD cực kỳ đơn giản (không cần Docker). | Có chi phí sử dụng hàng tháng (SaaS model). |
| Hỗ trợ mượt mà trên cả Mac chip Intel và Apple Silicon. | Độ trễ mạng (latency) có thể ảnh hưởng nếu container truyền tải lượng dữ liệu cực lớn. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Local Docker Desktop | Miễn phí, offline, nhưng tốn tài nguyên máy. |
| Colima / Lima | Các giải pháp thay thế Docker Desktop trên Mac, nhẹ hơn nhưng vẫn chạy local. |
| GitHub Actions Service Containers | Tốt cho CI nhưng khó tái hiện giống hệt trên máy local của dev. |

## 9. How
Quy trình sử dụng Testcontainers Cloud:
1. Đăng ký tài khoản tại `testcontainers.cloud`.
2. Tải và cài đặt **Testcontainers Cloud Desktop app** (dành cho máy local).
3. Đăng nhập và app sẽ tự động thiết lập một "Docker-compatible socket".
4. Trong code Java/Spring Boot, bạn không cần thay đổi gì:
```java
@Container
static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:15-alpine");
```
Testcontainers thư viện sẽ tự nhận diện socket của Cloud Desktop và đẩy container lên mây thay vì chạy local.

## 10. Production concerns
### Security
Dữ liệu truyền tải giữa máy bạn và cloud được mã hóa. Các container trên cloud được cô lập hoàn toàn giữa các khách hàng khác nhau.

### Scaling
Hỗ trợ chạy song song (Parallel execution) hàng chục hoặc hàng trăm container cùng lúc mà không làm treo máy tính của bạn.

## 11. Common mistakes
- Mistake: Nghĩ rằng Testcontainers Cloud là một bộ thư viện mới.
  Fix: Nó chỉ là một **môi trường thực thi** (runtime environment) cho bộ thư viện Testcontainers hiện có của bạn.

- Mistake: Quên tắt Desktop app khi không sử dụng, dẫn đến việc container vẫn có thể được khởi tạo trên cloud khi bạn vô tình chạy test.
  Fix: Kiểm tra trạng thái app trên thanh menu hệ thống.

## 12. Sample project
Xây dựng một hệ thống vi dịch vụ (Microservices) với Spring Boot, Kafka và Redis. Sử dụng Testcontainers Cloud để chạy Integration Tests:
- Dev gõ `./mvnw test` trên laptop.
- Kafka và Redis khởi chạy trên Cloud.
- Kết quả test trả về console của Dev trong vòng vài giây.

## 13. Interview
### Core Q&A
1. Q: Testcontainers Cloud giải quyết vấn đề gì lớn nhất của Docker-in-Docker trên CI?
   A: Nó loại bỏ hoàn toàn nhu cầu về quyền `privileged` hoặc việc mount `/var/run/docker.sock`, giúp pipeline an toàn hơn và dễ dàng chạy trên bất kỳ CI runner nào (kể cả những runner không hỗ trợ Docker).

2. Q: Tôi có cần sửa code JUnit cũ để dùng Testcontainers Cloud không?
   A: Không. Testcontainers Cloud tương thích ngược hoàn toàn với mã nguồn hiện tại của bạn.

### Scenario
"Dự án của bạn có bộ test chạy 10 phút trên máy dev vì máy yếu. Bạn được cấp ngân sách để cải thiện. Bạn làm gì?"
-> Trả lời: Tôi sẽ đề xuất sử dụng Testcontainers Cloud. Nó sẽ chuyển gánh nặng tính toán sang hạ tầng cloud mạnh mẽ, cho phép chạy test song song tốt hơn và giải phóng tài nguyên máy dev để họ có thể làm việc khác trong lúc chờ test pass.

## 14. References
- Official Website: https://testcontainers.com/cloud/
- Documentation: https://www.testcontainers.com/cloud/docs/

## 15. Real-world Code
Nghiên cứu cách AtomicJar (công ty đứng sau Testcontainers) tích hợp dịch vụ này vào các dự án lớn của họ trên GitHub.

## 16. Community
- Slack: #testcontainers-cloud trên Testcontainers Slack.
- Blog: "Testcontainers Cloud is now GA" - AtomicJar Blog.
- Talk: "Testing without Docker on your machine" - re:Invent hoặc JavaOne sessions.
