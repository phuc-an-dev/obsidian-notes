---
created: 2026-05-06
tags:
  - "#type/pattern"
  - "#status/draft"
  - "#lang/devops"
  - "#topic/docker"
related:
  - "[[Dockerfile.md]]"
  - "[[Docker Image.md]]"
  - "[[Docker Buildx Cross-compile.md]]"
---

## 1. What
Multi-stage Build là một kỹ thuật trong Dockerfile cho phép sử dụng nhiều câu lệnh `FROM` để chia quá trình xây dựng image thành nhiều giai đoạn (stages). Kỹ thuật này giúp bạn chuyển các artifact (như file JAR, file biên dịch) từ stage này sang stage khác và cuối cùng chỉ giữ lại những gì thực sự cần thiết để chạy ứng dụng trong image cuối cùng.

## 2. Why
Khi build một ứng dụng (ví dụ Java hoặc Node.js), bạn cần rất nhiều công cụ: JDK, Maven, Compiler, npm, v.v. Nếu build theo cách thông thường, image cuối cùng sẽ chứa tất cả các công cụ này, dẫn đến:
- **Dung lượng cực lớn**: Image phình to hàng GB.
- **Bảo mật kém**: Chứa nhiều build tools mà hacker có thể lợi dụng nếu xâm nhập được.
Multi-stage build giải quyết triệt để hai vấn đề này, tạo ra các image siêu nhẹ (chỉ chứa runtime) và an toàn hơn.

## 3. Mental Model
Hãy tưởng tượng quy trình **"Làm bánh mang đi"**:
- **Stage 1 (Nhà bếp)**: Bạn dùng lò nướng, máy đánh trứng, bát đĩa cồng kềnh để làm ra chiếc bánh.
- **Stage 2 (Hộp quà)**: Bạn chỉ nhấc chiếc bánh đã hoàn thành bỏ vào một chiếc hộp nhỏ xinh để giao cho khách.
- Khách hàng (Production environment) chỉ nhận chiếc hộp chứa bánh, họ không cần (và không nên có) cái lò nướng hay máy đánh trứng của bạn.

## 4. Where it fits
Vị trí trong quy trình Build:
`Source Code -> Stage 1 (Build Environment) -> Artifact (.jar/.js) -> Stage 2 (Runtime Environment) -> Final Image`

## 5. When to use
- Mọi ứng dụng cần biên dịch (Compiled languages) như Java, Go, Rust, C++.
- Các ứng dụng Frontend (React, Vue) cần build sang file tĩnh (HTML/JS) để serve bằng Nginx.
- Khi muốn giảm kích thước image tối đa cho môi trường Production.

## 6. When NOT to use
- Các script cực kỳ đơn giản (Python, Bash) không cần bước build phức tạp.
- Trong môi trường Development cần giữ lại build tools để debug nhanh (mặc dù vẫn có thể dùng target stage).

## 7. Trade-offs
| Pros | Cons |
|------|------|
| Kích thước image giảm tới 90%. | Dockerfile trở nên dài và phức tạp hơn. |
| Bảo mật cao (không có compiler/shell thừa). | Cần Docker phiên bản 17.05 trở lên. |
| Tách biệt rõ ràng môi trường build và runtime. | |

## 8. Alternatives
| Option | So sánh |
|--------|---------|
| Builder Pattern (Cũ) | Dùng 2 Dockerfile riêng biệt và một script Bash để copy file ra ngoài. Phức tạp và khó duy trì hơn nhiều so với Multi-stage. |

## 9. How
Ví dụ Multi-stage build cho một ứng dụng Spring Boot:

```dockerfile
# Stage 1: Build (đặt tên là builder)
FROM maven:3.9-eclipse-temurin-17 AS builder
WORKDIR /app
COPY pom.xml .
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Runtime (Image cuối cùng)
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
# Chỉ copy file JAR từ stage builder sang
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "app.jar"]
```

## 10. Production concerns
### Target Stages
Bạn có thể build đến một stage cụ thể bằng lệnh:
`docker build --target builder -t my-app:dev .`
Điều này hữu ích khi bạn muốn dùng stage đầu tiên cho việc chạy Unit Test trong CI.

### Debugging
Vì image cuối cùng thường rất thiếu thốn công cụ (không có curl, thậm chí không có bash), việc debug trực tiếp trên container Production sẽ khó khăn hơn. Luôn trang bị hệ thống logging tập trung tốt.

## 11. Common mistakes
- Mistake: Quên dùng `AS <name>` cho stage đầu tiên khiến việc `COPY --from` trở nên khó đọc (phải dùng index `0`, `1`).
  Fix: Luôn đặt tên stage gợi nhớ.

- Mistake: Vẫn dùng image runtime quá lớn (như bản full Ubuntu).
  Fix: Luôn dùng các bản `alpine` hoặc `slim` cho stage cuối.

## 12. Sample project
Tạo Dockerfile cho ứng dụng React:
1. Stage 1: Dùng Node image để chạy `npm run build`.
2. Stage 2: Dùng Nginx image để serve thư mục `build/` vừa tạo.
3. So sánh dung lượng image stage 1 (~1GB) và image cuối (~20MB).

## 13. Interview
### Core Q&A
1. Q: Lệnh `COPY --from=builder` có ý nghĩa gì?
   A: Nó hướng dẫn Docker hãy lấy tệp tin từ hệ thống tập tin của stage có tên là `builder` thay vì lấy từ máy host (build context).

2. Q: Image cuối cùng có chứa các layer của stage đầu tiên không?
   A: Không. Docker sẽ loại bỏ hoàn toàn các stage trung gian và chỉ giữ lại các layer được định nghĩa trong stage `FROM` cuối cùng.

### Scenario
"Làm thế nào để chạy Unit Test ngay trong quá trình build image nhưng không làm nặng image cuối cùng?"
-> Trả lời: Tôi sẽ đưa lệnh chạy test vào stage 1 (Build stage). Nếu test fail, quá trình build sẽ dừng lại và không tạo ra image lỗi. Nếu test pass, image cuối cùng (Stage 2) chỉ copy artifact đã được kiểm chứng vào, đảm bảo image nhẹ và an toàn.

## 14. References
- Official Docs: [Multi-stage builds](https://docs.docker.com/build/building/multi-stage/)
- Docker Blog: [Advanced Multi-stage Build Patterns](https://www.docker.com/blog/advanced-dockerfiles-faster-builds-less-bugs/)

## 15. Real-world Code
Hầu hết các "Official Images" của các ngôn ngữ biên dịch trên Docker Hub đều có ví dụ về Multi-stage build trong phần hướng dẫn.

## 16. Community
- Reddit: r/devops.
- Stack Overflow: Tag [docker-multi-stage-build].
