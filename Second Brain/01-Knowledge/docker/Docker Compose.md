---
created: 2026-05-06
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/docker"
related:
  - "[[Dockerfile.md]]"
  - "[[Container in Docker.md]]"
  - "[[docker-compose-up-d.md]]"
  - "[[yaml.md]]"
---

## 1. What
Docker Compose là một công cụ giúp định nghĩa và chạy các ứng dụng Docker đa container (multi-container). Bạn sử dụng một tệp YAML (`docker-compose.yml`) để cấu hình các dịch vụ (services), mạng (networks) và ổ đĩa (volumes) của ứng dụng, sau đó chỉ cần một lệnh duy nhất để khởi động toàn bộ hệ thống.

## 2. Why
Trong thực tế, một ứng dụng hiếm khi chạy đơn lẻ. Nó thường cần: Web server, Database, Cache, Message Queue. Nếu dùng Docker CLI thuần, bạn phải gõ hàng chục lệnh dài dằng dặc, tự nhớ IP, tự kết nối network thủ công. Docker Compose ra đời để:
- **Quản lý tập trung**: Toàn bộ kiến trúc hạ tầng được mô tả trong 1 file duy nhất.
- **Dễ dàng chia sẻ**: Đồng nghiệp chỉ cần `git pull` và gõ 1 lệnh là có môi trường giống hệt bạn.
- **Tự động hóa kết nối**: Tự động tạo network và cho phép các container gọi nhau bằng tên service.

## 3. Mental Model
Hãy tưởng tượng Docker Compose giống như một **"Nhà thầu xây dựng tổng thể"**:
- Nếu Docker CLI là việc bạn tự đi thuê từng thợ điện, thợ nước, thợ xây và tự chỉ họ cách làm việc với nhau.
- Thì Docker Compose là một bản hợp đồng ký với nhà thầu. Bạn chỉ cần ghi rõ trong bản hợp đồng (file YAML): "Tôi muốn 1 phòng khách (Web), 1 phòng bếp (DB), chúng phải thông nhau". Nhà thầu sẽ tự lo liệu việc gọi thợ và kết nối mọi thứ cho bạn chỉ sau một cái gật đầu của bạn.

## 4. Where it fits
Vị trí trong luồng phát triển:
`docker-compose.yml (Definition) -> docker-compose up -> Orchestrated Containers (Running System)`

## 5. When to use
- Môi trường phát triển local (Local Development) cho các ứng dụng có nhiều thành phần.
- Chạy các bộ test tích hợp (Integration Tests) yêu cầu database thật.
- Các dự án quy mô nhỏ hoặc trung bình trên một máy chủ duy nhất (Single-host deployment).

## 6. When NOT to use
- Triển khai trên cụm máy chủ lớn (Multi-host cluster) - trường hợp này nên dùng Kubernetes hoặc Docker Swarm.
- Khi ứng dụng của bạn cực kỳ đơn giản, chỉ có duy nhất 1 container và không cần cấu hình gì phức tạp.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Cực kỳ dễ học và sử dụng. | Chỉ hoạt động trên một máy chủ duy nhất (mặc định). |
| Khởi động toàn bộ hệ thống chỉ với `docker-compose up`. | Không có các tính năng cao cấp như Auto-healing hay Rolling Update mạnh mẽ như K8s. |
| Quản lý biến môi trường và volume rất tiện lợi. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Kubernetes (K8s) | Tiêu chuẩn cho Production, cực mạnh nhưng độ dốc học tập rất cao. |
| Docker Swarm | Tích hợp sẵn trong Docker, hỗ trợ đa máy chủ, đơn giản hơn K8s nhưng ít tính năng hơn. |
| Podman Compose | Giải pháp tương tự dành cho người dùng Podman. |

## 9. How
Ví dụ file `docker-compose.yml` cho ứng dụng Spring Boot + MySQL:

```yaml
version: '3.8'
services:
  db:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: mydb
    ports:
      - "3306:3306"
    volumes:
      - db_data:/var/lib/mysql

  app:
    build: .
    ports:
      - "8080:8080"
    depends_on:
      - db
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://db:3306/mydb

volumes:
  db_data:
```

Các lệnh cơ bản:
- `docker-compose up -d`: Khởi động toàn bộ system ở chế độ background.
- `docker-compose down`: Dừng và xóa toàn bộ container, network được tạo.
- `docker-compose ps`: Xem trạng thái các service trong file compose.
- `docker-compose logs -f app`: Xem log của 1 service cụ thể.

## 10. Production concerns
### Restart Policy
Luôn sử dụng `restart: always` hoặc `unless-stopped` để đảm bảo container tự khởi động lại nếu bị crash hoặc server reboot.

### Profiles
Dùng `profiles` để phân tách các service chỉ dùng cho dev (như Adminer, Swagger UI) và các service cho production.

### Env Files
Sử dụng tham số `env_file` để tách biệt các thông tin nhạy cảm ra khỏi file YAML chính.

## 11. Common mistakes
- Mistake: Hardcode mật khẩu database trực tiếp trong file YAML và commit lên Git.
  Fix: Dùng biến môi trường `${VAR_NAME}` và file `.env`.

- Mistake: Quên dùng Volume cho database.
  Fix: Luôn định nghĩa `volumes` để giữ lại dữ liệu khi container bị xóa.

## 12. Sample project
Thiết lập một stack hoàn chỉnh bao gồm:
1. Nginx làm Reverse Proxy.
2. Web App (Node.js/Spring Boot).
3. PostgreSQL Database.
4. PGAdmin để quản lý DB.
Tất cả kết nối với nhau qua một mạng nội bộ chung.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa `docker-compose up` và `docker-compose start` là gì?
   A: `up` sẽ tạo mới hoặc build lại container nếu có thay đổi cấu hình. `start` chỉ khởi động lại các container đã tồn tại nhưng đang bị dừng (stop).

2. Q: Thuộc tính `depends_on` có đảm bảo ứng dụng chỉ chạy khi Database đã sẵn sàng hoàn toàn không?
   A: Không. Nó chỉ đảm bảo container DB được khởi động trước. Để đảm bảo DB đã sẵn sàng nhận kết nối, bạn cần dùng các công cụ như `wait-for-it.sh` hoặc cơ chế retry trong ứng dụng.

### Scenario
"Làm thế nào để scale một service cụ thể (ví dụ tăng lên 3 container cho app) bằng Docker Compose?"
-> Trả lời: Tôi sử dụng lệnh `docker-compose up --scale app=3`. Lưu ý: Service được scale không nên map cứng vào một port host cố định, hoặc phải dùng Load Balancer đứng trước.

## 14. References
- Official Guide: [Docker Compose overview](https://docs.docker.com/compose/)
- YAML Spec: [Compose file reference](https://docs.docker.com/compose/compose-file/)

## 15. Real-world Code
Nghiên cứu file `docker-compose.yml` của dự án **Sentry** hoặc **Appwrite** trên GitHub để thấy cách họ quản lý hàng chục microservices cùng lúc.

## 16. Community
- Reddit: r/docker.
- Docker Community Slack (#compose channel).
- Stack Overflow: Tag [docker-compose].
