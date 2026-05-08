---
created: 2026-05-07
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/performance"
related:
  - "[[Docker Engine]]"
  - "[[Common Docker Commands]]"
  - "[[Container in Docker]]"
  - "[[Auth Docker with ECR.md]]"
  - "[[Docker Buildx Cross-compile]]"
  - "[[Testcontainers Cloud.md]]"
---

## 1. What
Docker Daemon (tên thực thi là `dockerd`) là một dịch vụ chạy ngầm (background process) đóng vai trò trung tâm trong hệ thống Docker. Nó lắng nghe các yêu cầu từ Docker API và thực hiện các nhiệm vụ nặng nhọc như xây dựng (build), chạy (run) và phân phối (distribute) các container.

## 2. Why
Trước khi có Docker Daemon, việc quản lý các tài nguyên Linux (như namespaces, cgroups) để tạo môi trường cô lập rất phức tạp và đòi hỏi nhiều câu lệnh thủ công. Docker Daemon ra đời để trừu tượng hóa các công nghệ này, cung cấp một giao diện API duy nhất để quản lý vòng đời của container một cách nhất quán và tự động.

## 3. Mental Model
Hãy tưởng tượng Docker Daemon như một **người quản lý bến cảng** (Harbor Master).
- Khách hàng (Docker Client/CLI) gửi yêu cầu: "Tôi muốn hạ thủy một con tàu mới".
- Người quản lý bến cảng (Daemon) nhận lệnh qua bộ đàm (API).
- Ông ấy trực tiếp điều phối xe cẩu, công nhân và kho bãi (tài nguyên hệ điều hành) để bốc dỡ hàng hóa (Images) và vận hành con tàu (Containers).
- Ông ấy làm việc thầm lặng trong văn phòng ngầm, bạn không thấy ông ấy làm việc trực tiếp, nhưng mọi hoạt động tại bến cảng đều do ông ấy kiểm soát.

## 4. Where it fits
Docker Client (CLI) -> **REST API over Unix Socket/TCP** -> **Docker Daemon (dockerd)** -> containerd -> runc -> Linux Kernel (LXC, Cgroups, Namespaces).

## 5. When to use
Docker Daemon luôn luôn được sử dụng mỗi khi bạn chạy bất kỳ câu lệnh Docker nào. Nó là thành phần bắt buộc phải chạy để Docker Engine có thể hoạt động.

## 6. When NOT to use
- Khi bạn chuyển sang các giải pháp container "daemonless" như **Podman**, nơi các container được chạy trực tiếp bởi người dùng mà không cần một service chạy ngầm với quyền root.
- Trong các môi trường cực kỳ hạn chế về tài nguyên, nơi việc duy trì một process chạy ngầm 24/7 là một gánh nặng.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Quản lý tập trung toàn bộ tài nguyên container. | Mặc định chạy với quyền `root`, tiềm ẩn rủi ro bảo mật (Single point of failure). |
| Cung cấp REST API mạnh mẽ để tích hợp với công cụ bên ngoài. | Nếu daemon bị treo hoặc crash, toàn bộ container (mặc định) sẽ bị ảnh hưởng. |
| Tự động quản lý việc khởi động lại container khi reboot. | Tiêu tốn một lượng tài nguyên RAM/CPU nhất định ngay cả khi không có container nào chạy. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Podman | Không cần daemon, hỗ trợ chế độ rootless tốt hơn, CLI tương thích 100% với Docker. |
| containerd | Một phần của Docker Daemon nhưng nhỏ gọn hơn, thường dùng trực tiếp trong Kubernetes (CRI). |
| CRI-O | Một runtime khác dành riêng cho Kubernetes, tối giản hơn Docker. |

## 9. How
Cấu hình Docker Daemon thông qua file `daemon.json` (thường nằm ở `/etc/docker/daemon.json` trên Linux):

```json
{
  "debug": true,
  "tls": true,
  "hosts": ["tcp://192.168.1.10:2376", "unix:///var/run/docker.sock"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "storage-driver": "overlay2"
}
```

Kiểm tra trạng thái daemon:
```bash
systemctl status docker
# Hoặc xem logs trực tiếp
journalctl -u docker -f
```

## 10. Production concerns
### Scaling
Mặc dù daemon không trực tiếp scale ảnh của bạn, nhưng nó giới hạn số lượng container có thể chạy trên một host dựa trên cấu hình bộ nhớ và CPU mà bạn thiết lập.

### Failure
Sử dụng tính năng **Live Restore** (`"live-restore": true`) trong `daemon.json` để cho phép các container tiếp tục chạy ngay cả khi Docker Daemon bị tắt hoặc đang được nâng cấp.

### Monitoring
Docker Daemon cung cấp một endpoint Prometheus (`/metrics`) để theo dõi sức khỏe của chính nó và các container mà nó quản lý.

## 11. Common mistakes
- Mistake: Để lộ Docker Daemon API (cổng 2375/2376) ra internet mà không có xác thực TLS.
  Fix: Luôn sử dụng Unix Socket mặc định hoặc cấu hình TLS chặt chẽ nếu cần truy cập từ xa.

- Mistake: Quên giới hạn kích thước logs của daemon, dẫn đến đầy ổ cứng.
  Fix: Cấu hình `log-driver` và `log-opts` trong `daemon.json`.

## 12. Sample project
Thiết lập một Docker Daemon có khả năng truy cập từ xa an toàn: Tạo các chứng chỉ SSL/TLS, cấu hình daemon lắng nghe qua TCP cổng 2376, và thiết lập biến môi trường `DOCKER_HOST` trên máy client để quản lý server từ xa.

## 13. Interview
### Core Q&A
1. Q: Làm thế nào để Docker Client giao tiếp với Docker Daemon?
   A: Thông qua REST API. Mặc định trên Linux là qua Unix Socket (`/var/run/docker.sock`), trên Windows/Mac là qua một máy ảo nhỏ.

2. Q: Nếu tôi `systemctl stop docker`, các container đang chạy sẽ ra sao?
   A: Mặc định chúng sẽ bị tắt theo. Tuy nhiên, nếu đã bật tính năng "Live Restore", chúng vẫn sẽ tiếp tục chạy ngầm.

3. Q: Tại sao việc thêm user vào group `docker` lại được coi là rủi ro bảo mật?
   A: Vì Docker Daemon chạy với quyền root. Việc có quyền truy cập vào daemon socket tương đương với việc có quyền root trên toàn bộ hệ thống (có thể mount `/` vào container và sửa đổi file hệ thống).

### Scenario
Server của bạn bị đầy ổ cứng và bạn nghi ngờ do Docker. Bạn sẽ kiểm tra gì ở tầng Daemon?
Trả lời: 1. Kiểm tra logs của daemon (`journalctl`). 2. Kiểm tra dung lượng của thư mục `/var/lib/docker` (nơi lưu trữ images/containers). 3. Kiểm tra xem có cấu hình log rotation trong `daemon.json` hay chưa.

## 14. References
- Official Docs: https://docs.docker.com/config/daemon/
- Dockerd Reference: https://docs.docker.com/engine/reference/commandline/dockerd/

## 15. Real-world Code
Nghiên cứu file cấu hình mặc định của Docker trên các bản phân phối Linux khác nhau tại `/etc/docker/daemon.json`.

## 16. Community
- GitHub: moby/moby (mã nguồn gốc của Docker Daemon).
- Stack Overflow: [docker-daemon] tag.
- Reddit: r/docker.
