---
created: 2026-05-06
tags:
  - "#type/library"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/docker"
related:
  - "[[Docker Image.md]]"
  - "[[github-actions-cd.md]]"
---

## 1. What
Docker Hub là một dịch vụ lưu trữ đám mây (Cloud-based registry) được cung cấp bởi Docker, cho phép người dùng tìm kiếm, lưu trữ và chia sẻ các Docker Images. Nó là registry mặc định mà Docker client sẽ tìm kiếm khi bạn thực hiện lệnh `docker pull`.

## 2. Why
Trước khi có Docker Hub, việc chia sẻ môi trường phần mềm giữa các lập trình viên hoặc triển khai lên server rất phức tạp (thường phải gửi file nén hoặc cấu hình server thủ công). Docker Hub ra đời để cung cấp một kho lưu trữ tập trung, giúp việc phân phối phần mềm trở nên dễ dàng như cách GitHub quản lý mã nguồn.

## 3. Mental Model
Hãy tưởng tượng Docker Hub giống như **"App Store"** hoặc **"Google Play"**.
- Các lập trình viên (Developers) là những người viết app và upload lên store.
- Các Docker Images là các "App" đã được đóng gói sẵn.
- Khi bạn cần dùng, bạn chỉ cần lên store tìm và tải về máy để chạy.

## 4. Where it fits
Vị trí trong hệ sinh thái Docker:
`Docker Client -> Docker Push -> Docker Hub (Registry) -> Docker Pull -> Docker Host (Container)`

## 5. When to use
- Khi bạn muốn chia sẻ image của mình cho cộng đồng hoặc cho team.
- Khi cần sử dụng các image chính thức (Official Images) của các phần mềm phổ biến như Nginx, MySQL, Node.js.
- Khi thiết lập quy trình CI/CD để tự động build và push image lên kho lưu trữ.

## 6. When NOT to use
- Đối với các dự án doanh nghiệp có dữ liệu cực kỳ nhạy cảm và không muốn lưu trữ trên cloud của bên thứ ba (Trường hợp này nên dùng Self-hosted Registry như Harbor hoặc AWS ECR).
- Khi hệ thống mạng nội bộ bị giới hạn internet, không thể pull image từ Docker Hub.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Miễn phí cho các Public repositories không giới hạn. | Giới hạn số lượng Pull (Rate limiting) cho người dùng miễn phí. |
| Tích hợp sẵn và cực kỳ dễ sử dụng với Docker CLI. | Chỉ có 1 Private repository miễn phí. |
| Chứa hàng triệu image sẵn có từ cộng đồng. | Rủi ro về bảo mật nếu sử dụng các image không chính thức (Untrusted). |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| AWS ECR / GCR | Tối ưu nếu bạn dùng hạ tầng cloud của AWS/Google, bảo mật và tốc độ cao hơn. |
| GitHub Packages | Tiện lợi nếu bạn muốn quản lý image ngay trong cùng một nơi với code. |
| Harbor | Giải pháp Open source để tự cài đặt Registry riêng (On-premise). |

## 9. How
Các lệnh cơ bản để tương tác với Docker Hub:

```bash
# 1. Đăng nhập vào Docker Hub
docker login -u <username>

# 2. Gắn tag cho image local để chuẩn bị push
docker tag my-image:latest <username>/my-image:v1.0

# 3. Đẩy image lên Docker Hub
docker push <username>/my-image:v1.0

# 4. Tải image từ Docker Hub về máy khác
docker pull <username>/my-image:v1.0
```

## 10. Production concerns
### Scaling
Sử dụng Docker Hub làm nơi trung chuyển để triển khai lên hàng nghìn node trong cluster. Cần lưu ý về giới hạn Rate Limit của Docker Hub có thể làm hỏng quy trình Auto-scaling nếu không dùng tài khoản trả phí hoặc cache.

### Failure
Nếu Docker Hub down, bạn không thể pull image mới. Giải pháp là luôn giữ một bản sao của các image quan trọng trong Registry nội bộ hoặc sử dụng các mirror registry.

### Monitoring
Theo dõi dung lượng lưu trữ và lỗ hổng bảo mật thông qua tính năng "Vulnerability Scanning" tích hợp sẵn của Docker Hub.

## 11. Common mistakes
- Mistake: Push image chứa thông tin nhạy cảm (API Key, mật khẩu DB).
  Fix: Sử dụng `.dockerignore` để loại bỏ các file bí mật và dùng Environment Variables thay thế.

- Mistake: Sử dụng image từ nguồn không uy tín (không có dấu tick xanh Official hoặc Verified Publisher).
  Fix: Luôn ưu tiên sử dụng Official Images để đảm bảo tính an toàn và tối ưu.

## 12. Sample project
Tạo một tài khoản Docker Hub, build một image đơn giản từ Dockerfile (ví dụ một trang HTML tĩnh với Nginx), push lên Docker Hub và nhờ một người bạn pull về máy họ để chạy thử.

## 13. Interview
### Core Q&A
1. Q: Làm thế nào để phân biệt Official Image trên Docker Hub?
   A: Official Image là các image được duy trì bởi Docker và chính chủ sở hữu phần mềm đó, có nhãn "Official Image" và thường có tên ngắn gọn (ví dụ: `nginx` thay vì `user/nginx`).

2. Q: Docker Hub Rate Limit ảnh hưởng như thế nào đến CI/CD?
   A: Đối với tài khoản miễn phí, Docker Hub giới hạn số lần pull trong 6 giờ. Nếu CI/CD chạy quá nhiều lần, lệnh pull sẽ bị từ chối, gây lỗi build/deploy.

### Scenario
"Công ty bạn yêu cầu chuyển toàn bộ image từ Docker Hub về một Registry nội bộ vì lý do bảo mật. Bạn sẽ thực hiện các bước nào?"
-> Trả lời: 
1. Setup Registry nội bộ (ví dụ Harbor).
2. Pull các image cần thiết từ Docker Hub về máy trung gian.
3. Retag các image đó theo địa chỉ Registry mới.
4. Push lên Registry mới.
5. Cập nhật lại các file cấu hình Deployment (K8s/Docker Compose) để trỏ về Registry mới.

## 14. References
- Official Site: [Docker Hub](https://hub.docker.com/)
- Documentation: [Docker Hub Docs](https://docs.docker.com/docker-hub/)

## 15. Real-world Code
Hầu hết các dự án Open Source lớn đều có repo chính thức trên Docker Hub:
- [Official Postgres Image](https://hub.docker.com/_/postgres)
- [Official Redis Image](https://hub.docker.com/_/redis)

## 16. Community
- Docker Forum
- Reddit: r/docker
- Stack Overflow: Tag [docker-hub]
