---
created: 2026-05-07
tags:
  - "#type/tutorial"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/docker"
related:
  - "[[Docker Compose.md]]"
  - "[[Common Docker Commands.md]]"
---

## 1. What
`docker compose up -d` là câu lệnh được sử dụng để xây dựng (build), tạo mới (create), khởi động (start) và gắn (attach) vào các container cho một dịch vụ, nhưng với cờ `-d` (detached mode), các container sẽ chạy dưới nền (background). Điều này cho phép terminal của bạn được giải phóng ngay sau khi các lệnh khởi động được gửi đi.

## 2. Why
Khi chạy `docker compose up` (chế độ mặc định - foreground), toàn bộ logs của tất cả các container sẽ đổ dồn vào terminal. Nếu bạn tắt terminal hoặc nhấn `Ctrl+C`, các container cũng sẽ bị dừng lại. `up -d` giải quyết vấn đề này bằng cách:
- Cho phép dịch vụ chạy bền bỉ dưới dạng background process.
- Giúp bạn tiếp tục gõ các lệnh khác trên cùng một terminal.
- Phù hợp cho môi trường Production hoặc khi phát triển các dịch vụ backend không cần theo dõi log liên tục.

## 3. Mental Model
Hãy tưởng tượng việc khởi động dịch vụ giống như **gọi món tại nhà hàng**:
- `up` (foreground): Bạn đứng ngay tại quầy chờ lấy món. Bạn thấy đầu bếp làm gì (logs) và bạn không thể đi đâu khác cho đến khi xong.
- `up -d` (detached): Bạn gọi món qua app rồi đi làm việc khác. Nhà hàng (Docker) tự lo việc chế biến dưới bếp (background). Khi nào muốn biết món ăn xong chưa, bạn mới mở app ra kiểm tra (docker logs).

## 4. Where it fits
Terminal -> `docker compose up -d` -> Docker Engine (Daemon) -> Containers Running in Background.

## 5. When to use
- Khi chạy các stack dịch vụ ổn định (Database, Cache, Proxy).
- Trong các script tự động hóa triển khai (CI/CD).
- Khi bạn muốn chạy nhiều project Docker cùng lúc trên một máy.

## 6. When NOT to use
- Khi bạn mới cấu hình xong `docker-compose.yml` và cần debug lỗi khởi động (startup errors). Ở chế độ `-d`, bạn sẽ không thấy lỗi ngay lập tức nếu container bị crash lúc startup.
- Khi bạn cần quan sát log realtime để theo dõi luồng xử lý của ứng dụng đang phát triển.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Giải phóng terminal, chạy bền bỉ. | Khó phát hiện lỗi khởi động ngay lập tức. |
| Dễ dàng tích hợp vào quy trình vận hành. | Phải dùng thêm lệnh `docker compose logs` để xem chuyện gì đang xảy ra. |
| Không bị tắt container khi ngắt kết nối SSH/Terminal. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `docker compose up` | Chạy ở foreground, xem log trực tiếp, dễ debug. |
| `docker compose start` | Chỉ khởi động lại các container đã tồn tại, không tạo mới hay cập nhật cấu hình. |
| `docker compose run` | Chạy một lệnh cụ thể trong một service mới. |

## 9. How
Khởi chạy dịch vụ ở chế độ detached:
```bash
docker compose up -d
```

Nếu bạn thay đổi nội dung Dockerfile và muốn build lại trước khi chạy:
```bash
docker compose up -d --build
```

Để xem logs của các container đang chạy dưới nền:
```bash
docker compose logs -f
```

Dừng và xóa các container đang chạy ở chế độ detached:
```bash
docker compose down
```

## 10. Production concerns
### Restart Policies
Trong file `docker-compose.yml`, luôn đi kèm với `restart: always` để đảm bảo khi daemon khởi động lại hoặc container bị crash, nó sẽ tự động chạy lại ở chế độ detached.

### Monitoring
Vì không thấy log trực tiếp, bạn cần một hệ thống giám sát (như Prometheus, Grafana hoặc đơn giản là Portainer) để theo dõi trạng thái các container "chạy ngầm" này.

## 11. Common mistakes
- Mistake: Sửa file `docker-compose.yml` rồi chạy `docker compose up -d` nhưng thấy container cũ vẫn chạy (do không có thay đổi đáng kể về cấu hình service).
  Fix: Dùng `--force-recreate` nếu muốn ép buộc tạo mới container.

- Mistake: Chạy `up -d` và thấy terminal báo "Done" nhưng thực tế ứng dụng bên trong bị lỗi và thoát ngay lập tức.
  Fix: Luôn kiểm tra lại bằng `docker compose ps` sau khi chạy `up -d`.

## 12. Sample project
Chạy một cụm WordPress nhanh gọn:
1. Tạo file `docker-compose.yml` với dịch vụ `db` (MySQL) và `wordpress`.
2. Gõ `docker compose up -d`.
3. Truy cập `localhost:8080` để cài đặt mà không bị vướng bận bởi hàng dài logs trong terminal.

## 13. Interview
### Core Q&A
1. Q: Làm thế nào để "chui" vào lại một container đang chạy ở chế độ detached?
   A: Sử dụng lệnh `docker compose exec <service_name> bash` hoặc `sh`.

2. Q: Cờ `-d` viết tắt của từ gì?
   A: Detached (Tách rời).

### Scenario
"Tôi chạy `docker compose up -d` nhưng website không lên. Tôi phải làm gì?"
-> Trả lời:
1. Chạy `docker compose ps` để xem container có trạng thái `Up` hay `Exited`.
2. Nếu `Exited`, chạy `docker compose logs <service_name>` để xem lỗi khởi động.
3. Nếu vẫn không rõ, thử chạy lại ở foreground bằng `docker compose up` (không có `-d`) để quan sát trực tiếp.

## 14. References
- Docker Docs: [docker compose up reference](https://docs.docker.com/engine/reference/commandline/compose_up/)
- Detached vs Foreground: [Docker documentation](https://docs.docker.com/engine/reference/run/#detached--d)

## 15. Real-world Code
Trong các file `Makefile` của dự án, người ta thường đặt lệnh:
```make
start:
	docker compose up -d
stop:
	docker compose down
```

## 16. Community
- Stack Overflow: [How to run docker-compose up in background?](https://stackoverflow.com/questions/33066528/)
- Reddit: r/docker - thảo luận về việc quản lý logs khi dùng detached mode.
