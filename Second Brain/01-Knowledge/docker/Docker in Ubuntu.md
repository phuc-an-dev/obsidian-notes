---
created: 2026-05-06
tags:
  - "#type/tutorial"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/docker"
related:
  - "[[Docker Image.md]]"
  - "[[Common Docker Commands.md]]"
---

## 1. What
Docker in Ubuntu là quá trình cài đặt, cấu hình và tối ưu hóa Docker Engine trên hệ điều hành Ubuntu Linux. Đây là môi trường phổ biến nhất để chạy Docker trong thực tế nhờ tính ổn định và sự hỗ trợ mạnh mẽ từ cộng đồng.

## 2. Why
Trước khi có Docker, việc cài đặt các dependencies cho ứng dụng trên Ubuntu thường gây ra xung đột phiên bản (Dependency hell). Cài đặt Docker trên Ubuntu giúp tạo ra một lớp trừu tượng, cho phép chạy nhiều ứng dụng với các yêu cầu môi trường khác nhau trên cùng một host mà không làm bẩn hệ thống gốc.

## 3. Mental Model
Hãy tưởng tượng Ubuntu là một **"Khu đất trống"** và Docker là một **"Hệ thống nhà tiền chế"**. Thay vì bạn phải xây gạch, trát vữa trực tiếp lên đất (cài phần mềm trực tiếp), bạn chỉ cần lắp đặt khung thép Docker. Sau đó, bạn có thể đặt bất kỳ căn nhà (Container) nào lên đó một cách nhanh chóng và có thể nhấc đi nơi khác dễ dàng.

## 4. Where it fits
Vị trí trong phân lớp hệ thống:
`Hardware -> Ubuntu OS -> Docker Engine -> Docker Containers`

## 5. When to use
- Khi bạn muốn thiết lập một server Linux để chạy các ứng dụng container hóa.
- Trong các quy trình CI/CD sử dụng Self-hosted Runner trên Ubuntu.
- Khi phát triển ứng dụng cần môi trường tương đồng nhất với Production.

## 6. When NOT to use
- Khi bạn sử dụng các dịch vụ Managed Container như AWS Fargate hoặc Google Cloud Run (nơi nhà cung cấp quản lý sẵn runtime).
- Nếu máy tính của bạn dùng Windows hoặc macOS (nên dùng Docker Desktop thay vì cài trực tiếp trong WSL/VM trừ khi có nhu cầu đặc thù).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Hiệu năng native (không qua lớp ảo hóa như trên Windows/macOS). | Cần quản lý việc cập nhật phiên bản OS và Docker thủ công. |
| Hỗ trợ đầy đủ nhất các tính năng của Linux Kernel (cgroups, namespaces). | Yêu cầu kiến thức về quản lý User/Group để đảm bảo bảo mật. |
| Dễ dàng debug thông qua logs hệ thống (journalctl). | Chiếm dụng tài nguyên disk cho các lớp image nếu không dọn dẹp thường xuyên. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Podman | Không có daemon chạy ngầm, bảo mật hơn vì không cần quyền root mặc định. |
| containerd | Runtime nhẹ hơn, thường dùng làm back-end cho Kubernetes thay vì Docker đầy đủ. |

## 9. How
Quy trình cài đặt chuẩn từ Docker Repository:

```bash
# 1. Cập nhật và cài đặt các gói cần thiết
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg

# 2. Thêm Docker GPG key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 3. Thiết lập repository
echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 4. Cài đặt Docker Engine
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 5. Chạy test
sudo docker run hello-world
```

## 10. Production concerns
### Security
Mặc định lệnh `docker` yêu cầu quyền `sudo`. Để bảo mật, không nên add user vào group `docker` trên Production nếu không thực sự cần thiết vì user đó sẽ có quyền tương đương root.

### Storage Driver
Trên Ubuntu hiện đại, Docker mặc định dùng `overlay2`. Đây là driver tối ưu nhất về hiệu năng và dung lượng.

### Logging
Cấu hình `log-driver` trong `/etc/docker/daemon.json` để giới hạn kích thước file log, tránh làm đầy ổ cứng:
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

## 11. Common mistakes
- Mistake: Cài đặt Docker từ repository mặc định của Ubuntu (`apt install docker.io`).
  Fix: Luôn cài từ Docker Official Repository để có phiên bản mới nhất và ổn định nhất.

- Mistake: Quên khởi động Docker cùng hệ thống.
  Fix: Chạy `sudo systemctl enable docker.service`.

## 12. Sample project
Thiết lập một con VPS Ubuntu trắng, cài đặt Docker, sau đó sử dụng Docker Compose để chạy một cụm WordPress + MySQL với cấu hình tự động khởi động lại (Restart policy).

## 13. Interview
### Core Q&A
1. Q: Tại sao cần GPG key khi cài đặt Docker trên Ubuntu?
   A: GPG key dùng để xác thực các gói phần mềm tải về từ Docker repository là chính chủ, chưa bị can thiệp hoặc thay đổi bởi bên thứ ba.

2. Q: Làm thế nào để chạy lệnh `docker` mà không cần gõ `sudo`?
   A: Thêm user vào group docker: `sudo usermod -aG docker $USER`. Lưu ý sau đó cần logout và login lại.

### Scenario
"Server Ubuntu của bạn bị treo do Docker chiếm dụng quá nhiều disk. Bạn xử lý thế nào?"
-> Trả lời:
1. Chạy `docker system df` để kiểm tra phân bổ dung lượng.
2. Dùng `docker system prune` để xóa các container, network và image không dùng đến.
3. Kiểm tra log của các container trong `/var/lib/docker/containers/` xem có file nào quá lớn không và cấu hình lại log rotation.

## 14. References
- Official Guide: [Install Docker Engine on Ubuntu](https://docs.docker.com/engine/install/ubuntu/)
- Post-install steps: [Linux post-installation steps](https://docs.docker.com/engine/install/linux-postinstall/)

## 15. Real-world Code
Sử dụng Ansible để tự động hóa việc cài đặt Docker trên nhiều node Ubuntu cùng lúc:
- [Geerlingguy Ansible Role Docker](https://github.com/geerlingguy/ansible-role-docker)

## 16. Community
- Ubuntu Forums
- Docker Community Slack (#linux channel)
- Stack Overflow: Tag [docker] [ubuntu]
