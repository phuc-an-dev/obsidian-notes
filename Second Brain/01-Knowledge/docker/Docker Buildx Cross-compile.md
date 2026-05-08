---
created: 2026-05-07
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/docker"
related:
  - "[[Docker Image.md]]"
  - "[[Docker Daemon.md]]"
  - "[[Multi-stage Build in Docker.md]]"
---

## 1. What
Docker Buildx là một plugin CLI mở rộng lệnh `docker build` với các tính năng mạnh mẽ từ bộ công cụ Moby BuildKit. Tính năng nổi bật nhất của nó là hỗ trợ **Cross-compile** (biên dịch chéo), cho phép tạo ra các container image có thể chạy trên nhiều kiến trúc CPU khác nhau (như x86_64, ARM64, ARMv7) từ một máy build duy nhất.

## 2. Why
Trước khi có Buildx, để build một image chạy trên chip ARM (như Raspberry Pi hoặc AWS Graviton), bạn cần một máy chạy chip ARM tương ứng. Trong kỷ nguyên hiện đại:
- Developer dùng Mac M1/M2 (ARM) nhưng deploy lên server Linux (x86).
- Cần hỗ trợ đa dạng thiết bị đầu cuối với các kiến trúc chip khác nhau.
Buildx giải quyết vấn đề này bằng cách sử dụng giả lập QEMU hoặc kết nối tới các remote nodes để build image cho mọi nền tảng mà không cần thay đổi phần cứng máy build.

## 3. Mental Model
Hãy tưởng tượng Docker Buildx giống như một **"Thông dịch viên đa năng"**.
- Bạn chỉ biết nói tiếng Việt (máy tính của bạn chạy kiến trúc x86).
- Nhưng "Thông dịch viên" này có khả năng viết lại nội dung lời nói của bạn sang tiếng Anh, Pháp, Nhật (kiến trúc ARM, PowerPC, v.v.) cùng một lúc.
- Kết quả là bạn có các bản dịch khác nhau của cùng một thông điệp, giúp mọi người ở khắp nơi trên thế giới (mọi loại chip) đều có thể hiểu và thực thi được yêu cầu của bạn.

## 4. Where it fits
Source Code -> Dockerfile -> **Docker Buildx (BuildKit + QEMU)** -> Manifest List -> Container Registry.

## 5. When to use
- Khi muốn xây dựng "Multi-platform images" để hỗ trợ cả máy Mac chip Apple Silicon và server cloud truyền thống.
- Khi triển khai ứng dụng lên các thiết bị IoT (thường dùng ARM).
- Trong các pipeline CI/CD để tự động hóa việc đẩy các bản build đa kiến trúc lên Docker Hub hoặc ECR.

## 6. When NOT to use
- Khi dự án chỉ chạy trên một loại hạ tầng cố định duy nhất.
- Khi tác vụ build cực kỳ nặng về CPU. Việc giả lập kiến trúc khác qua QEMU sẽ chậm hơn từ 5-10 lần so với build native. Trong trường hợp này, nên dùng remote builder thật.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Build một lần, chạy mọi nơi (Multi-arch). | Tốc độ build qua giả lập QEMU rất chậm. |
| Quản lý manifest list tự động. | Cấu hình ban đầu trên Linux host phức tạp (cần binfmt). |
| Hỗ trợ các tính năng build hiện đại (Cache mount, Secrets). | Không thể load trực tiếp multi-arch image vào Docker local daemon (phải push lên registry). |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Native Build | Dùng máy thật kiến trúc đó để build. Nhanh nhất nhưng khó quản lý quy mô lớn. |
| `docker manifest` | Công cụ cũ để gộp các image đã build riêng biệt thành một list. Thủ công và tốn công hơn Buildx. |

## 9. How
Quy trình build đa kiến trúc:

### 1. Chuẩn bị (Chỉ cần 1 lần)
```bash
# Cài đặt QEMU giả lập (trên Linux)
docker run --privileged --rm tonistiigi/binfmt --install all

# Tạo một builder mới hỗ trợ multi-platform
docker buildx create --name mybuilder --use
docker buildx inspect --bootstrap
```

### 2. Build và Push
```bash
# Build cho cả x86 và ARM, sau đó đẩy lên Registry
docker buildx build --platform linux/amd64,linux/arm64 \
  -t username/my-app:v1 --push .
```

### 3. Kiểm tra kiến trúc của Image
```bash
docker buildx imagetools inspect username/my-app:v1
```

## 10. Production concerns
### Performance
Đối với các dự án lớn, hãy cấu hình `remote builder` bằng cách trỏ Buildx tới các instance chạy kiến trúc thật (ví dụ một con EC2 ARM64) để tránh độ trễ của QEMU.

### Caching
Sử dụng cờ `--cache-from` và `--cache-to` với type là `registry` hoặc `gha` (GitHub Actions) để tối ưu tốc độ build giữa các lần chạy.

## 11. Common mistakes
- Mistake: Quên cờ `--push`. Vì Docker daemon local không hỗ trợ lưu trữ multi-arch manifest, lệnh build sẽ báo lỗi nếu bạn cố lưu vào máy local mà không push.
  Fix: Luôn push trực tiếp lên registry hoặc chỉ định một platform duy nhất nếu muốn lưu local.

- Mistake: Không cài đặt handler `binfmt` trên host Linux, dẫn đến lỗi "exec format error" khi build layer đầu tiên của kiến trúc khác.
  Fix: Luôn chạy tool `tonistiigi/binfmt` trước khi dùng Buildx trên máy Linux mới.

## 12. Sample project
Tạo một ứng dụng Golang (Go hỗ trợ cross-compile cực tốt). Dùng Dockerfile multi-stage và Buildx để tạo một image siêu nhỏ (< 10MB) chạy được trên cả Windows Docker (amd64) và Raspberry Pi (arm32v7).

## 13. Interview
### Core Q&A
1. Q: "Manifest List" trong Docker là gì?
   A: Là một file chỉ mục (Index) trỏ tới các image cụ thể cho từng kiến trúc. Khi bạn pull một tag, Docker client sẽ đọc manifest này để tải về đúng các layer phù hợp với máy của bạn.

2. Q: Làm thế nào để tăng tốc Buildx khi build cho ARM trên máy x86?
   A: Sử dụng tính năng `remote nodes` trong Buildx để gán một máy ARM thật vào cụm builder, hoặc tối ưu Dockerfile bằng cách tận dụng `BUILDPLATFORM` và `TARGETPLATFORM` trong cross-compilation của ngôn ngữ (như Go).

### Scenario
"CI/CD của bạn chạy trên GitHub Actions (x86) và mất 30 phút để build image cho ARM64. Bạn làm gì?"
-> Trả lời: 
1. Sử dụng Docker Layer Caching (`type=gha`).
2. Nếu mã nguồn là Go/Rust, tôi sẽ dùng cross-compile của chính ngôn ngữ đó thay vì giả lập toàn bộ OS qua QEMU.
3. Thuê một runner ARM thật (như AWS Graviton) để build native.

## 14. References
- Docker Docs: [Working with Buildx](https://docs.docker.com/build/buildx/)
- BuildKit GitHub: [moby/buildkit](https://github.com/moby/buildkit)
- Multi-platform images: [Official Guide](https://docs.docker.com/build/building/multi-platform/)

## 15. Real-world Code
Sử dụng GitHub Action chính thức để setup Buildx:
```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v2
```

## 16. Community
- Docker Slack: #buildkit channel.
- Stack Overflow: Tag [docker-buildx].
- Blog: "Faster Multi-platform builds with BuildKit" trên Docker Blog.
