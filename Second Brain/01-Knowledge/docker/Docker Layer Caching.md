---
created: 2026-05-06
tags:
  - "#type/concept"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/docker"
related:
  - "[[Dockerfile.md]]"
  - "[[Docker Engine.md]]"
---

## 1. What
Docker Layer Caching là một cơ chế tối ưu hóa của Docker Engine giúp tái sử dụng các kết quả của những lần build trước đó. Mỗi dòng lệnh trong Dockerfile (như `RUN`, `COPY`, `ADD`) tạo ra một lớp (layer) dữ liệu. Nếu dòng lệnh và các file liên quan không thay đổi, Docker sẽ lấy kết quả từ cache thay vì thực hiện lại lệnh đó.

## 2. Why
Quá trình build image có thể tốn rất nhiều thời gian (tải thư viện, biên dịch code). Layer Caching giúp:
- **Tăng tốc độ build**: Giảm thời gian từ vài phút xuống còn vài giây cho các thay đổi nhỏ.
- **Tiết kiệm băng thông**: Không cần tải lại các package từ internet nếu layer chứa chúng đã có trong cache.
- **Tối ưu quy trình CI/CD**: Giúp lập trình viên nhận được phản hồi nhanh hơn sau mỗi lần push code.

## 3. Mental Model
Hãy tưởng tượng Layer Caching giống như việc **"Chụp ảnh từng công đoạn nấu ăn"**:
- Bước 1: Luộc trứng. Bạn chụp một tấm ảnh trứng đã luộc.
- Bước 2: Bóc vỏ. Bạn chụp ảnh trứng đã bóc vỏ.
- Nếu lần sau bạn muốn làm món trứng kho, và bạn đã có sẵn ảnh "trứng đã bóc vỏ" (cache), bạn không cần đi luộc trứng và bóc lại từ đầu nữa. Bạn chỉ cần lấy "ảnh" đó ra và làm tiếp bước kho thịt.
- Nhưng nếu ở bước 1 bạn đổi sang luộc trứng vịt thay vì trứng gà, tấm ảnh bước 2 sẽ trở nên vô dụng và bạn phải chụp lại từ đầu.

## 4. Where it fits
Vị trí trong quy trình:
`docker build -> Check Cache for Instruction -> Cache Hit (Reuse) / Cache Miss (Execute) -> Create New Layer`

## 5. When to use
- Luôn luôn hiện diện mặc định trong Docker.
- Cần đặc biệt chú ý khi viết Dockerfile cho các ứng dụng có nhiều dependencies (Node.js, Java, Python).

## 6. When NOT to use
- Khi bạn muốn đảm bảo image được build hoàn toàn mới với các bản cập nhật mới nhất từ internet (ví dụ: `apt-get update`).
- Trong trường hợp này, dùng flag `--no-cache` khi build.

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Tốc độ cực nhanh cho các lần build lặp lại. | Chiếm dụng dung lượng đĩa cứng để lưu trữ các layer cũ. |
| Giảm tải cho các server package/repository. | Có thể gây ra lỗi nếu cache chứa các dữ liệu cũ không còn phù hợp (vd: bản vá bảo mật). |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| `--no-cache` | Bỏ qua hoàn toàn cache, chậm nhưng đảm bảo image mới nhất. |
| BuildKit Cache Mounts | Cơ chế cache nâng cao cho các thư mục cụ thể (như `.m2` hoặc `node_modules`), hiệu quả hơn layer cache truyền thống. |

## 9. How
Quy tắc vàng để tận dụng cache: **"Phần ít thay đổi đặt lên trước, phần hay thay đổi đặt xuống sau"**.

**Cách viết SAI (làm mất cache của npm install mỗi khi sửa code):**
```dockerfile
FROM node:alpine
WORKDIR /app
COPY . .  # Sửa 1 dòng code là lệnh tiếp theo chạy lại hết
RUN npm install
CMD ["node", "app.js"]
```

**Cách viết ĐÚNG (tận dụng cache cho dependencies):**
```dockerfile
FROM node:alpine
WORKDIR /app
COPY package*.json ./ 
RUN npm install       # Chỉ chạy lại khi package.json thay đổi
COPY . .              # Sửa code chỉ làm chạy lại lệnh này và CMD
CMD ["node", "app.js"]
```

## 10. Production concerns
### Invalidating Cache
Nếu bạn muốn ép Docker chạy lại một lệnh `RUN apt-get update` mà không muốn dùng `--no-cache` cho toàn bộ file, bạn có thể thêm một biến môi trường thay đổi được (như `ARG BUILD_DATE`).

### Security Patches
Hãy cẩn thận với các lệnh cài đặt security patches. Nếu layer đó đã bị cache từ 1 tháng trước, image của bạn sẽ không nhận được các bản vá mới nhất trừ khi bạn invalid cache thủ công.

## 11. Common mistakes
- Mistake: Gộp tất cả các lệnh `COPY` và `RUN` vào cuối file.
- Mistake: Thay đổi một comment hoặc một dòng lệnh ở đầu Dockerfile (làm mất cache của toàn bộ các dòng phía sau).

## 12. Sample project
Tạo một Dockerfile cho Python:
1. `COPY requirements.txt .`
2. `RUN pip install -r requirements.txt`
3. `COPY . .`
Thử thay đổi một file `.py` và quan sát xem lệnh `pip install` có chạy lại không (nếu đúng nó sẽ hiện chữ `CACHED`).

## 13. Interview
### Core Q&A
1. Q: Điều gì làm cho một layer bị "Cache Miss" (vô hiệu hóa cache)?
   A: (1) Nội dung lệnh trong Dockerfile thay đổi. (2) Các file được `COPY` hoặc `ADD` vào có nội dung thay đổi. (3) Một layer phía trước nó bị Cache Miss.

2. Q: Tại sao lệnh `RUN apt-get update && apt-get install -y package` nên nằm trên cùng một dòng?
   A: Để đảm bảo tính nhất quán. Nếu tách làm 2 dòng, lệnh `update` có thể bị cache và khi bạn thêm một package mới ở dòng dưới, nó sẽ cài package đó dựa trên danh sách cũ kỹ từ cache, dẫn đến lỗi hoặc thiếu bản vá.

### Scenario
"Build CI của bạn mất 15 phút, trong đó 10 phút là tải thư viện Maven. Làm thế nào để dùng Layer Caching tối ưu trường hợp này?"
-> Trả lời: Tôi sẽ `COPY` file `pom.xml` vào trước, sau đó chạy `mvn dependency:go-offline` (để tải toàn bộ lib). Sau đó mới `COPY` mã nguồn và chạy `mvn package`. Như vậy, trừ khi `pom.xml` đổi, bước tải thư viện sẽ luôn được CACHED.

## 14. References
- Docker Guide: [Optimizing builds with cache](https://docs.docker.com/build/cache/)
- BuildKit: [Cache mounts documentation](https://docs.docker.com/build/guide/cache/)

## 15. Real-world Code
Hầu hết các CI/CD chuyên nghiệp (GitHub Actions, GitLab CI) đều hỗ trợ lưu trữ (persist) các layer cache này giữa các lần chạy khác nhau để tối ưu tốc độ.

## 16. Community
- Docker Forum.
- Stack Overflow: Tag [docker] [caching].
