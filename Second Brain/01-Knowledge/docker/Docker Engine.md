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
  - "[[Common Docker Commands.md]]"
  - "[[Docker Daemon.md]]"
---

## 1. What
Docker Engine là thành phần cốt lõi của nền tảng Docker, đóng vai trò là một ứng dụng client-server giúp khởi tạo và quản lý các containers. Nó bao gồm ba thành phần chính: một daemon chạy ngầm (dockerd), một REST API để tương tác với daemon, và một giao diện dòng lệnh (Docker CLI) để người dùng nhập lệnh.

## 2. Why
Trước khi có Docker Engine, việc ảo hóa chủ yếu dựa trên Virtual Machines (VM) vốn rất nặng nề vì mỗi VM cần một hệ điều hành đầy đủ. Docker Engine ra đời để tận dụng khả năng cô lập của Linux Kernel (namespaces và cgroups), giúp chạy hàng chục ứng dụng trên cùng một máy chủ mà vẫn đảm bảo tính độc lập, nhẹ nhàng và khởi động cực nhanh.

## 3. Mental Model
Hãy tưởng tượng Docker Engine giống như một **"Nhà máy đóng đóng gói tự động"**:
- **Dockerd (Daemon)**: Là hệ thống máy móc điều hành bên trong nhà máy, làm mọi việc nặng nhọc như lắp ráp, vận hành.
- **REST API**: Là bảng mạch điều khiển của hệ thống máy móc đó.
- **Docker CLI**: Là chiếc remote cầm tay bạn dùng để gửi lệnh cho nhà máy (vd: "Đóng gói kiện hàng mới", "Dừng dây chuyền số 1").

## 4. Where it fits
Vị trí trong kiến trúc hạ tầng:
`Infrastructure (Server) -> Host OS (Linux/Windows) -> Docker Engine -> Containers (App 1, App 2, ...)`

## 5. When to use
- Khi cần đóng gói ứng dụng để triển khai nhất quán trên mọi môi trường.
- Khi muốn tối ưu hóa tài nguyên phần cứng bằng cách chạy nhiều dịch vụ trên một server duy nhất.
- Khi xây dựng kiến trúc Microservices yêu cầu sự cô lập giữa các thành phần.
- Trong các quy trình CI/CD để tự động hóa việc build và test.

## 6. When NOT to use
- Khi ứng dụng yêu cầu can thiệp cực sâu vào phần cứng hoặc kernel đặc thù mà ảo hóa container không hỗ trợ.
- Khi chạy các ứng dụng Windows Legacy cũ mà không thể chuyển đổi sang kiến trúc hiện đại.
- Nếu bạn chỉ cần chạy một website tĩnh cực kỳ đơn giản (nên dùng các giải pháp Hosting/SaaS thay vì tự quản trị Docker Engine).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Cực kỳ nhẹ và sử dụng ít tài nguyên hơn VM. | Tính bảo mật thấp hơn VM vì dùng chung Kernel với Host OS. |
| Khởi động container gần như ngay lập tức. | Quản lý mạng (Networking) phức tạp hơn khi hệ thống mở rộng. |
| Hệ sinh thái hỗ trợ cực lớn (Docker Hub). | Dữ liệu trong container sẽ mất nếu không cấu hình Volume đúng cách. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Podman | Không có daemon chạy ngầm, bảo mật hơn vì không yêu cầu quyền root mặc định. |
| Containerd | Thành phần cấp thấp hơn, thường được Kubernetes sử dụng trực tiếp thay vì Docker Engine đầy đủ. |
| VirtualBox / VMware | Ảo hóa hoàn toàn, bảo mật hơn nhưng nặng nề và chậm hơn nhiều. |

## 9. How
Kiểm tra thông tin Docker Engine:

```bash
# 1. Kiểm tra phiên bản của Client và Server (Daemon)
docker version

# 2. Xem thông tin chi tiết về tài nguyên, số lượng container, image
docker info

# 3. Kiểm tra trạng thái dịch vụ của Engine trên Linux
sudo systemctl status docker

# 4. Chạy một container thử nghiệm để xác nhận Engine hoạt động tốt
docker run hello-world
```

## 10. Production concerns
### Docker Socket Security
Mặc định, Docker Engine lắng nghe trên `/var/run/docker.sock`. Bất kỳ ai có quyền truy cập file này đều có quyền tương đương `root` trên host. Tuyệt đối không expose file này vào các container không tin cậy.

### Storage Driver
Sử dụng `overlay2` (mặc định trên Ubuntu) để có hiệu năng tốt nhất. Tránh dùng các driver cũ như `devicemapper` trên Production.

### Logging
Cấu hình giới hạn dung lượng log cho Engine trong file `/etc/docker/daemon.json` để tránh làm đầy ổ cứng server.

## 11. Common mistakes
- Mistake: Chạy Docker Engine với quyền user thông thường mà chưa cấu hình group.
  Fix: `sudo usermod -aG docker $USER` (sau đó logout và login lại).

- Mistake: Để mặc định Engine mở port 2375 (API) ra internet mà không có mã hóa TLS.
  Fix: Luôn dùng TLS nếu cần điều khiển Docker Engine từ xa.

## 12. Sample project
Thiết lập một "Docker Monitor":
1. Cài đặt Docker Engine trên Ubuntu.
2. Chạy một container Nginx.
3. Chạy thêm một container Netdata hoặc Portainer để giám sát Engine thông qua docker.sock (với cấu hình read-only).

## 13. Interview
### Core Q&A
1. Q: Docker Engine gồm những thành phần chính nào?
   A: Gồm Docker Daemon (dockerd), REST API và Docker CLI.

2. Q: Sự khác biệt giữa Docker Engine và Docker Desktop là gì?
   A: Docker Engine là phần lõi chạy trên Linux. Docker Desktop là gói phần mềm bao gồm Docker Engine kèm theo một máy ảo Linux siêu nhẹ và giao diện GUI để chạy được trên Windows và macOS.

### Scenario
"Daemon Docker bị treo (dockerd not responding), bạn sẽ làm gì?"
-> Trả lời:
1. Kiểm tra logs hệ thống bằng `journalctl -u docker`.
2. Kiểm tra xem đĩa cứng có bị đầy không (nguyên nhân phổ biến khiến daemon crash).
3. Thử restart dịch vụ bằng `sudo systemctl restart docker`.
4. Nếu vẫn lỗi, kiểm tra file `/etc/docker/daemon.json` xem có cấu hình nào sai cú pháp vừa mới thêm vào không.

## 14. References
- Official Architecture: [Docker Engine Overview](https://docs.docker.com/engine/)
- Deep Dive: [Docker Engine Components](https://docs.docker.com/get-started/overview/#docker-architecture)

## 15. Real-world Code
Nghiên cứu file cấu hình daemon chuẩn cho Production:
```json
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2"
}
```

## 16. Community
- Reddit: r/docker.
- Docker Community Slack.
- Stack Overflow: Tag [docker-engine].
