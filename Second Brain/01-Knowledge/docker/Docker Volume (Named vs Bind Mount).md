---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/docker"
related:
  - "[[Docker Compose.md]]"
  - "[[Container in Docker.md]]"
---

## 1. What
Docker Volume và Bind Mount là hai cơ chế chính để lưu trữ dữ liệu bền vững (persistence) bên ngoài vòng đời của container.
- **Named Volume**: Dữ liệu được Docker quản lý hoàn toàn trong một thư mục riêng biệt trên host (`/var/lib/docker/volumes/`).
- **Bind Mount**: Gắn một thư mục hoặc file cụ thể bất kỳ trên máy host vào một đường dẫn trong container.

## 2. Why
Mặc định, dữ liệu bên trong container là tạm thời (ephemeral) và sẽ mất sạch khi container bị xóa. Việc sử dụng Volume giúp:
- Lưu trữ dữ liệu quan trọng (Database, logs, user uploads) một cách an toàn.
- Chia sẻ dữ liệu giữa các container khác nhau.
- Tách biệt logic ứng dụng (trong container) và dữ liệu (ngoài container) để dễ dàng nâng cấp hoặc di chuyển.

## 3. Mental Model
Hãy tưởng tượng Container là một **"Chiếc xe hơi đi thuê"**:
- **Named Volume**: Giống như việc bạn thuê một cái kho tại bãi đậu xe của công ty cho thuê. Họ quản lý kho đó, đảm bảo an toàn và bạn có thể lấy đồ ra dùng cho bất kỳ chiếc xe nào bạn thuê sau này.
- **Bind Mount**: Giống như việc bạn mang theo chiếc vali cá nhân của mình và đặt nó vào cốp xe. Bạn tự quản lý vali đó, biết chính xác nó nằm ở đâu trong nhà mình (host path).

## 4. Where it fits
Vị trí trong kiến trúc lưu trữ:
`Container Filesystem -> Docker Storage Driver (Thin pool) -> Mount Point (Volume/Bind Mount) -> Host Disk`

## 5. When to use
- **Named Volume**: Khuyên dùng cho hầu hết các trường hợp trên Production (Database, application data). An toàn hơn vì Docker quản lý quyền hạn và không phụ thuộc vào cấu trúc thư mục của host.
- **Bind Mount**: Dùng cho môi trường Development (gắn source code từ máy vào container để hot-reload) hoặc khi cần truy cập các file cấu hình hệ thống của host.

## 6. When NOT to use
- Khi dữ liệu thực sự chỉ mang tính tạm thời (như cache xử lý ảnh trong một request).
- Tránh dùng Bind Mount trên Production nếu có thể, vì nó tạo ra sự phụ thuộc chặt chẽ vào cấu trúc thư mục của máy chủ vật lý, gây khó khăn khi scale-out.

## 7. Trade-offs
| Feature | Named Volume | Bind Mount |
|---------|--------------|------------|
| **Quản lý** | Docker quản lý | Người dùng quản lý |
| **Tính di động** | Cao (dễ backup/migrate) | Thấp (phụ thuộc đường dẫn host) |
| **Hiệu năng** | Cao nhất trên Docker Desktop | Cao |
| **Quyền hạn** | Cách ly tốt | Có thể gây lỗi permission trên host |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| tmpfs mount | Lưu trữ dữ liệu trong RAM của host, cực nhanh nhưng sẽ mất khi container stop. |
| Cloud Storage (S3) | Lưu trữ bên thứ ba, phù hợp cho file tĩnh quy mô lớn thay vì database. |

## 9. How
Cách sử dụng trong lệnh `docker run`:

```bash
# 1. Sử dụng Named Volume
docker run -d --name db -v my_data:/var/lib/mysql mysql

# 2. Sử dụng Bind Mount (đường dẫn tuyệt đối)
docker run -d --name app -v /home/user/project:/app node

# 3. Xem danh sách volumes
docker volume ls
```

Trong Docker Compose:
```yaml
services:
  web:
    volumes:
      - ./src:/app          # Bind mount
      - db_data:/data/db    # Named volume

volumes:
  db_data:
```

## 10. Production concerns
### Backup
Đối với Named Volume, bạn cần chạy một container phụ để nén thư mục volume và export ra ngoài.
`docker run --rm -v my_data:/source -v $(pwd):/backup alpine tar cvf /backup/data.tar /source`

### Driver
Docker hỗ trợ các Volume Drivers cho phép lưu trữ volume trực tiếp lên AWS EBS, Azure Disk hoặc NFS, giúp dữ liệu có thể di chuyển giữa các node trong cụm cluster.

## 11. Common mistakes
- Mistake: Dùng đường dẫn tương đối cho Bind Mount trong lệnh `docker run` (phải dùng đường dẫn tuyệt đối hoặc biến `$PWD`).
- Mistake: Xóa container kèm volume (`docker rm -v`) mà chưa backup dữ liệu quan trọng.

## 12. Sample project
Thiết lập một cụm WordPress:
1. Một volume đặt tên là `wp_files` cho mã nguồn WordPress.
2. Một volume đặt tên là `db_data` cho MySQL.
3. Thử xóa sạch container và chạy lại để kiểm tra dữ liệu vẫn còn nguyên.

## 13. Interview
### Core Q&A
1. Q: Điều gì xảy ra với dữ liệu trong Named Volume khi container bị xóa?
   A: Dữ liệu vẫn còn nguyên trên đĩa cứng của host. Nó chỉ bị xóa khi bạn gọi lệnh `docker volume rm` một cách tường minh.

2. Q: Tại sao Bind Mount thường gây lỗi permission?
   A: Vì user bên trong container (vd: root) và user bên ngoài host có thể có UID/GID khác nhau, dẫn đến việc container không có quyền ghi vào thư mục host.

### Scenario
"Bạn cần đồng bộ cấu hình Nginx từ máy host vào 10 container khác nhau. Bạn chọn giải pháp nào?"
-> Trả lời: Tôi chọn Bind Mount cho file `nginx.conf` với chế độ read-only (`:ro`). Cách này giúp tôi chỉ cần sửa file ở 1 nơi duy nhất trên host và tất cả container sẽ nhận thay đổi ngay lập tức mà không cần rebuild image.

## 14. References
- Official Guide: [Manage data in Docker](https://docs.docker.com/storage/)
- Docker Blog: [Volumes vs Bind Mounts](https://www.docker.com/blog/understanding-docker-volumes/)

## 15. Real-world Code
Kiểm tra thư mục `/var/lib/docker/volumes` trên một server Linux đang chạy để thấy cách Docker tổ chức dữ liệu thực tế.

## 16. Community
- Reddit: r/docker.
- Stack Overflow: Tag [docker-volume].
