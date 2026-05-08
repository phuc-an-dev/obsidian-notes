---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/architecture"
related:
  - "[[Container in Docker.md]]"
  - "[[Docker Compose.md]]"
---

## 1. What
Container Orchestration (Điều phối Container) là quá trình tự động hóa việc triển khai, quản lý, mở rộng và kết nối các container trong một môi trường phân tán. Nếu container là một đơn vị thực thi đơn lẻ, thì Orchestration là hệ thống điều khiển toàn bộ "đội quân" container đó.

## 2. Why
Khi chạy một ứng dụng thực tế với hàng trăm container trên hàng chục máy chủ khác nhau, việc quản lý thủ công là bất khả thi. Container Orchestration giải quyết các bài toán:
- **Scheduling**: Tự động chọn máy chủ còn trống để đặt container vào.
- **Scaling**: Tự động tăng số lượng container khi lượng truy cập tăng đột biến.
- **Self-healing**: Tự động khởi động lại hoặc thay thế các container bị lỗi.
- **Load Balancing**: Tự động chia tải giữa các bản sao của ứng dụng.
- **Service Discovery**: Giúp các container tìm thấy nhau trong một mạng lưới phức tạp.

## 3. Mental Model
Hãy tưởng tượng Container Orchestration giống như một **"Nhạc trưởng của dàn nhạc giao hưởng"**:
- Mỗi nghệ sĩ là một **Container**, chơi một nhạc cụ riêng (Service).
- Nhạc trưởng (**Orchestrator**) không trực tiếp chơi nhạc.
- Nhạc trưởng ra hiệu khi nào nghệ sĩ bắt đầu (Start), điều chỉnh âm lượng (Scaling), và nếu một nghệ sĩ bị ốm, nhạc trưởng sẽ gọi người thay thế ngay lập tức (Self-healing) để bản nhạc không bị gián đoạn.

## 4. Where it fits
Vị trí trong hạ tầng:
`Developer -> Orchestrator API (K8s/Swarm) -> Cluster of Servers -> Docker Engine -> Containers`

## 5. When to use
- Khi hệ thống chuyển sang kiến trúc Microservices với nhiều thành phần phụ thuộc.
- Khi cần đảm bảo tính sẵn sàng cao (High Availability) - ứng dụng vẫn chạy kể cả khi một vài máy chủ bị sập.
- Khi triển khai ứng dụng trên quy mô lớn (Enterprise scale).
- Khi cần thực hiện các chiến lược triển khai hiện đại như Blue-Green hoặc Canary Deployment.

## 6. When NOT to use
- Đối với các ứng dụng nhỏ, đơn lẻ chạy trên một máy chủ duy nhất (dùng Docker Compose là đủ).
- Khi team chưa có kinh nghiệm và dự án không yêu cầu tính mở rộng phức tạp (vì Orchestration tăng độ phức tạp của hệ thống rất nhiều).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Tự động hóa vận hành cực kỳ mạnh mẽ. | Độ dốc học tập (learning curve) rất cao. |
| Khả năng mở rộng (Scalability) vô hạn. | Cần nhiều tài nguyên để chạy chính hệ thống điều phối (Control Plane). |
| Đảm bảo uptime tối đa cho dịch vụ. | Khó debug các lỗi liên quan đến networking và cấu hình cluster. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Kubernetes (K8s) | Phổ biến nhất, mạnh mẽ nhất, tiêu chuẩn ngành. |
| Docker Swarm | Tích hợp sẵn trong Docker, dễ dùng hơn K8s nhưng ít tính năng hơn. |
| AWS ECS / Fargate | Dịch vụ Managed của AWS, giảm bớt gánh nặng quản trị cluster. |
| Nomad | Đơn giản, linh hoạt, hỗ trợ cả container và các tiến trình truyền thống. |

## 9. How
Các khái niệm cốt lõi trong một hệ thống Orchestration (ví dụ Kubernetes):
- **Cluster**: Tập hợp các máy chủ (Nodes).
- **Control Plane**: "Bộ não" điều khiển cluster.
- **Pod/Task**: Đơn vị nhỏ nhất chứa một hoặc nhiều container.
- **Deployment**: Định nghĩa trạng thái mong muốn (vd: "Tôi luôn muốn có 3 bản sao của App A").
- **Service**: Điểm truy cập cố định (IP/DNS) cho một nhóm container.

## 10. Production concerns
### Resource Quotas
Luôn giới hạn tài nguyên cho từng đội nhóm hoặc dự án để tránh tình trạng một ứng dụng chiếm dụng toàn bộ tài nguyên của Cluster.

### State Management
Việc quản lý dữ liệu bền vững (Stateful apps) trên Orchestration khó hơn nhiều so với container đơn lẻ vì container có thể bị di chuyển giữa các máy chủ khác nhau. Cần dùng các giải pháp như Persistent Volumes (PV).

### Security
Bảo mật mạng (Network Policies) và quản lý quyền truy cập (RBAC) là yếu tố sống còn để ngăn chặn sự lây lan nếu một container bị tấn công.

## 11. Common mistakes
- Mistake: Cố gắng tự cài đặt và quản trị cụm Kubernetes từ đầu cho dự án nhỏ (nên dùng Managed K8s như EKS, GKE).
- Mistake: Không cấu hình giới hạn tài nguyên (Limits/Requests), dẫn đến tranh chấp tài nguyên giữa các service.

## 12. Sample project
Xây dựng một "Mini Cluster" bằng **Minikube** hoặc **Kind**:
1. Định nghĩa một Deployment với 3 replicas của ứng dụng Nginx.
2. Thử xóa thủ công một Pod và quan sát cách hệ thống tự động tạo lại Pod mới.
3. Thực hiện scale-up lên 5 replicas bằng một câu lệnh duy nhất.

## 13. Interview
### Core Q&A
1. Q: "Desired State" trong Orchestration là gì?
   A: Là trạng thái mà người dùng mong muốn hệ thống đạt được (vd: chạy 5 container). Orchestrator sẽ liên tục giám sát trạng thái thực tế và thực hiện các hành động cần thiết để đưa thực tế về đúng với mong muốn.

2. Q: Sự khác biệt giữa Docker Compose và Kubernetes?
   A: Docker Compose dùng để quản lý đa container trên **một máy chủ duy nhất**. Kubernetes dùng để quản lý hàng ngàn container trên **một cụm nhiều máy chủ**.

### Scenario
"Ứng dụng của bạn cần update phiên bản mới mà không được làm gián đoạn người dùng. Bạn làm thế nào?"
-> Trả lời: Tôi sử dụng chiến lược **Rolling Update**. Orchestrator sẽ thay thế từng container cũ bằng container mới một cách tuần tự. Nếu container mới bị lỗi, hệ thống sẽ tự động dừng lại và rollback về bản cũ, đảm bảo luôn có container hoạt động để phục vụ khách hàng.

## 14. References
- CNCF: [Cloud Native Landscape](https://landscape.cncf.io/)
- Kubernetes Documentation: [What is Kubernetes?](https://kubernetes.io/docs/concepts/overview/)

## 15. Real-world Code
Nghiên cứu các file **Helm Charts** hoặc **Terraform scripts** dùng để triển khai hạ tầng lên AWS EKS hoặc Google GKE.

## 16. Community
- Reddit: r/kubernetes.
- KubeCon + CloudNativeCon.
- Stack Overflow: Tag [kubernetes], [docker-swarm].
