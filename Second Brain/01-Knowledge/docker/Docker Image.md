---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/docker"
related:
  - "[[docker-hub.md]]"
  - "[[Common Docker Commands.md]]"
  - "[[Docker in Ubuntu.md]]"
  - "[[Docker Buildx Cross-compile.md]]"
  - "[[Amazon Elastic Container Registry (ECR).md]]"
---

## 1. What
Docker Image là một file thực thi dạng "read-only", chứa tất cả các thành phần cần thiết để chạy một ứng dụng: từ mã nguồn, thư viện, biến môi trường cho đến các tệp tin cấu hình. Nó đóng vai trò là một "bản thiết kế" (blueprint) để khởi tạo các Docker Containers.

## 2. Why
Trước khi có Docker Image, việc đảm bảo ứng dụng chạy giống hệt nhau trên máy lập trình viên, máy test và server Production là một thách thức cực lớn ("It works on my machine" problem). Docker Image ra đời để đóng gói mọi thứ vào một gói duy nhất, giúp đảm bảo tính nhất quán tuyệt đối của môi trường chạy ứng dụng.

## 3. Mental Model
Hãy tưởng tượng Docker Image giống như một **"Đĩa cài đặt phần mềm"** hoặc một **"File ISO"**.
- Bạn không thể sửa đổi nội dung của đĩa khi đang chạy (Read-only).
- Từ một cái đĩa, bạn có thể cài đặt phần mềm lên bao nhiêu máy tùy thích (Container).
- Mọi máy được cài từ cái đĩa đó đều sẽ có phần mềm giống hệt nhau.

## 4. Where it fits
Vị trí trong luồng làm việc:
`Dockerfile (Source) -> Docker Build -> Docker Image (Blueprint) -> Docker Run -> Container (Instance)`

## 5. When to use
- Khi muốn đóng gói ứng dụng để triển khai lên nhiều môi trường khác nhau.
- Khi cần tạo ra các phiên bản phần mềm có thể khôi phục (Versioning).
- Khi xây dựng kiến trúc Microservices, nơi mỗi service cần một môi trường riêng biệt.

## 6. When NOT to use
- Khi dữ liệu thay đổi thường xuyên (Dữ liệu nên được lưu ở Volume, không nên lưu trong Image).
- Khi ứng dụng yêu cầu can thiệp sâu vào phần cứng đặc thù mà ảo hóa container không hỗ trợ tốt.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Nhẹ và khởi động nhanh hơn máy ảo (VM). | Dung lượng có thể phình to nếu không biết cách tối ưu các Layer. |
| Tính nhất quán cực cao giữa các môi trường. | Khó chỉnh sửa nội dung bên trong sau khi đã build (phải build lại). |
| Cấu trúc phân lớp (Layers) giúp tiết kiệm dung lượng khi lưu trữ nhiều image giống nhau. | Rủi ro bảo mật nếu các lớp base image có lỗ hổng. |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Virtual Machine Image (OVA/ISO) | Nặng hơn, chứa cả hệ điều hành đầy đủ, khởi động chậm hơn. |
| Snap / Flatpak | Các định dạng đóng gói ứng dụng cho Linux, nhưng không mạnh về quản lý môi trường server/cloud. |

## 9. How
Cấu trúc Layer của Docker Image:
Mỗi lệnh trong Dockerfile (FROM, RUN, COPY) tạo ra một lớp (Layer). Các lớp này được xếp chồng lên nhau.

```bash
# Xem danh sách các image trên máy
docker images

# Xây dựng image từ Dockerfile
docker build -t my-app:v1 .

# Xem lịch sử các layer của image
docker history my-app:v1

# Xóa một image
docker rmi my-app:v1
```

## 10. Production concerns
### Scaling
Sử dụng Multi-stage build để giảm kích thước image, giúp việc pull image từ registry về các node trong cluster nhanh hơn, giảm thời gian scale-out.

### Failure
Nếu Image bị lỗi (ví dụ: thiếu thư viện), container sẽ crash ngay khi khởi động. Cần sử dụng các công cụ như `Trivy` để quét lỗ hổng bảo mật của image trước khi đưa lên Production.

### Monitoring
Theo dõi kích thước image. Một image Production lý tưởng chỉ nên chứa runtime và application code, không nên chứa build tools hay source code rác.

## 11. Common mistakes
- Mistake: Lưu trữ dữ liệu log hoặc database bên trong image.
  Fix: Luôn sử dụng Docker Volumes để lưu trữ dữ liệu bền vững (Persistent data).

- Mistake: Sử dụng `latest` tag cho Production.
  Fix: Luôn tag image bằng phiên bản cụ thể (ví dụ: `v1.2.3` hoặc commit hash) để đảm bảo có thể rollback chính xác.

## 12. Sample project
Tạo một Dockerfile cho ứng dụng Node.js:
1. Sử dụng base image `node:alpine` để tối ưu dung lượng.
2. COPY `package.json` và chạy `npm install` trước.
3. COPY mã nguồn sau (để tận dụng layer cache).
4. Build và kiểm tra dung lượng image.

## 13. Interview
### Core Q&A
1. Q: "Layer Caching" trong Docker Image là gì?
   A: Khi build image, Docker sẽ lưu lại kết quả của từng bước. Nếu lần build sau các bước trước đó không thay đổi (ví dụ: file `package.json` không đổi), Docker sẽ dùng lại cache thay vì chạy lại lệnh, giúp tốc độ build cực nhanh.

2. Q: Tại sao nên dùng Alpine Linux làm base image?
   A: Vì Alpine cực kỳ nhẹ (chỉ khoảng 5MB), giúp giảm kích thước image cuối cùng, tăng tốc độ truyền tải qua mạng và giảm bề mặt tấn công bảo mật.

### Scenario
"Image của bạn nặng tới 2GB mặc dù ứng dụng chỉ có vài MB. Bạn sẽ làm gì để giảm dung lượng?"
-> Trả lời:
1. Sử dụng Multi-stage builds để tách biệt môi trường build và môi trường chạy.
2. Chọn Base Image nhỏ hơn (Alpine, Distroless).
3. Hợp nhất các lệnh `RUN` (ví dụ: `apt-get update && apt-get install && rm -rf /var/lib/apt/lists/*`) để tránh tạo ra layer thừa.
4. Sử dụng `.dockerignore` để loại bỏ `node_modules`, `.git`, và các file không cần thiết.

## 14. References
- Official Docs: [Docker Images](https://docs.docker.com/storage/storagedriver/)
- Best Practices: [Dockerfile Best Practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

## 15. Real-world Code
- [Explore Docker Hub Official Images](https://hub.docker.com/search?q=&type=image&image_filter=official)

## 16. Community
- Docker Community Slack
- Stack Overflow: Tag [docker-image]
