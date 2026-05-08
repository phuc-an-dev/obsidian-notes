---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/docker"
related:
  - "[[Docker Image.md]]"
  - "[[Multi-stage Build in Docker.md]]"
---

## 1. What
Dockerfile là một tệp văn bản (text file) chứa một tập hợp các câu lệnh tuần tự mà người dùng có thể gọi trên dòng lệnh để tạo ra một Docker Image. Nó đóng vai trò là "công thức" (recipe) để Docker Engine tự động hóa quy trình xây dựng môi trường cho ứng dụng.

## 2. Why
Trước khi có Dockerfile, việc tạo image phải làm thủ công (chạy container, cài phần mềm, rồi commit), rất khó để tái hiện và quản lý phiên bản. Dockerfile ra đời để:
- **Tái hiện (Reproducibility)**: Bất kỳ ai có Dockerfile đều có thể tạo ra một image giống hệt nhau.
- **Quản lý phiên bản**: Có thể lưu trữ Dockerfile trong Git cùng với mã nguồn ứng dụng.
- **Tự động hóa**: Cho phép tích hợp vào quy trình CI/CD để build image tự động.

## 3. Mental Model
Hãy tưởng tượng Dockerfile giống như một **"Bản thiết kế xây dựng"** hoặc một **"Công thức nấu ăn"**:
- **FROM**: Chọn nguyên liệu nền (Base OS).
- **RUN**: Các bước chế biến (Cài phần mềm).
- **COPY**: Thêm gia vị riêng của bạn (Mã nguồn ứng dụng).
- **CMD**: Hướng dẫn cách trình bày món ăn khi mang ra bàn (Lệnh khởi chạy ứng dụng).

## 4. Where it fits
Vị trí trong luồng công việc:
`Dockerfile (Source) -> docker build -> Docker Image (Blueprint) -> docker run -> Container (Instance)`

## 5. When to use
- Khi bắt đầu đóng gói bất kỳ ứng dụng nào để chạy dưới dạng container.
- Khi cần tùy chỉnh môi trường chạy (cài thêm thư viện hệ thống, cấu hình biến môi trường).
- Khi muốn tối ưu hóa quy trình build thông qua Multi-stage build.

## 6. When NOT to use
- Khi bạn chỉ cần sử dụng các image có sẵn từ Docker Hub mà không cần thay đổi gì.
- Để lưu trữ dữ liệu động (Dockerfile chỉ dùng để định nghĩa cấu trúc tĩnh của image).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Minh bạch và dễ hiểu (Infrastructure as Code). | Viết Dockerfile không tối ưu có thể làm phình to dung lượng image. |
| Hỗ trợ layer caching giúp build nhanh. | Khó debug các lỗi xảy ra trong quá trình build nếu script phức tạp. |
| Dễ dàng chia sẻ và bảo trì. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Cloud Native Buildpacks | Tự động tạo image từ code mà không cần viết Dockerfile, nhưng ít linh hoạt hơn. |
| `docker commit` | Tạo image từ container đang chạy, nhanh nhưng không thể quản lý phiên bản và khó tái hiện. |

## 9. How
Cấu trúc một Dockerfile chuẩn cho Node.js:

```dockerfile
# 1. Base image
FROM node:20-alpine

# 2. Thư mục làm việc
WORKDIR /app

# 3. Copy dependencies trước để tận dụng cache
COPY package*.json ./
RUN npm install

# 4. Copy mã nguồn sau
COPY . .

# 5. Cổng ứng dụng lắng nghe
EXPOSE 3000

# 6. Lệnh khởi chạy
CMD ["npm", "start"]
```

Lệnh build: `docker build -t my-app:v1 .`

## 10. Production concerns
### Security
Sử dụng các base image nhỏ gọn (như Alpine hoặc Distroless) để giảm bề mặt tấn công. Tránh chạy container bằng quyền `root`.

### Build Context
Sử dụng `.dockerignore` để loại bỏ các file rác (như `node_modules`, `.git`) khỏi build context, giúp tăng tốc độ truyền tải dữ liệu đến daemon.

### Layers
Mỗi lệnh trong Dockerfile tạo ra một layer. Hãy gộp các lệnh `RUN` liên quan lại với nhau để giảm số lượng layer không cần thiết.

## 11. Common mistakes
- Mistake: Quên dùng `.dockerignore`.
  Fix: Luôn tạo file `.dockerignore` để loại bỏ các file không cần thiết.

- Mistake: Để các thông tin nhạy cảm (API Key) trong Dockerfile.
  Fix: Sử dụng biến môi trường (`ENV`) hoặc Build Secrets thay thế.

## 12. Sample project
Viết Dockerfile cho một ứng dụng Spring Boot:
1. Dùng Maven để build file JAR.
2. Dùng OpenJDK image làm runtime.
3. Cấu hình ENTRYPOINT để nhận các tham số Java VM.

## 13. Interview
### Core Q&A
1. Q: Sự khác biệt giữa `CMD` và `ENTRYPOINT` là gì?
   A: `ENTRYPOINT` định nghĩa lệnh chính sẽ chạy khi khởi động container và không dễ bị ghi đè. `CMD` cung cấp các tham số mặc định cho ENTRYPOINT hoặc là một lệnh có thể bị ghi đè hoàn toàn khi dùng `docker run`.

2. Q: Tại sao thứ tự các lệnh trong Dockerfile lại quan trọng?
   A: Vì Docker sử dụng cơ chế Layer Caching. Nếu một layer thay đổi, tất cả các layer phía sau nó đều phải build lại. Do đó, nên đặt các phần ít thay đổi (cài đặt library) lên trước và phần hay thay đổi (mã nguồn) xuống sau.

### Scenario
"Image của bạn có dung lượng 1GB trong khi code chỉ có 10MB. Bạn tối ưu như thế nào qua Dockerfile?"
-> Trả lời:
1. Dùng base image nhỏ hơn (Alpine).
2. Sử dụng Multi-stage build để loại bỏ build tools.
3. Gộp các lệnh `RUN apt-get`.
4. Dùng `.dockerignore`.

## 14. References
- Official Reference: [Dockerfile reference](https://docs.docker.com/engine/reference/builder/)
- Best Practices: [Dockerfile best practices](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)

## 15. Real-world Code
Nghiên cứu Dockerfile của các dự án lớn như **Nginx**, **PostgreSQL** trên GitHub để học cách họ tối ưu hóa layer và bảo mật.

## 16. Community
- Reddit: r/docker.
- Stack Overflow: Tag [dockerfile].
