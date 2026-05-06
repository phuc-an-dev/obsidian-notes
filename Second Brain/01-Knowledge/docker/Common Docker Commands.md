---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/docker"
related:
  - "[[Docker Image.md]]"
  - "[[Docker in Ubuntu.md]]"
---

## 1. What
Common Docker Commands là danh sách các lệnh CLI (Command Line Interface) thiết yếu nhất để tương tác với Docker Engine. Các lệnh này bao quát toàn bộ vòng đời của một container, từ việc quản lý image, vận hành container cho đến kiểm tra hệ thống và dọn dẹp tài nguyên.

## 2. Why
Docker có hàng trăm tập lệnh và tùy chọn khác nhau. Việc ghi nhớ toàn bộ là không khả thi và không cần thiết. Tập trung vào các lệnh sử dụng nhiều nhất giúp lập trình viên tăng tốc độ làm việc, xử lý nhanh các tình huống thực tế và xây dựng được nền tảng vững chắc để học các kỹ năng nâng cao như Orchestration (Kubernetes).

## 3. Mental Model
Hãy tưởng tượng Docker CLI giống như một **"Bảng điều khiển máy bay"**.
- Có hàng trăm nút bấm, nhưng phi công chủ yếu sử dụng khoảng 20 nút quan trọng nhất để cất cánh, hạ cánh và kiểm tra thông số.
- Việc nắm vững 20 nút này giúp bạn điều khiển được toàn bộ chuyến bay (Container lifecycle) một cách an toàn.

## 4. Where it fits
Vị trí trong quy trình:
`Lập trình viên -> Docker CLI -> Docker Daemon -> Docker Objects (Images, Containers, Volumes, Networks)`

## 5. When to use
- Trong quá trình phát triển (Local Development) để build và test ứng dụng.
- Khi debug lỗi trên server Production.
- Khi viết các script tự động hóa CI/CD.

## 6. When NOT to use
- Khi hệ thống đã chuyển sang sử dụng các công cụ Orchestration cao cấp (như Kubernetes), bạn nên dùng `kubectl` thay vì dùng các lệnh `docker` trực tiếp trên từng node.
- Khi sử dụng các giao diện quản lý đồ họa (GUI) như Portainer hoặc Docker Desktop (nếu bạn không thích dùng Terminal).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Kiểm soát chi tiết và mạnh mẽ nhất đối với Docker Engine. | Đòi hỏi phải ghi nhớ cú pháp và các tham số. |
| Có sẵn trên mọi môi trường hỗ trợ Docker. | Dễ gây lỗi hệ thống nếu gõ sai lệnh (ví dụ xóa nhầm volume). |
| Dễ dàng tích hợp vào script tự động hóa. | Giao diện text-only có thể gây khó khăn cho người mới. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Docker Compose | Sử dụng file YAML để quản lý nhiều lệnh docker cùng lúc. |
| Portainer | Giao diện Web trực quan để quản lý Docker mà không cần gõ lệnh. |
| Docker Desktop GUI | Phù hợp cho người dùng Windows/macOS thích thao tác chuột. |

## 9. How
Dưới đây là các nhóm lệnh quan trọng nhất:

### Quản lý Images
```bash
docker build -t name:tag .    # Build image từ Dockerfile
docker images                 # Liệt kê các image hiện có
docker rmi image_id           # Xóa một image
docker pull image_name        # Tải image từ registry
```

### Quản lý Containers
```bash
docker run -d --name app name # Chạy container ở chế độ background
docker ps                     # Liệt kê các container đang chạy (-a để xem tất cả)
docker stop container_id      # Dừng container
docker start container_id     # Khởi động lại container đã dừng
docker rm -f container_id     # Xóa container (cưỡng chế)
docker exec -it app bash      # Truy cập vào terminal bên trong container
```

### Kiểm tra và Debug
```bash
docker logs -f container_id   # Xem log realtime của container
docker inspect container_id   # Xem cấu hình chi tiết dạng JSON
docker stats                  # Xem mức độ chiếm dụng tài nguyên (CPU, RAM)
```

### Dọn dẹp hệ thống
```bash
docker system prune           # Xóa sạch container/network/dangling image
docker volume prune           # Xóa sạch các volume không dùng đến
```

## 10. Production concerns
### Safety
Trên Production, hãy cực kỳ cẩn thận với lệnh `docker rm -f` và `docker system prune`. Luôn kiểm tra kỹ `docker ps` trước khi thực hiện hành động xóa.

### Logging
Sử dụng `docker logs --tail 100` thay vì load toàn bộ log để tránh làm treo terminal nếu file log quá lớn.

### Monitoring
Lệnh `docker stats` là cứu cánh đầu tiên khi server có dấu hiệu quá tải để xác định container nào đang "ăn" nhiều RAM/CPU nhất.

## 11. Common mistakes
- Mistake: Quên tham số `-it` khi dùng `docker exec`.
  Fix: Luôn dùng `-it` (interactive + tty) để có thể tương tác với bash/sh bên trong container.

- Mistake: Xóa container mà không biết dữ liệu bên trong sẽ mất.
  Fix: Kiểm tra xem container có dùng volume không trước khi xóa.

## 12. Sample project
Tạo một kịch bản:
1. Pull image `nginx:alpine`.
2. Chạy nó ở port 8080.
3. Chỉnh sửa file `index.html` bên trong bằng lệnh `exec`.
4. Xem log truy cập.
5. Cuối cùng là dọn dẹp sạch sẽ.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa `docker stop` và `docker kill` là gì?
   A: `docker stop` gửi tín hiệu SIGTERM để container tắt một cách êm ái (Graceful shutdown). `docker kill` gửi tín hiệu SIGKILL để buộc container dừng ngay lập tức.

2. Q: Làm thế nào để xóa toàn bộ các container đang chạy?
   A: Sử dụng lệnh: `docker rm -f $(docker ps -aq)`.

### Scenario
"Container của bạn đang chạy nhưng không thể truy cập qua trình duyệt. Bạn dùng những lệnh nào để kiểm tra?"
-> Trả lời:
1. `docker ps` để xem port mapping có đúng không.
2. `docker logs` để xem ứng dụng bên trong có báo lỗi gì không.
3. `docker inspect` để kiểm tra IPAddress và Network settings.
4. `docker exec` để chui vào trong ping thử ra ngoài internet hoặc kiểm tra port nội bộ.

## 14. References
- Official Cheat Sheet: [Docker CLI Cheat Sheet](https://docs.docker.com/get-started/docker_cheatsheet.pdf)
- Full Reference: [Docker CLI reference](https://docs.docker.com/engine/reference/commandline/cli/)

## 15. Real-world Code
Nhiều công ty sử dụng các "Alias" trong `.bashrc` hoặc `.zshrc` để gõ lệnh nhanh hơn:
`alias dps="docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'"`

## 16. Community
- Reddit: r/docker
- Stack Overflow: Tag [docker]
- Các bài viết "Docker Cheat Sheet" trên Dev.to hoặc Medium.
