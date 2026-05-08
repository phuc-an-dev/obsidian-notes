---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/docker"
related:
  - "[[Docker Image.md]]"
  - "[[Docker Engine.md]]"
  - "[[Docker Daemon.md]]"
---

## 1. What
Container là một đơn vị phần mềm tiêu chuẩn, đóng gói mã nguồn và tất cả các phụ thuộc của nó để ứng dụng có thể chạy nhanh chóng và đáng tin cậy từ môi trường máy tính này sang môi trường máy tính khác. Một Docker Container là một thể hiện (instance) thực thi của một Docker Image.

## 2. Why
Trước khi có container, việc chạy ứng dụng phụ thuộc rất nhiều vào hệ điều hành host, dẫn đến lỗi "Works on my machine". Container ra đời để:
- **Cô lập (Isolation)**: Các ứng dụng chạy trong container không ảnh hưởng đến nhau và không ảnh hưởng đến host.
- **Tính di động (Portability)**: Chạy giống hệt nhau trên Laptop, Cloud, hay On-premise.
- **Hiệu quả tài nguyên**: Nhiều container có thể chạy trên cùng một Kernel OS, sử dụng ít RAM/CPU hơn so với Virtual Machines.

## 3. Mental Model
Hãy tưởng tượng Container giống như một **"Căn hộ chung cư cao cấp"**:
- **Image** là bản vẽ thiết kế của căn hộ.
- **Container** là căn hộ thực tế mà bạn đang ở.
- Tất cả các căn hộ (Container) đều sử dụng chung hệ thống móng và điện nước của tòa nhà (Host OS Kernel).
- Nhưng mỗi căn hộ hoàn toàn biệt lập, người ở phòng này không thể biết người ở phòng kia đang làm gì (Process Isolation).

## 4. Where it fits
Mối quan hệ trong hệ sinh thái:
`Docker Image (Tĩnh) -> docker run -> Docker Container (Động/Đang chạy)`

## 5. When to use
- Khi triển khai ứng dụng microservices.
- Khi cần chạy nhiều phiên bản của cùng một phần mềm (vd: MySQL 5.7 và MySQL 8.0) trên cùng một máy mà không xung đột.
- Khi xây dựng môi trường CI/CD để chạy các bản test trong một sandbox sạch sẽ.

## 6. When NOT to use
- Khi ứng dụng yêu cầu hiệu năng phần cứng cực hạn (như tính toán khoa học chuyên sâu) mà độ trễ ảo hóa nhẹ của container vẫn là một vấn đề.
- Khi ứng dụng có kiến trúc quá cũ, gắn chặt với các thành phần cứng đặc thù của Windows cũ hoặc Mainframe.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Khởi động trong vài giây. | Bảo mật kém hơn VM (vì dùng chung Kernel). |
| Tiết kiệm tài nguyên tối đa. | Dữ liệu bên trong container là tạm thời (Ephemeral) - sẽ mất nếu container bị xóa mà không dùng Volume. |
| Quản lý vòng đời dễ dàng (Stop/Start/Remove). | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Virtual Machine (VM) | Bảo mật cao hơn, cô lập hoàn toàn OS, nhưng nặng nề và khởi động chậm. |
| LXC (Linux Containers) | Giải pháp container cấp thấp hơn, Docker thực chất từng dựa trên LXC. |

## 9. How
Các lệnh quản lý Container phổ biến:

```bash
# 1. Tạo và chạy một container từ image
docker run -d --name my-web -p 80:80 nginx

# 2. Xem danh sách các container đang chạy
docker ps

# 3. Truy cập vào terminal của container
docker exec -it my-web bash

# 4. Xem log của container
docker logs -f my-web

# 5. Dừng và xóa container
docker stop my-web && docker rm my-web
```

## 10. Production concerns
### Statelessness
Container trên Production nên được thiết kế kiểu **Stateless**. Mọi dữ liệu quan trọng (User uploads, Database files) phải được lưu trữ ở ngoài container (AWS S3, Docker Volumes, hoặc External DB).

### Health Checks
Luôn định nghĩa `HEALTHCHECK` trong Dockerfile hoặc trong Orchestrator (K8s) để hệ thống tự động khởi động lại container nếu ứng dụng bên trong bị treo.

### Resource Limits
Luôn giới hạn CPU và RAM cho container để tránh một container bị lỗi "ăn" sạch tài nguyên của cả server (OOM Killer).

## 11. Common mistakes
- Mistake: Lưu dữ liệu quan trọng bên trong container.
  Fix: Sử dụng Docker Volumes (`-v`) hoặc Bind Mounts.

- Mistake: Chạy quá nhiều tiến trình (process) trong một container.
  Fix: Tuân thủ triết lý "One process per container".

## 12. Sample project
Thiết lập một cụm container:
1. Một container ứng dụng Node.js.
2. Một container Redis để làm cache.
3. Kết nối chúng qua Docker Network và kiểm tra sự cô lập.

## 13. Interview
### Core Q&A
1. Q: Trạng thái "Paused" và "Stopped" của container khác nhau thế nào?
   A: "Stopped" là tiến trình bên trong container đã bị hủy (SIGTERM/SIGKILL). "Paused" là tiến trình vẫn tồn tại trong RAM nhưng bị Kernel đóng băng (freezed), không được cấp CPU.

2. Q: Làm thế nào để container A liên lạc được với container B?
   A: Thông qua Docker Network. Khi nằm chung một network, các container có thể gọi nhau bằng tên container (Container Name) thay vì IP.

### Scenario
"Container của bạn báo lỗi OOM (Out of Memory) liên tục. Bạn xử lý thế nào?"
-> Trả lời:
1. Kiểm tra giới hạn RAM gán cho container (`docker inspect`).
2. Xem log ứng dụng để tìm lỗi rò rỉ bộ nhớ (Memory leak).
3. Nếu ứng dụng thực sự cần nhiều RAM hơn, tăng giới hạn `mem_limit`.
4. Sử dụng `docker stats` để theo dõi mức độ chiếm dụng theo thời gian thực.

## 14. References
- Official Docs: [What is a container?](https://www.docker.com/resources/what-container/)
- Runtime Specs: [OCI Container Spec](https://github.com/opencontainers/runtime-spec)

## 15. Real-world Code
Nghiên cứu cách Kubernetes quản lý các "Pods" (một nhóm các containers) để hiểu về quy trình vận hành container ở quy mô lớn.

## 16. Community
- Docker Captains.
- Reddit: r/docker.
- Stack Overflow: Tag [docker-container].
